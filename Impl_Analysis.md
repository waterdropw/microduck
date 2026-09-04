# 实现分析（Impl Analysis）

> 对 microduck 关键数据链路的实现级分析，逐链路一篇。依据：源码模块注释与实测记录
> （`media-bringup.md` 等）。写于 2026-09-04。

---

## 之一：摄像头数据流程（`mediad`）

### 管线全景

```
videotestsrc（默认，无摄像头也有完整会话）      ← media-bringup：采集不能只是裸 v4l2src
或 v4l2src（摄像头，media.camera = true）
        │  原始 YUV 帧（不转换、不旋转）
        ▼
       tee ──────────────────────────────────────────────┐
        │                                               │
   queue → webrtcsink（编码归它管）                 queue（leaky，丢旧不背压）
        │  encoder-setup → mpph264enc                     → appsink（按需取帧）
        ▼                                                    │
   WebRTC 视频轨 ← 信令服务器(:8443，进程内)          三路消费者，各取各的：
   + "control" datachannel（robot API 管道）          ① get_frame：按需 JPEG（LLM agent 用）
   + "teleop" datachannel（不可靠传输）                ② duck-detect：2 Hz 找鸭子
                                                       ③ exposure：0.5 Hz 采样亮度
```

**tee 放在编码之前是整个设计的支点**：`architecture.md` §5.3 要的"按需单帧 JPEG"和
§2 要的"感知贴着传感器、推导特征而非搬运像素"，都需要未编码帧——一条原始支路两用，
谁也不必解码刚编码好的东西。

### 逐环节说明

#### ① 采集与旋转

- 默认源是 `videotestsrc`——不是偷懒占位，而是**无摄像头板子（大多数）的正确源**：
  信令、协商、datachannel、控制 API 全部可以在没有采集路径的情况下演练；
  摄像头是同一个编码器后面的另一个源元素（`media.camera = true` 切换）
- 摄像头在头上装歪了 1/4 圈，但**管线里不转**。旋转曾用 `videoflip` 放在 tee 前，
  破坏了 `mpph264enc` 到 SoC 2D 引擎（RGA）的零拷贝路径——MPP 落回**软件逐帧转换**，
  实测：97 °C、CPU 从 1.8 GHz 降频到 408 MHz、一个会话丢 1565 帧、30 fps 相机出 8 fps
- 旋转下放给消费者，两边都免费：浏览器用 GPU 上的 CSS transform；检测器折进它
  本来就要做的 letterbox 重采样。`--flip-in-pipeline` 仍存在，但代价已写在注释里，
  是一个知情选择而非默认

#### ② 软件自动曝光（`exposure.rs`）

- 存在的原因是实测：Rockchip 的 rkaiq 3A 引擎在流启动时设一次曝光**然后就停**——
  手写一个暗得多的值（`exposure=300 analogue_gain=256`），25 秒无人纠正。
  一次收敛不是自动曝光；鸭子从窗边走进走廊，画面定格在窗边
- 引擎错过 stream-start 事件时连那一次都没有（"3A 不工作了，重启有时能修"的形状）——
  本模块让曝光完全不再依赖两个 unit 的启动顺序
- 实现：0.5 Hz 从 tee 原始支路采样亮度（UYVY 的 luma 是每隔一个字节，均值只是一次
  子采样遍历，**零解码、零二次开摄像头**——原型第一版并行开 `v4l2-ctl` 采样自路径，
  在驱动层和主管线相争、间歇性杀掉管线），阻尼乘法步进写传感器
- 每 10 秒静默期也重写当前值：`v4l2src` 不是唯一写传感器的人，别人改过要知道

#### ③ tee 分流

- 两条支路各有独立 `queue`：没有队列的 tee 在单线程上跑分支，慢消费者会卡住别人
  （感知消费者暂停视频轨就是那个形状）
- 原始支路 **leaky**：阻塞的读者丢旧帧、不背压——§2 要的语义是"最新的快照，
  绝不是一队过期的"；卡住的读者付出帧数，编码器永远不付出

#### ④ appsink 按需取帧

- 每个 buffer 都到达 appsink，**几乎全部到那儿就被丢弃不读**
- 请求由**下一次采集**应答，不是缓存上一帧——"最新"的定义
- 三路消费者（get_frame / 检测 / 曝光）共享这一条支路，互不知晓

#### ⑤ duck-detect（`detect.rs` + `duck-detect` crate）

- **独立线程而非 tokio 任务**：每帧 60 ms 阻塞推理；这里的 tokio runtime 伺服 WebRTC
  信令，检测占了 worker 会话建立就无端卡顿
- **2 Hz 节流是热学数字**：放开跑能把 Zero 3 推到 95 °C 降频——"为了看清楚而走不好路
  的机器人"；2 Hz 占约 1/10 核
- **模型按序加载，NPU → CPU 回退**：`.rknn`（Rockchip NPU，dlopen `librknnrt.so`）
  失败 → `.onnx`（CPU，`ort`）——release 同时打包两种模型正是为此；回退是
  warn 进 journal 的决定（"60 ms 与 60 ms 别人家 CPU 的区别"），全失败才拒绝并提示
  `robot-setup-npu`
- **预处理契约**（跨仓库只有注释强制）：letterbox 不拉伸、灰 114 填充（ultralytics
  的习惯值，标定和训练看到的都是它）、RGB 非 BGR、90° 旋转折进重采样
- 输出 `Sighting`（原始帧坐标系的边界框 + 每帧耗时），`broadcast` 通道发给各 peer——
  每 peer 一份订阅，慢消费者在滞后时跳过、拖不住检测线程
- 计数器（looks/seen）进健康报告；每 10 秒一条 INFO——"检测器活着吗"不该需要开浏览器
  （房间里没鸭子 = 线程死了 / tee 静了 / 工作正常，三种状态从事外看无法区分）

#### ⑥ 编码与推流

- **`webrtcsink` 拿原始视频、自己管编码器**。曾试过 `mpph264enc ! h264parse ! webrtcsink`
  预编码：能跑，但悄悄丢了两样——拥塞控制够不到编码器、无法随链路调码率；
  peer 的 PLI（关键帧请求）出不来，丢过一个关键帧的观看者坏到下一个 GOP
- 代价：webrtcsink 不认识 `mpph264enc`，需要一个软件 `videoconvert ! videoscale` 臂——
  **上游插件带着他们打的补丁**（`microduck-gst-plugins` 的 `patches/`），两者必须成对
- 信令服务器跑在进程内（`run-signalling-server`），不用单独构建/发布一个 binary

#### ⑦ 会话与路由

- 每个 peer 两条 datachannel：`control`（可靠有序，robot API 管道）、
  `teleop`（`maxRetransmits: 0`——重传一个 80 ms 前的摇杆指令比丢掉更糟）
- `route.rs` 的可达面**由测试钉死**：`only_these_mutating_calls_are_reachable_over_webrtc`
  逐个点名 WebRTC 上允许的变更调用（跑技能、换策略、Hub 浏览/安装、账号登录）；
  `the_pin_and_the_factory_reset_are_never_available`——PIN 和恢复出厂永远不可达
- `upstream.rs` 持**五个** socket（robotd/configd/updaterd/padd/tofd），比 `btd` 多两个：
  pad 原始输入和 ToF 深度流，BLE 容量装不下、`btd` 刻意不持
- 每个操作超时有界、无例外——任何 peer 都可能是死的；`robotd` 最可能缺席，因为
  它是更新要重启的那个
- `:8080` 控制台：单页、零构建、daemon 自己伺服；host 由浏览器填（`location.hostname`），
  端口和 **API 版本**启动时注入——页面与二进制永不漂移，版本对比横幅才可信

### 配置面（`/etc/robot/robotd.toml` 的 `[media]` 段，`robotctl configure` 编辑）

| 键 | 含义 |
|---|---|
| `camera` | true=真摄像头，false=testsrc 测试图（无摄像头板子的正确默认） |
| `quality` | `1080p30`/`720p30`/`720p15`/`360p30`——全部 16:9，"更小"永不意味着"裁剪"；720p30 是所有实测的基准档 |
| `bitrate` | 起始码率；`congestion_control = "disabled"` 时就是终值 |
| `congestion_control` | `gcc`（默认，单 peer 时 7.6% 核——**既是网络设置也是 CPU 设置**）/ `homegrown`（更便宜更钝）/ `disabled`（删掉那个线程，bitrate 变成定值） |

### 功能清单

| 功能 | 入口 | 说明 |
|---|---|---|
| 实时视频通话 | WebRTC（信令 :8443） | 码率自适应，质量档全部实测过 |
| 远程控制台 | `http://<robot>:8080` | 零构建单页；API 版本注入杜绝页/固件漂移 |
| 按需 JPEG 帧 | `get_frame`（WebSocket/datachannel） | **LLM/服务器 agent 的路**：一两秒一帧 + 状态 blob，几十行代码驱动 |
| 找别的鸭子 | duck-detect，2 Hz | 边界框 notification 推给 peer；NPU→CPU 回退 |
| 软件自动曝光 | mediad 内部 0.5 Hz 环 | 替代只收敛一次的 rkaiq AE；不依赖 unit 启动顺序 |
| 手柄原始输入 / ToF 深度流 | teleop + tof.stream 经 mediad 转发 | `btd` 因 BLE 容量刻意不做 |
| 技能/策略/Hub 管理 | control datachannel | 可达面由测试逐个钉死；PIN/恢复出厂永不可达 |

### 对 Pi 验证的意义（接 `PI4_BENCH_RUNBOOK.md` G1/G6）

摄像头链路在 Pi 上唯一真正缺的是 `mpph264enc` 这一个编码器元素（Rockchip MPP 硬件
专属）。把编码器换成 `x264enc`（软编）或 `v4l2h264enc`（Pi 4 的 V4L2 M2M 硬编）后：
**get_frame、duck-detect（CPU ONNX 回退路径）、自动曝光、控制台**全部可在 Pi 上验证——
即摄像头链路的功能面在 Pi 上是完整可验的，缺的只是那块硬编的带宽/功耗特性。

---

## 之二：策略训练—发布—部署链路（`microduck_rl` → HF Hub → `robotd`）

> 依据：`docs/policy-manifest.md`（契约全文）、`docs/design/policy-channel-design.md`
> （通道设计）、`pollen-robotics/microduck_rl` 仓库（训练侧）。

### 全链路总览

```
① 训练 (microduck_rl, GPU)          ② 导出           ③ 发布 (HF Hub)         ④ 部署 (本仓库)
──────────────────────────          ──────           ──────────────          ─────────────
mjlab (MuJoCo Warp) 4096 并行环境    export.py        官方集 (9 个 onnx)       seed-policies.sh
PPO (rsl_rl)                        归一化器烘焙      pollen-robotics/         ↓
BAM 执行器建模 + 域随机化 + 背隙孪生   进 ONNX 图       microduck-policies      robotctl policy
~1-2 小时/步态                       [1,61]→[1,14]    社区单策略仓库           load/add/update
                                                     (manifest schema 2)     热重载,不重启
```

### ① 训练侧：sim2real 的三个关键决定

**执行器保真度是 sim2real 的主要矛盾**（官方原话："在这个尺度——微型舵机驱动 800 g
双足——执行器保真度就是 sim2real 差距的大头"）：

- **BAM M6 执行器模型**：为 XL330 建模电压控制律、反电动势、Coulomb/Stribeck/
  负载相关摩擦——而非 MuJoCo 默认力矩模型
- **域随机化**（`FrictionDRBamActuator`）：电池电压、负载下的电压跌落、指令延迟、
  摩擦幅值，逐环境随机，`ENABLE_*` 开关可关
- **背隙孪生**：每个任务都训一个"±1° 齿隙"的孪生版本——14 个关节各串一个无驱动的
  `passive_<joint>_backlash` 铰链，**编码器反馈穿过背隙读数**（与真硬件的齿轮间隙一致）

```bash
# 走路策略，4096 并行环境，1–2 小时
uv run train Mjlab-Velocity-Flat-MicroDuck --env.scene.num-envs 4096
# 无本地 GPU：直接送 Hugging Face Jobs 云训
uv run train Mjlab-Velocity-Flat-MicroDuck --env.scene.num-envs 4096 --hf-jobs
```

**观测契约在训练侧就锁死**：61 维（48 本体感知 + 13 指令），与运行时逐位一致——
这是"策略热切换"可行的根基。机器人模型是 MJCF，与本仓库 `kinematics` 同一几何真源。

### ② 导出：唯一合法路径

```bash
uv run scripts/export.py Mjlab-PoliteBow-Flat-MicroDuck --wandb-run-path <...>
```

- **观测归一化器烘焙进 ONNX 图**——运行时 `duck-control` 不做（也不知道）任何归一化，
  所以必须走这条导出路径，手工转换的 checkpoint 不认
- 图形状锁死 `[1,61] → [1,14]`，导出即验证

### ③ 发布：一个 manifest，两种形状

```bash
uv run publish --onnx output.onnx --repo <user>/microduck-<name> --kind episodic --duration-s 4.0
```

发布器上传前做三件事：验证图维度、**用合理输入跑一遍拒绝 NaN/常数输出**、写
schema 2 的 `manifest.json`（默认私有仓库）。manifest 两个轴决定策略在机器人上的命运：

- **`kind`（谁收尾）**：
  - `episodic`——跑完自己回安全姿态 → 可当通用技能（`policy add`）
  - `perpetual`——跑到被叫停 → 槽位步态（`policy load`），或带 `unwind_s` 当技能（`--hold`）
  - `scripted`——可中途改指令的 episodic，daemon 自己驱动其时序（sit↔stand）
- **`command.encoding`（喂什么）**：
  - `constant`——定速 twist（所有踢球、roulade、社区一次性技能）
  - `phase`——`[cos 2πφ, sin 2πφ, 0]` 相位编码（捡东西的俯身周期、roller 蹲伏）
  - `posture_flag`——一个槽位携带 `sit`/`stand`（sit↔stand）

只有 constant-command 的 episodic 策略能当通用技能；`phase`/`posture_flag` 被
`policy add` 拒绝，只属于 daemon 驱动的槽位——守护在 encoding 上，不在名字上。

**官方集**（`pollen-robotics/microduck-policies`，9 个文件）：`alpha_walking`、
`alpha_stand`、`alpha_sitstand`、`alpha_ground_pick`、`ball_kick_left/right`、`roller`、
`roller_crouch`、`roulade`。**名字是角色不是训练 run**——"换哪个 run 当走步策略"是改
一个 pin，不是改每台机器的配置。

### ④ 部署侧：通道存在的理由和机制

通道存在的理由（policy-channel-design.md §1，原先 9 个 .onnx 塞在 daemon artifact 里
的三个后果）：**步态重训要发 daemon 版；daemon 修 bug 要重下 6 MB 没变过的权重；
笔记本上训的策略只能通过 CI 或整体 sideload 上鸭子。**

| 命令 | 作用 |
|---|---|
| `robotctl policy load walk <repo>` | 装**槽位**（walk/stand/ground_pick/sitstand，daemon 自驱）——一次配置编辑 + **热重载，不重启** |
| `robotctl policy add polite-bow <repo>` | 加**技能**——技能只是 `robotd.toml` 里 4 个数字（时长/动作尺度/链式/unwind），曾是七处代码的改动 |
| `robotctl policy update` / `reset` | 升级官方集 / 一条命令回到出厂集 |

**四道拒绝门**，坏策略到不了电机：

1. `model_api` 新于 daemon → 拒绝
2. `obs_len ≠ 61` / `action_len ≠ 14` / `robot.model ≠ microduck` → **下载 800 KB
   之前**拒绝
3. 加载时（不是推理时）再验图形状——错误的宽度必须在机器人还站着时失败
4. 官方集只收 61 维家族——原型期的 51 维遗留策略被点名两个宽度拒绝（这个检查
   还顺手抓住过截断下载）

### 两个设计呼应

1. **"谁收尾"决定部署形态**：episodic 技能窗口到期直接交还步态（机器人自己是站着的）；
   perpetual 技能若直接到期会把一个单脚平衡的机器人扔给步态——所以 daemon 提供
   `unwind`（回程驱动的指令）补上策略没有的收尾。
2. **社区策略的"试了能退"**：`policy load`/`add` 全是配置编辑，`reset` 一条命令回到
   官方集——"试一个不是我训的策略，看看行不行，还能退回去"就是这个通道的第二个需求。

### 对 Pi 验证的意义

策略链路的部署侧在 Pi 上**完整可验**：`--fake` 模式下策略推理是真跑的（真 ONNX 会话、
61 维观测逐位构造），只差最后的电机执行。可验：`policy load` 的槽位热重载、`policy add`
的技能配置、manifest 四道拒绝门（用一个故意改错的 manifest 试）、`policy update` 从
HF 拉官方集。需要手动装 ONNX Runtime arm64（`workspace.metadata.onnxruntime` 钉的
1.28.0）。
