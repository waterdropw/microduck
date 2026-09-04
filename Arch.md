# Microduck 模块功能详解

> 全部 19 个 crate（+ 支撑目录）的功能说明，按仓库自己的三层分组：**守护进程 → 库 → 工具**。
> 依据：各 crate 源码头注释、`CONTRIBUTING.md`、`docs/design/architecture.md`（2026-09-04 整理）。

---

## 一、守护进程（7 个，每个一个 crate、一个进程、一个 socket）

全部通过 **JSON-RPC 2.0 / NDJSON / unix socket** 通信（`/run/<service>.sock`），互相之间是客户端关系，无 broker。

### `robotd` — 控制守护进程（核心）
- **50 Hz 控制循环**：读 15 个 Dynamixel 舵机 + IMU（共用一条 UART 总线），跑 ONNX 强化学习策略（走路/打滚/起立/捡东西），写回舵机
- 唯一能碰电机的进程——客户端只发**意图**（intents："走这么快"、"坐下"、"看那里"），内部安全层（`duck-control::safety`）裁决可执行性：跌倒检测、关节/温度限位、安全姿态，任何客户端都无法绕过
- **健康定义是精心设计的**：`robot.health` 不是"循环跑过一次"，而是"循环在赶截止时间"——60% 目标频率的循环活着、能应答、但已判定为坏，这正是更新器自动回滚所依赖的信号
- 附带功能：叫声（quack）、合唱（chorale，多只鸭子自动合奏）、特雷门琴（theremin，头部深度传感器当音高控制器）
- IPC 侧只读控制循环通过原子量发布的快照，**绝不回调进循环**——卡死的循环会把自己报成不健康，而不是拖死应答

### `updater` — 更新引擎 + `updaterd`
- 签名验证（minisign）、解包（tar.zst）、**原子交换**（移动 `/opt/robot/daemon/current` 符号链接）、重启单元、**健康门**（问 `robot.health`）、失败自动回滚
- 启动顺序有意设计：`Engine::recover_on_start` 在开始提供 socket **之前**运行——开机进了坏 release 的机器人，在任何客户端能发指令之前就已开始回退
- `updaterd` 自己排除在重启集合之外（它不能在更新中重启自己，这是 `architecture.md` §8.3 "运行版本 ≠ 安装版本" 的根因）
- **所有逻辑在库里**，daemon 只是很薄的一层——`updater/tests/apply.rs` 用故障注入驱动真实引擎（坏签名、不健康的 release、换链后断电），是这个系统"到底保证什么"的诚实答案

### `configd` — 配置守护进程
- wifi（驱动 NetworkManager over D-Bus，自己**永不存储**凭据）、机器人名字/身份、配对 PIN、重启、游戏配对手柄
- 关键设计约束：**必须在 `robotd` 死掉时可用**——机器人坏了的时候，人们最需要的就是重新配网/回滚，所以配置不能住在 `robotd` 里
- 状态存储：`/var/lib/robot/config/config.json`，`flock` + 写临时文件 + `rename(2)` 原子替换
- 手柄配对在这里而不是 `padd`：配对需要 root 和 BlueZ，而 `padd` 刻意是无特权客户端

### `btd` — 蓝牙前门
- BLE GATT 服务器，把 **API 的一个子集**（配网、状态、更新触发/进度）转发到对应 socket
- **什么都不拥有**——纯传输适配器，这样 app 挂了、换 SDK、加传输都不影响机器人行为
- 不转发 `padd`/`tofd` 的流：BLE 带宽装不下
- v4 起要求 `system.authenticate`，BLE 客户端先过配对 PIN 才能调用其他方法

### `padd` — 手柄 → 意图
- 读手柄（gilrs），把摇杆/按键翻译成意图，**作为一个普通客户端**发到 `robotd` 的 socket——无特权、无特殊访问
- 它独立成进程的额外价值：意图 API 也是 app/SDK/远程客户端的路径，`padd` 让这条路径**每天被人使用**，不会悄悄烂掉
- 一个 socket hop 的代价是几十微秒，对比 20 ms 的 tick，可忽略
- 按键映射在 `robotd.toml` 的 `[pad]` 段——学会新技能的机器人不用发版就能绑到按键上

### `mediad` — 摄像头 / 音频 / WebRTC / 远程网关
- GStreamer 管线（硬件 H.264 编码走 Rockchip MPP 的 `mpph264enc`），WebRTC 流媒体 + 信令服务器（`:8443`），控制台（`:8080`）
- **远程前门**：每个 WebRTC peer 得到一个 `control` datachannel，直通机器人 API（`mediad` 做代理）；`teleop` datachannel 用不可靠传输——重传一个 80ms 前的摇杆指令比丢掉更糟
- 服务器端 LLM/agent 的推荐路径是 WebSocket 而非 WebRTC：`get_frame` 按需取 JPEG + 状态 blob，几十行代码就能驱动机器人
- **刻意不鉴权**（信令端口可达即可驱动）：设计决策而非疏忽——配对 PIN 目前是共享的 `000000`，加门禁只添步骤不增安全
- **不在恢复路径上**：`mediad` 起不来，机器人照样走、照样收更新、照样蓝牙可达——所以允许它依赖 release 资产里的插件和设备节点组权限

### `tof`（`tofd`）— 头部深度传感器
- 8×8 VL53L5/8CX ToF 传感器（I²C），**只发布帧、不读任何东西**（`tof.stream` 订阅，默认 15 Hz）
- 独立成进程的具体理由：传感器上电要经 I²C 传 ~90KB 固件、耗时数秒；总线与音频编解码器共享；大多数鸭子根本没装这个头模组——这些都不该进 50 Hz 循环的进程里
- `--fake` 模式无传感器发假帧——离开板子后它唯一的运行方式

---

## 二、库（8 个，无 socket、无 systemd、没人启动它们）

### `duck-ipc-proto` — 线路协议契约（整个系统的枢纽）
- 所有 `update.*` / `robot.*` / `net.*` / `system.*` / `pad.*` / `tof.*` 方法的 serde 类型定义，方法与参数通过 `Call` 枚举**类型级绑定**——不可能把 A 方法的参数发给 B 方法
- 协议版本在 `hello` 里交换；版本号 bump 规则有成文约定（"一个 bump 在两个方向上都不承诺任何事"）
- **依赖被限定在 serde/serde_json/semver**——恢复路径上的所有服务都要说这些类型，任何修改都不得引入 http/tar/crypto/async 运行时依赖

### `duck-control` — 控制内核
- `model.rs`（机器人模型）· `bus.rs`（Dynamixel 总线）· `imu.rs` · `obs.rs`（策略观测向量）· `policy.rs`（ONNX 推理，`ort` dlopen 运行时）· `safety.rs` · `fall.rs`（跌倒检测）· `io.rs`
- 刻意不是 daemon：**没有 tokio、没有 socket**，边界由编译器强制——"进程关注点不得渗入驱动电机的代码"

### `kinematics` — MJCF 前向运动学
- 解析训练用的 MJCF 模型（与 `microduck_rl` 同一几何真源），加载时**编译一次**：名字→索引，每个 site 一条展平的 root→site 链——查询是链上 fold，无哈希无分配，为 50 Hz 循环里每 tick 算两只脚设计
- 头部链和手部链（`tofd` 的深度帧要和 `robot.state` 的关节状态结合做重投影，用的就是这里的头部 FK）

### `odometry` — 腿式里程计
- 任意时刻一只脚掌的某个角点是着地点：把它锚定到世界系（平地上 Z=0），IMU 定躯干姿态，正运动学推出躯干世界位置；换步时锚点迁移，估计值不跳变
- 从 Rhoban 人形机器人 `model_service` 移植；脚链改用 `kinematics` 的 MJCF 模型——几何只有一份真源

### `sounds` — 叫声合成器
- 一个整数种子（SoC 序列号）确定性导出一套**声音人格**：音域、谐波倾斜、鼻音度、颤音、quack 度、节奏——每只鸭子叫声都独一无二，同一只永远一样
- 7 种声音 tag（`alarm`/`greet`/`inquire`/`peck`/`chirp`/`coo`/`wheee`），每个 tag 多个变体防止像复读机
- 从 Python/numpy 重写为 Rust：release 自带声音生成，板子上不需要 venv/numpy/ffmpeg

### `pet-detect` — 摸头检测
- 40-band log-mel 频谱（1 秒窗口 @16kHz）→ ~20KB CNN → `PettingEvent::Start/End`（带迟滞）
- 从 `microduck_pet_detect` 逐数移植——mel 布局是训练契约，`pet-features` 二进制存在的意义就是让训练和推理共享同一份特征代码

### `duck-detect` — 找到别的鸭子（NPU 优先，CPU 回退）
- 单类目标检测，320×320 输入：优先 INT8 `.rknn` 在 Rockchip NPU 上跑（dlopen RKNN 运行时），加载失败回退 `.onnx` 走 CPU（`ort`）——artifact 同时打包两种模型正是为此
- 三件事：letterbox（灰 114 填充，不拉伸）、运行时、候选框解码；外加 `duck-bench` 在真板上测性能
- 预处理必须与训练完全一致（RGB 非 BGR、90° 旋转）——跨仓库没人强制这一点，全靠注释和数字

### `robotd-params` — `robotd` 启动参数
- `robotd.toml` 的 schema、默认值、校验，取代原型期的 142 个 CLI 旗标
- **启动读一次、不监视**——文件任何改动都要重启 `robotd`，编辑器永远不用问"哪个键是活的"
- 独立成 crate 的原因：`robotctl configure` 交互式编辑这个文件，对着拷贝的 schema 编辑就是 schema 漂移的温床

---

## 三、工具（4 个）

### `robotctl` — 板上 CLI（随 release 发布）
- **薄客户端**：解析 argv → 发一条 JSON-RPC → 打印流式通知和结果 → 映射退出码；零业务逻辑
- 全部子命令：`net` `system` `robot` `quack` `chorale` `theremin` `configure` `pad` `update` `account` `policy` `monitor` `health` `version`
- `robotctl health` 合并硬件（`robotd`）+ 软件（`updaterd`）为一个答案且**不健康时退出非零**——可作脚本门禁；`robotctl version` 在 `updaterd` 挂掉时也能工作，并区分两种版本偏差（各自意味着不同的诊断）

### `duckctl` — 笔记本侧客户端（永不发布、永不交叉编译）
- 手机 app 的替身，经蓝牙（`btleplug`：macOS CoreBluetooth / Linux BlueZ / Windows WinRT）连真机器人——测试 `btd` 对真实射频的唯一途径
- 刻意从 `default-members` 排除：`cargo board --bins` 绝不给开发者的机器交叉编译一个蓝牙栈

### `xtask` — 发布侧工具链（永不发布）
- `package`（打包）/ `sign`（签名）/ `promote`（staging→stable 同字节重签）
- 关键不对称性：`xtask` 链接完整 `minisign`（**能签**），`updaterd` 只链 `minisign-verify`（**不能签**）——机器人上不存在签名的私钥路径
- 用 Rust 而非 shell：复用和 `updaterd` 验证端完全相同的 `minisign`/`tar`/`zstd`/`sha2`，行为漂移无处藏身

### `test-support` — 测试用的签名 release 夹具
- 生成密钥对 → 打 `.tar.zst` → 签名 → 写 manifest → 签 manifest；统一了原先四份各自漂移的拷贝
- 同样用真引擎的 crate 组合，保证夹具产不出"真代码会拒绝"的工件

---

## 四、非 crate 的支撑目录

| 目录 | 内容 |
|---|---|
| `deploy/` | 板子配置：`updater.toml`、`robotd.toml`、信任锚、journald drop-in |
| `hooks/` | preinstall / postinstall——更新时在板内跑的钩子；`install.sh` 对板子做的任何事都属于这里（updater-design.md §9.1） |
| `scripts/` | 供板/开发/CI 三类脚本：`provision-board.sh`、`dev-push.sh`、`cross-sysroot.sh`、`board-test.sh`（CI 里 60 项断言）、`robot-rescue`（恢复，装到 `/usr/local/sbin`）等 |
| `docs/` | `robot/`（用）· `design/`（原理，每个模块一篇 design doc）· `project/`（路线图、事故记录）· `ideas/`（未设计） |

---

## 五、硬件清单：型号与模块对应

整机约 25 cm / 800 g。计算核心是一块 Radxa Zero 3W，外接一块自定义 HAT（音频编解码器和 ToF 传感器都挂在 HAT 引出的 `i2c3` 上，经排针 3/5 脚）；`imu_to_dxl` 板与 15 个舵机共用 Dynamixel 串行总线（见下）。

### 总表：硬件 ↔ 型号 ↔ 接口 ↔ 驱动它的模块

| 硬件 | 型号 | 接口 / 总线 | 驱动模块 |
|---|---|---|---|
| 主控板 | **Radxa Zero 3W**（Rockchip RK3566，4× Cortex-A55，aarch64；0.8 TOPS NPU；rkisp ISP） | — | 所有 daemon 的宿主；NPU 归 `duck-detect`，ISP 归 `mediad` |
| 舵机 ×15 | **ROBOTIS Dynamixel XL330**（协议 v2） | `/dev/ttyS2`（UART2）@ 1 Mbps，`TIOCEXCL` 独占 | `duck-control::bus`（rustypot `Xl330Controller`）← `robotd` 50 Hz 循环 |
| IMU | **ST LSM6DSV16X**，装在 `imu_to_dxl` v2 板上 | 挂在 Dynamixel 总线上，`id 200`，与舵机同一次 `sync_read` 读出 | `duck-control::imu` ← `robotd` |
| 音频编解码器 | **TI TLV320AIC3104**（扬声器 + 麦克风） | I²S 音频流 + `i2c3` 控制（排针 3/5 脚），DKMS 树外驱动，overlay 源码在 `deploy/audio/` | `robotd::sound`（放声）、`pet-detect`（mic 采集） |
| ToF 深度传感器 | **ST VL53L8CX**（旧代 VL53L5CX，同接口按 ID 自动识别互换） | `i2c3`，与编解码器**共享总线**；地址 0x29 / 0x52；稳定设备名 `/dev/i2c-pihat` | `tof`（`tofd`） |
| 摄像头 | **Sony IMX219**（MIPI CSI；Radxa IQ 包另支持 OV5647） | MIPI CSI → Rockchip vendor rkisp + rkaiq 3A | `mediad`（推流）、`duck-detect`（NPU 检测别的鸭子） |
| 无线（WiFi + 蓝牙） | **AIC8800**（板载 SDIO 二合一） | NetworkManager（WiFi）/ BlueZ（BT） | `configd`（配网、手柄配对）、`btd`（BLE 前门）、`padd`（手柄输入） |
| 手柄 | 蓝牙游戏手柄 | BlueZ → evdev → gilrs | `padd` |
| 电池 | **NP-F550** 电池组（6.6 V 空载读数 = 0%，8.2 V = 100%） | 电压经总线读出 | `duck-control::model::battery_percent` ← `robotd`（`robot.health`、低电自动关机） |
| 板载温度 | sysfs 温区（`soc-thermal` 等，取各温区最大值） | `/sys/class/thermal` | `robotd::soc`（刻意走 sysfs 而非 `RobotIo`——电机总线挂了也要能读温度） |

### 值得知道的硬件细节

- **一条 UART，十六个设备**：15 个 XL330 加 `imu_to_dxl` 板共享 `/dev/ttyS2`，没有第二条总线。IMU 板（id 200）排在 id 向量最前，让它在舵机突发前应答；芯片的 SFLP 块直接输出游戏旋转四元数并自估陀螺零偏，主机侧**无滤波、无融合**——一个板、一条代码路径。
- **`setup-board.sh` 必须关掉 `serial-getty@ttyS2`**：Armbian 默认在 UART2 上跑登录控制台，一个 `agetty` 占着口，所有舵机对谁都不可见。`fuser -v /dev/ttyS2` 是排查这个的第一命令。
- **`i2c3` 是共享总线**：ToF 传感器和音频编解码器同在 `i2c3`（排针 3/5 脚）。`setup-board.sh` 的音频段落负责把总线本身带起来，ToF 段只补 `/dev/i2c-pihat` 这个稳定名。
- **ToF 上电要传 ~90 KB 固件**（每次启动都传，走 I²C 需要数秒）——这是它独立成 `tofd` 进程的直接原因之一；两代传感器（L5CX/L8CX）在板上是可互换的，daemon 按 ID 读数选驱动。
- **内核选择受硬件约束**：音频编解码器的 I²S 时钟树只存在于 Armbian **vendor（BSP 6.1）** 内核，rkaiq 3A 也只在 vendor rkisp 上有——所以固件被锁定在 vendor 内核，不能换 mainline。
- **XL330 出厂 `return_delay_time` = 250**（500 µs 往返/设备），16 个设备就是 8 ms——模型初始化时必须把它调小，这是控制循环能跑 50 Hz 的前提之一。
- **AIC8800 无线芯片的蓝牙问题**（两个独立故障，两个旗标分别绕过；这块芯片**不是最终量产的射频方案**，换掉后 workaround 和旗标一并删除）：
  1. **`btd` 广播期间手柄无法新建 bond**。`--pause-btd-on-pair` 让 `robotctl pad pair` 在配对窗口期间暂停 `btd`；实测还证明它才是早年那些"驱动背锅"的真相——以前归咎于 aic8800 驱动的失败案例全都发生在 `btd` 广播时，那才是未被控制的变量。
  2. **约半数 Zero 3W 在 BlueZ 默认 `Privacy = off` 下根本无法配对**，只有 `Privacy = device` 能用（`--weird-ble` 设定）。但 `device` 在不需要它的板子上会**破坏重连**——干净的 bond 也会以 `PIN or Key Missing` 摆动，比配不上更糟，因为它看起来成功了。
  两者修复的不是同一个故障，不能合并处理：一块能用 `off` + 暂停配对的板子要新旗标；一块 `off` 下完全无法配对的板子仍要 `--weird-ble`。
- **`scripts/pad-link-test.sh` 测的是这条蓝牙链路的两种死法**：**掉线**（设备消失，日志里有，死人开关会触发）和**停顿**（连接在、报告停了几百毫秒——更危险，`padd` 会拿旧摇杆值继续发 50 Hz 意图，`robotd` 看到的都是"新鲜"指令，死人开关不触发，机器人带着过期指令走）。
- **电池电压经 Dynamixel 总线读出**（总线无应答 = 读数为 0，报未知而非显示 0%）；满/空映射（8.2/6.6 V）写在 `duck-control::model`，客户端画电量条不用知道电池型号。
- **IMX219 的 3A（白平衡/曝光）要装 rkaiq**：不装也能用，但画面发绿、噪声大、曝光固定——`scripts/setup-rkaiq.sh` 装 Rockchip 的 camera-engine-rkaiq 和 IMX219 调优文件。

---

## 六、当前的功能与可玩性（按使用门槛分层）

> 依据：README "It does things"、`robotctl --help`、官方策略集、`docs/ideas/autonomous_behavior.md`。

### 第一层：遥控与技能（开箱可用）

| 玩法 | 说明 |
|---|---|
| 手柄驾驶 | 蓝牙手柄 + RL 步态，摇杆即走；松手自站（`alpha_walking` 策略） |
| 一键技能 | 手柄 5 个单发按钮绑定技能：左/右踢球、捡东西（嘴尖触地叼起）、前滚翻、坐站切换；按键映射在 `robotd.toml`，改动不需要发版 |
| 摔倒自起 | VelStand/StandUp 策略：推倒后自行爬起 |
| 装轮滑行 | 脚底装被动轮，D-pad 上键加载 `roller` 策略——同一只鸭子切换运动模式；轮滑族另有滑坡、原地旋转、蹲滑等共 6 个任务的策略 |

### 第二层：个性化与"活物感"设计

- **独一无二的声音**：叫声由 SoC 序列号做种子合成——每只鸭子的音色都不同且终身不变；7 种叫声（问候/好奇/警报/骑行 wheee 等）各有变体
- **特雷门琴**（`robotctl theremin`）：头部 ToF 传感器作为音高控制器，手靠近喙音调升高，嘴部随音符开合，实时合成
- **合唱**（`robotctl chorale`）：同一房间的两只鸭子自动开始合奏，更多鸭子陆续加入；靠 BLE 广播以稳定 id 识别同伴（不受蓝牙地址轮换影响），节拍同步 ±20 ms 且无需时钟同步
- **摸头反应**：麦克风 + 20 KB CNN 识别"被摸头"，发出回应声
- **识别同类**：摄像头 + NPU 检测其他 Microduck，边界框实时推送至控制台

### 第三层：社区与自定义扩展

- **策略即插即用**：Hugging Face 上任何人发布的策略一条命令即可安装——`robotctl policy add polite-bow <user>/microduck-polite-bow`；社区已有第三方训练的技能（如单脚站立姿势类）
- **自训步态快速上机**：MJCF → PPO → ONNX → `publish` → `policy load walk`，从 GPU 到机器人行走约半小时，全程不需要 daemon 发版
- **可定制项**：每只鸭子的声音人格可重新生成、手柄按键可重绑、`robotctl monitor` 实时 3D 观察机器人状态

### 第四层：开发者玩法（当前验证路径所在）

- **无硬件运行软件栈**：交叉编译 → qemu / `robotd --fake` / `tofd --fake`；上游另有 `duck-body` + duck-sim 多鸭仿真栈（多只鸭子在同一公寓场景中交互，每只带摄像头与独立控制台）
- **完整的更新系统研究样本**：签名验证 → 健康门 → 自动回滚的闭环，且故障注入路径齐备（`--unhealthy`、`--inject-fault`）
- **外接 LLM 作为高层大脑**：`mediad` 的 WebSocket 路径即为此设计——`get_frame` 获取一帧图像 + 发送高层意图，几十行代码即可让 LLM 驾驶机器人；daemon 侧无需任何改动

### 尚不具备的能力（截至 main `bc41fb5`）

- 无自主行为：不遥控时不会自行活动（原型机的 16 状态自主机在待移植清单上，见 `docs/ideas/autonomous_behavior.md`）
- 无语音识别、无避障、无路径规划
- 定位：**遥控的 RL 双足机器人 + 传感器与表达丰富的平台**；自主性属于下一阶段

---

## 理解全局的一条线索

这个仓库的模块划分全在回答同一个问题——"**`robotd` 挂了之后，机器人还剩什么？**"

- `configd` / `updaterd` / `btd` 刻意**零依赖** `robotd`（恢复路径：机器人坏了正是最需要配网、回滚的时候）
- `padd` / `mediad` / `tofd` 可以依赖它（没相机没手柄的机器人仍是一台可更新的机器人）
- 所有库不碰进程概念，保证驱动电机的代码里永远渗不进 socket 或 systemd 的关注点
- 每条状态只有一个所有者（单一写入者），其他人只能读或订阅
