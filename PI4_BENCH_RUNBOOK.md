# Pi 4 软件栈全流程验证 runbook

> 目标：在一块 Raspberry Pi 4（Pi OS Lite 64 位，Debian arm64 用户态）上，用本机交叉编译的
> 产物验证 microduck 软件栈的完整流程——**bootstrap 安装 → 更新 → 健康门 → 自动回滚**，
> 全部走真实引擎（`updaterd` 的库，不是 mock）。不需要 HAT、舵机、摄像头。
>
> 依据：`install.sh` 的目标就是"任意 Debian 12/13 arm64 用户态"；`robotd --fake` 是官方的
> 无硬件模式；`updaterd install --from` 是离线 bootstrap 路径。写于 2026-09-04。

## 阶段 A：宿主机——打包签名两个 bench release

产物已编好（`target/aarch64-unknown-linux-gnu/release/`，qemu 冒烟通过）。

```bash
export PATH=~/.cargo/bin:~/.local/bin:$PATH
cd ~/repos/microduck

# 1. 生成 bench 密钥对（自己的信任锚，不用 team.dev）
cargo run -q -p xtask -- keygen   # 按 --help 确认输出路径参数
# 得到 bench.secret / bench.pub（名字以实际输出为准）

# 2. 暂存二进制——清单与 dev-push.sh / release.yml 完全一致
mkdir -p staged
for b in updaterd robotctl robotd configd btd padd mediad sounds \
         pet-detect pet-features tofd; do
  cp target/aarch64-unknown-linux-gnu/release/$b staged/
done

# 3. 打 v1（--include 清单照抄 scripts/dev-push.sh 第 363–399 行，此处省略）
cargo run -q -p xtask -- package \
  --version 0.10.0-bench.1 --channel daemon --bin-dir staged --out dist1 \
  --revision "$(git rev-parse --short HEAD)" --zstd-level 1 \
  --include "..."   # ← 照抄 dev-push.sh
cargo run -q -p xtask -- sign --dir dist1 --key bench.secret

# 4. 打 v2（同一份二进制也行，版本号即差异）
cargo run -q -p xtask -- package \
  --version 0.10.0-bench.2 --channel daemon --bin-dir staged --out dist2 \
  --revision "$(git rev-parse --short HEAD)" --zstd-level 1 --include "..."
cargo run -q -p xtask -- sign --dir dist2 --key bench.secret
```

`dist*/` 里应有：`<version>.manifest.json` + `.minisig`、`.tar.zst` + `.minisig`
——这正是 `updaterd install --from` 期望的布局（`updater::source::local`）。

## 阶段 B：Pi——刷机与环境

1. Raspberry Pi Imager → **Raspberry Pi OS Lite (64-bit)**（Bookworm/Trixie 均可），
   预配 ssh 和用户
2. 基础包：`sudo apt update && sudo apt install -y network-manager`
   （Bookworm 起默认就是 NM；configd 需要）
3. 布局 + 信任锚 + bench 配置：

```bash
sudo mkdir -p /etc/robot/trusted_keys /opt/robot/daemon /var/lib/robot/updater
sudo cp bench.pub /etc/robot/trusted_keys/
sudo tee /etc/robot/updater.toml <<'EOF'
trusted_keys_dir = "/etc/robot/trusted_keys"
state_dir        = "/var/lib/robot/updater"
robot_socket     = "/run/robotd.sock"
hw_rev           = 1

# bench 与客户机的两处关键差异（deploy/updater.toml 里都是 false）
allow_dev_keys        = true
allow_fault_injection = true

auto_apply = "off"    # bench 手动触发

[component.daemon]
install_dir   = "/opt/robot/daemon"
keep_previous = 1

[component.daemon.source]
# 永远不会被用到——所有 apply 都走 --from 本地目录；占位即可
type           = "github_releases"
repo           = "pollen-robotics/microduck"
tag_prefix     = "daemon-v"
manifest_asset = "manifest.json"

[component.daemon.on_apply]
action = "restart"
units  = ["robotd", "configd"]

[component.daemon.health]
probe   = "socket"
timeout = "30s"
EOF
```

## 阶段 C：Pi——bootstrap 安装 v1

```bash
# 宿主机：把自举二进制和 dist1 送过去
scp target/aarch64-unknown-linux-gnu/release/updaterd pi@<ip>:/tmp/
scp -r dist1 pi@<ip>:/tmp/

# Pi 上：
sudo /tmp/updaterd install --config /etc/robot/updater.toml --from /tmp/dist1
# 走的就是真引擎：验签 → 解包 → 原子交换 → postinstall(装 units/sysusers) → journal
# （install 模式强制 on_apply=none / health=none——首装时没有可重启的单元和可问的 robotd）
```

装完先给 `robotd` 配 fake drop-in（`--fake` 只能走命令行；drop-in 在 `.d/` 目录，
postinstall 重装 unit 文件不会覆盖它）：

```bash
sudo mkdir -p /etc/systemd/system/robotd.service.d
sudo tee /etc/systemd/system/robotd.service.d/bench-fake.conf <<'EOF'
[Service]
ExecStart=
ExecStart=/opt/robot/daemon/current/bin/robotd --socket /run/robotd.sock --fake --no-policy
EOF
```

起服务（先起核心四个；`mediad` 跳过——`mpph264enc` 是 Rockchip 专属）：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now updaterd robotd tofd configd padd
# tofd 没传感器会自己说明（架构文档 §1：没装传感器的板子照样跑它）
# padd 没手柄配对就空转，正常

robotctl version   # 应报 bench.1
robotctl health    # fake robotd → 硬件侧 healthy；退出码 0
```

## 阶段 D：Pi——更新与回滚（本验证的核心）

### D1. 正常更新（健康 → 保留）

```bash
scp -r dist2 pi@<ip>:/tmp/
ssh pi@<ip> 'sudo robotctl update apply --from /tmp/dist2'
# 引擎完整流程：preflight → 验签 → 解包 → 交换 current → postinstall
#              → systemctl restart robotd,configd → 健康门问 robotd(fake=healthy) → 保留
robotctl version      # 安装版本与运行版本一致
robotctl update log   # journal 里一条 apply 记录
```

### D2. 故障注入（不健康 → 自动回滚）

```bash
# 把 robotd 换成报不健康的（fake + unhealthy）
sudo tee /etc/systemd/system/robotd.service.d/bench-fake.conf <<'EOF'
[Service]
ExecStart=
ExecStart=/opt/robot/daemon/current/bin/robotd --socket /run/robotd.sock --fake --no-policy --unhealthy
EOF
sudo systemctl restart robotd
robotctl health      # 退出码非零——这就是健康门要看的

# 再推一个 bench.3（宿主机如法打包），apply：
sudo robotctl update apply --from /tmp/dist3
# 预期：交换成功 → 健康门失败 → 引擎自动换回 bench.2 → journal 记录回滚
robotctl version     # 回到 bench.2
ls /opt/robot/daemon/releases/   # keep_previous=1：旧版还在
```

### D3. 坏签名（篡改 → 拒绝）

```bash
# 宿主机打 bench.4，然后故意翻转 artifact 里的一个字节，再 apply：
# 预期：preflight/verify 阶段拒绝，current 不动，journal 记录失败原因
```

### D4.（可选）socket 授权层

```bash
# 非 root、且不在允许名单里的用户跑 robotctl update apply
# → 读类调用（status/log/list）通，变更类 PERMISSION_DENIED
# （0660 + 组 = 谁能说话；allow_users = 谁能改——两层分开验证）
```

## 阶段 E：外设接入（按难度分组，从零接线到真控制环）

| 组 | 外设 | 接入方式 | 归属模块 | 配置/代码改动 |
|---|---|---|---|---|
| G4 | 蓝牙手柄 | 板载 BT，零接线 | `padd` + `configd` | 纯配置 |
| G2 | 扬声器 + 麦克风 | USB 声卡即插 | `robotd::sound` / `pet-detect` | 纯配置（`audio.device`） |
| G3 | ToF 深度传感器 | I²C 四根线 | `tofd` | 纯配置（`--bus`） |
| G1 | 摄像头 | CSI 排线或 USB | `mediad` | ⚠️ **需改代码**（编码器） |
| G5 | 15 舵机 + IMU | USB-UART + 独立供电 | `robotd` 真总线 | 纯配置（`--port`），硬件最讲究 |
| G6 | NPU | — | `duck-detect` | ❌ Pi 上没有，不可验 |

### G4 蓝牙手柄（最简单）

1. Pi 板载蓝牙 + BlueZ，Pi OS 自带
2. `sudo robotctl pad pair`（`configd` 驱动 BlueZ 完成 bond）
3. 验证链：`robotctl pad list` → 按键 → `robotctl monitor` 看意图流 → `robotd --fake` 收到 walk 意图
4. AIC8800 的两个 workaround 在 Pi 上不适用也不需要——顺带验证"workaround 是板级的不是设计级的"

### G2 扬声器 + 麦克风（USB 声卡）

真机上的 TLV320AIC3104 对软件只是"一个 ALSA 声卡"；DKMS/overlay 那些适配工作是 Rockchip 板级的，Pi 完全绕开。

1. 硬件：任意 USB 声卡（playback + capture 双通道）
2. `sudo apt install -y alsa-utils`（放声和采集都是 `aplay`/`arecord` 子进程调用）
3. `robotctl configure` 设 `audio.device`（`aplay -l` 查名，`default` 或 `plughw:CARD=X`）
4. 验证：`robotctl quack`（Pi 上会用板序列号当种子）→ 摸麦克风触发 `pet-detect` → `robotctl theremin`（需 G3）
5. 坑：USB 麦克风增益与真机 codec 差异大，`pet-detect` 阈值可能要调

### G3 ToF 深度传感器（VL53L8CX 分线板）

```
Vin → 3.3V   GND → GND   SDA → GPIO2   SCL → GPIO3
```

1. `/boot/firmware/config.txt`：`dtparam=i2c_arm=on,i2c_arm_baudrate=400000`（ToF 上电传 90 KB 固件，100 kHz 太慢）
2. `tofd --bus /dev/i2c-1`（真机的 `/dev/i2c-pihat` 在 Pi 上不存在，`--bus` 就是为不同接线准备的）
3. 验证：`robotctl monitor` 订阅 `tof.stream`，手在传感器前移动看 `distance_mm` 变化
4. 坑：分线板默认地址要和 `--address 0x29/0x52` 对得上（`tofd` 两个都试）

### G1 摄像头（唯一需要动代码的）

`mediad` 管线：`videotestsrc/v4l2src → NV12 → videoflip → tee → webrtcsink → mpph264enc`，**编码器元素名固定在 `pipeline.rs` 里**（Rockchip MPP 零拷贝路径），Pi 上插件不存在，管线起不来。

1. 硬件：**第一次建议 USB UVC 摄像头**（`v4l2src device=/dev/video0` 直接工作）；Pi 官方 CSI 摄像头在 Bookworm 走 libcamera，`v4l2src` 看不到，要用 `libcamerasrc`
2. 代码改动（`mediad/src/pipeline.rs`）：编码器可选化——Pi 4 有 V4L2 M2M 硬编（`v4l2h264enc`，bcm2835-codec），或退 `x264enc` 软编。**旋转必须留在管线里**（`videoflip` 在 tee 之前），否则重复"97 °C 降频到 8 fps"的旧坑
3. `webrtcsink`/`webrtcsrc` 从 `microduck-gst-plugins` 源码构建（arm64 通用），`setup-gstreamer.sh` 是 Rockchip 环境写的，Pi 上手工装 GStreamer 1.24+ 再编
4. 配置：`/etc/robot/robotd.toml` 的 `[media]`：`camera = true`、`quality = "720p30"`（每个 rung 都是测过的）
5. 验证：浏览器开 `http://<pi>:8080` → WebRTC 会话出画面；顺带验证 `[media]` 段"改它只 restart mediad 不动 robotd"的配置边界
6. 默认 `camera = false` 时走 `videotestsrc` 测试图——没摄像头也有 WebRTC 控制通道

### G5 舵机 + IMU（真控制环，硬件最讲究）

1. **接口**：Dynamixel 是单线半双工 TTL，普通 USB-UART 不能直连——用 **ROBOTIS U2D2**（或 LN-101/OpenCR）
2. **供电**：XL330 是 5 V，15 个舵机峰值不小，**独立 5V 电源**（共地），绝不能从 Pi 取电
3. 接线：XL330 菊花链 + `imu_to_dxl` 板（id 200）同一总线；没有 IMU 板也能跑，但观测向量缺姿态输入
4. 配置：`robotd --port /dev/ttyUSB0`（`--help` 原话："为接线不同的板子准备"）；systemd drop-in 里把 `--fake` 换成 `--port`
5. 策略：手动装 ONNX Runtime arm64（`workspace.metadata.onnxruntime` 锁定的 1.28.0）+ 从 HF `pollen-robotics/microduck-policies` 拉策略
6. 验证阶梯：`--no-policy` 先跑通总线读写 → `robotctl robot init`（上力矩回 home）→ `robotctl robot stand` → 最后才挂策略走路
7. **安全**：这一步起机器人会真的动——架空、急断电在手边

### G6 duck-detect（NPU 优先，CPU ONNX 回退——Pi 可验）

`duck-detect` 的模型加载是**按序尝试**：`.rknn`（NPU，dlopen `librknnrt.so`）失败 → `.onnx`（CPU，`ort`）——artifact 同时打包两种模型正是为此。Pi 没有 NPU，但 CPU 路径完整可用（每帧推理会慢于 NPU 的 60 ms，检测线程自带 2 Hz 节流，不构成问题）。需要 ONNX Runtime arm64（与 G5 的策略运行时是同一个）。**修正**：本节早期版本说"没有 CPU 回退路径"，是错的。

### 其余

| 项 | 说明 |
|---|---|
| `robotctl monitor` 3D 视图 | `--fake` 下可试，姿态静止 |
| LLM/语音交互 | 本仓库无板上实现——设计上是服务器侧 agent 走 `mediad` 的 WebSocket（见 Arch.md"远程网关"） |

## 排错入口

- `journalctl -u updaterd -u robotd -f` —— 一切的第一现场
- `robotctl version` 在 updaterd 挂掉时也能工作，并区分两种版本偏差
- `/var/lib/robot/updater/update-log.jsonl` —— 断电也活的更新历史（§8.2）
- `updaterd --self-test` —— 只加载配置构造引擎就退出，验证二进制/库/配置兼容
