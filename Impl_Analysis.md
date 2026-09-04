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
  预编码：能跑，但悄悄丢了两样——拥塞控制无法访问编码器、无法随链路调码率；
  peer 的 PLI（关键帧请求）出不来，丢过一个关键帧的观看者坏到下一个 GOP
- 代价：webrtcsink 不认识 `mpph264enc`，需要一个软件 `videoconvert ! videoscale` 臂——
  **上游插件带着他们打的补丁**（`microduck-gst-plugins` 的 `patches/`），两者必须成对
- 信令服务器跑在进程内（`run-signalling-server`），不用单独构建/发布一个 binary

#### ⑦ 会话与路由

- 每个 peer 两条 datachannel：`control`（可靠有序，robot API 管道）、
  `teleop`（`maxRetransmits: 0`——重传一个 80 ms 前的摇杆指令比丢掉更糟）
- `route.rs` 的可达面**由测试固定**：`only_these_mutating_calls_are_reachable_over_webrtc`
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
| 技能/策略/Hub 管理 | control datachannel | 可达面由测试逐项固定；PIN/恢复出厂永不可达 |

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

**观测契约在训练侧即已锁定**：61 维（48 本体感知 + 13 指令），与运行时逐位一致——
这是"策略热切换"可行的根基。机器人模型是 MJCF，与本仓库 `kinematics` 同一几何真源。

### ② 导出：唯一合法路径

```bash
uv run scripts/export.py Mjlab-PoliteBow-Flat-MicroDuck --wandb-run-path <...>
```

- **观测归一化器烘焙进 ONNX 图**——运行时 `duck-control` 不做（也不知道）任何归一化，
  所以必须走这条导出路径，手工转换的 checkpoint 不认
- 图形状固定为 `[1,61] → [1,14]`，导出即验证

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
- **`command.encoding`（输入什么指令）**：
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
笔记本上训的策略只能通过 CI 或整体 sideload 部署到机器人。**

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
HF 拉官方集。需要手动装 ONNX Runtime arm64（`workspace.metadata.onnxruntime` 锁定的
1.28.0）。

---

## 之三：RL 训练代码库分析（`microduck_rl`）

> 依据：`microduck_rl` 仓库源码、README、AGENTS.md（奖励设计手册——本篇多处直译其
> "每条都用真实失败换来的规则"）。补充之二的部署侧视角，本篇是训练侧的实现细节。

### 技术栈与总体流程

```
Onshape CAD ──onshape-to-robot──▶ MJCF 模型变体 (walk/groundcontact/rollers/±backlash)
                                        │
                                        ▼
mjlab (MuJoCo Warp, GPU 并行) + rsl_rl PPO, 50 Hz, 4096 envs
  BAM M6 XL330 执行器 + 域随机化 + 背隙孪生
                                        │  wandb (project: mjlab_microduck)
                                        ▼
scripts/export.py ──▶ ONNX (归一化器烘焙进图, [1,61]→[1,14])
                                        │
                        ┌── publish ──▶ HF Hub (schema-2 manifest)
                        └── scripts/infer_policy.py ──▶ CPU MuJoCo 部署预演
```

- 需 CUDA GPU（MuJoCo Warp）；无卡走 `--hf-jobs`（Hugging Face Jobs 云训，`train_hook.py`
  拦截该旗标转投 `hf_jobs.py`）
- ARM 机（DGX Spark/Jetson）有专门的轮子陷阱：PyPI 的 aarch64 torch 是 CPU-only，
  `[tool.uv.sources]` 只对 aarch64 把 torch 路由到 cu129 索引，且 pin 必须 `==`
  （`>=` 曾把 torch 悄悄升级到 2.13.0）——`tests/test_aarch64_cuda_torch.py` 锁定这两个约束

### 任务族（13 个，注册于 `tasks/__init__.py`）

| 任务 | 说明 |
|---|---|
| `Velocity-{Flat,Rough}` | **主任务**：速度指令 + 头部姿态指令的行走；同时是所有其他环境的**共享基座**（robot/DR/obs/commands 从它派生） |
| `VelStand` | 行走 + 摔倒自恢复合一 |
| `StandUp` | 从俯卧/仰卧/坐姿站起 |
| `SitStand` | 指令驱动的坐↔站（posture flag） |
| `GroundPick` | 蹲下嘴尖触地再站起（phase 指令） |
| `BallKick` | 踢 70mm/15g 球——**actor 对球是盲的**（观测里没有球） |
| `Roulade` | 前滚翻 |
| `Rollers/Swizzle/RollerCrouch/RollerSlope/RollerStandUp/Spin` | 轮滑族（脚底被动轮） |

每个主任务都有 **Backlash 孪生**（任务名插 `-Backlash-`，`backlash.py` 的
`make_backlash_variant()` 包装任意 env cfg），且孪生必须镜像宿主任务的机器人模型
（walk/groundcontact/rollers 对应同款），保证背隙 A/B 对比不被模型差异污染。

### 机器人模型层

- MJCF 从 **Onshape** 经 `onshape-to-robot` 导出，一个 `config_mjcf_*.json` 一个变体：
  - `robot_walk.xml`——剥掉躯干/头部碰撞体（摔倒廉价，训练快）
  - `robot_groundcontact.xml`——精选碰撞集，机身能够物理接触地面（VelStand/StandUp/
    SitStand/GroundPick/BallKick/Roulade 用）
  - `robot_groundcontact_rollers.xml`——加被动轮
  - `robot_*_backlash.xml`——由 `add_backlash.py` **生成**而非手写
- 14 舵机布局：0–4 左腿（hip_yaw/roll/pitch, knee, ankle）、5–8 颈头、9–13 右腿；
  无驱动关节一律 `passive_*` 前缀（轮子、背隙铰链），所有选择器用 `^(?!passive_).*`
- ⚠️ roller/backlash 模型的被动关节**交错**在舵机之间——`mdp.py` 禁止硬编码关节索引，
  必须用 `_servo_joint_ids` 助手（普通模型上是恒等，交错模型上算对）

### 执行器建模：sim2real 的核心（`actuator/friction_dr_bam.py`）

- **BAM M6 XL330 模型**：电压控制律（不是理想 PD）、反电动势、
  Coulomb/Stribeck/负载相关摩擦
- **`FrictionDRBamActuator` 子类存在的原因**：BAM 把 MuJoCo 的 `dof_frictionloss`
  归零（摩擦由 BAM 自己在 `compute()` 里算），所以 mjlab 自带的
  `dr.dof_frictionloss` 域随机化在 BAM 下是**静默 no-op**——子类加了逐环境的
  `friction_scale`，在 `_compute_friction_budget` 内乘进速度无关摩擦预算
  （Coulomb+Stribeck+负载项）；粘性项留标称
- **背隙建模**：每个舵机串一个无驱动 `passive_<joint>_backlash` 铰链（±1°，共 2°），
  真实编码器在游隙的**输出侧**，所以固件 PD 仿真（`BacklashEncoderBamActuator`）和
  `joint_pos`/`joint_vel` **观测**都读 `qpos[servo] + qpos[backlash]`——穿过背隙读数。
  观测/动作维度不变，ONNX 导出和运行时零改动

### 域随机化清单（velocity cfg 的 `ENABLE_*`，全部有具体数字）

| 项 | 范围 | 备注 |
|---|---|---|
| 躯干+头部总成 CoM | ±3mm 起步，课程升到 ±8mm | 头部总成逐体随机；`bearing_roll` 是历史误列（右髋链接）但为保持 DR 行为保留 |
| 质量+惯量 | ±5%（同时） | |
| 折算转子惯量 armature | ±10% | BAM 下 armature 是真被设置的（不像 frictionloss 被归零） |
| 关节摩擦 | ±10%（乘 BAM 摩擦预算） | |
| 速度推搡 | ±0.3 m/s，每 3–6s | **曾是 ±0.5**——比最大步速（0.4）还大的叠加冲击会训练出始终处于防摔状态的步态（2026-07 审计后收窄） |
| IMU 安装误差 | ≤6° 随机轴 | **零中心**：训练的是对误差*幅值*的容忍，补偿不了系统性安装偏置——真机 ~5° 俯仰偏置在运行时源头修正 |
| 编码器偏置 | ±0.86°/关节，逐环境恒定 | actor 观测看到 `joint_pos + bias` |
| 地形 | 台阶 ≤1.5cm、网格 ≤1cm、坡 1.7°–5.7° | |
| 脚底摩擦 | (0.7, 1.3) | 从 (0.3, 1.2) 收窄——更抓地的脚垫 |

**两条 DR 铁律**（AGENTS.md"不变量"）：DR 绝不能跨 reset 累积（曾有一个累积式 CoM
随机器让所有长跑劣化了数月）；观测若重映射到传感器视角（背隙编码器、偏置），同一量的
**跟踪奖励必须量同一个视角**——否则策略因纠正自己看到的东西而被惩罚。

### 观测与指令契约（训练侧的另一半）

- 61 维 = 48 本体感知 + 13 指令块 `[twist(3), head_pose(4), body_pose(6)]`，**全族
  共享**；不用某指令槽的环境**零填充而非删除**（保留 obs 项、采样小范围）
- **死权重规则**：从未非零的指令输入，其输入神经元永远死着——每个指令槽从第 0 步
  就保持小的非零采样范围（即使奖励权重为 0），为将来的课程保活
- **零指令要显式训练**（`zero_command_prob` 式精确采零）：均匀采样几乎采不到全零
  指令——而那正是部署的空闲态
- **原地转圈**（lin=0, |ang| 大）占 15% 的桶：独立均匀采样让它只占 ~2% 经验，永远学不会

### 奖励设计：velocity 配方的具体数字

| 项 | 权重 | 参数 |
|---|---|---|
| 线速度跟踪 | 2.0 | std=√0.1 |
| 角速度跟踪 | 2.0 | std=√0.5 |
| 姿态 pose | 1.0 | **只作用于腿**（头/颈是指令驱动的——混进去会把头部跟踪奖到压过指令，收敛成"无视指令"）；行/走两档 std，阈值 0.01 |
| 直立 upright | 2.0 | std=√0.05——原本 1.0/std²=0.1 下 4° 倾斜只罚 ~0.05/步，形同免费 |
| 空中时间 air_time | 3.0 | 0.125–0.300s 计入 |
| 动作平滑 action_rate | -0.1（stage-0，课程渐升） | 技能发现**之后**才引入——探索期施加惩罚会使"什么都不做"成为最优 |
| 足底打滑 foot_slip | -0.1 | |
| 自碰撞 self_collisions | -1.0 | 精确点名电池座等真实碰撞对 |
| body_ang_vel / angular_momentum | -0.05 / -0.02 | **动态任务要压低**——它们惩罚的正是动态动作物理上需要的东西 |

### 奖励设计的实战教训（AGENTS.md，每条对应一次真实失败）

1. **符号约定曾导致四个环境出错**：自惩罚函数（返回 ≤0）配**正**权重；mjlab 基座
   代价函数（返回 ≥0）配**负**权重。自惩罚配负权重 = 双重否定 = 奖励违规行为，策略
   会反复利用该奖励（用臀部跳跃移动、故意摔倒坐地）。**可靠的检查方法：每次运行，
   wandb 里每个 `Episode_Reward/<penalty>` 必须 ≤ 0**
2. **RL 优化奖励的字面定义**：每个未明确约束的自由度都会被利用（弹道式甩动代替
   翻滚、肩部侧滚代替矢状面运动、头部三点支撑代替站立）——用硬状态门（支撑接触、
   姿态轴检查、闩锁）编码"什么算完成了动作"，不要用小惩罚去引导
3. **杜绝"巨额奖励"**：任何"到达 X"的奖励必须限速——提前到达目标状态后按步累积
   奖励，等于允许用任意激烈的动作换取高额回报。指令式转换用**斜坡内部目标**
   （恒速混合）：领先斜坡零收益，所以"慢"才是最优
4. **永远不要以坏状态（摔倒、低位）作为正奖励的门控**——策略会停留在成本最低的
   合格姿态中反复获取奖励。应使用势能塑形（按进展增量计分，如 Δcos(tilt)：
   起身得分、保持为零、无法刷取）
5. **姿态落地任务**：单一固定目标 + 关节/高度的高斯+L1 + |a_z| 落地冲击罚——**不是**
   关键帧/航点轨迹（策略会在航点上停留不走）。路径正是 RL 应当自行发现的部分
6. **正则项分两种**：动作阻塞型（body_ang_vel 等）对动态任务压低；平滑型
   （action_rate）安全但要在技能发现后从 ~0 课程引入
7. **移植正则项时比较奖励总量而非权重**：PPO 看相对优势——同一 action_rate 权重在
   4 倍大的正任务栈下相对强度弱 4 倍
8. **跟踪高斯 std ≈ 你仍在意的误差**，不是最大误差；但收紧前先问误差是策略可逃的
   还是行为固有的——占体重 38% 的头走路**必然**摆动，收紧瞬时头跟踪 std 曾把步态
   罚到原地不动；只定价可逃的部分（如对 1s EMA 收 L1——罚直流偏置，放过振荡）
9. **关节停在硬限位**：用 qpos 侧的限位接近度罚——库存 `dof_pos_limits` 只在最后
   ~7.5% 行程触发，指令侧罚无效（宽 ctrlrange 是故意的：低 kp 舵机需要过冲）

### 训练流程与运维

- **先跑冒烟测试**（64 envs × 5 iters，"几分钱抓住 ~95% 配置错误"），再上长跑
- 预算：简单一次性技能 ≈ **1000 iters** @4096 envs；步态和课程重的恢复类要
  **4000–6000**
- 读 wandb：均值奖励升 **且** 每个惩罚项 ≤ 0 **且** 主任务项真的在长（总奖励可以
  纯靠正则项上涨而技能从未发生）；`Episode_Reward/<term>` 记的是**加权后**值
- **先测量再理论**：跑"失败"了先做 headless 评估（分出生类型的测试组、终态聚类、
  角速度剖面）再改奖励——过去的"失败"事后证明是：checkpoint 太早、成功判据将同一
  行为簇一分为二、奖励上限与实测物理特性相冲突
- **逆向课程出生点**（从动作中途、包括接近完成处开始 episode）是"学会了开头、
  学不会最后一里"的可靠解法——否则前沿拿不到 on-policy 数据
- 课程阶段切换要看策略**实际学会没有**：某指标恰在课程边界处下滑 = 节奏错了，
  拉长阶段或推迟引入，绝不提前

### 导出与发布（之二"唯一合法路径"的实现侧）

- `export.py` 调 `runner.export_policy_to_onnx`，产出的是 `actor(normalizer(obs))`
  ——**机器人跑的就是训练看到的**。sim 内 `play` 自己会套归一化器，所以手工转换
  忘了归一化在 sim 里看不出来，只有在真机上才会暴露
- `publish` **直接调 `run_export`**，发布的策略不可能跳过这一步；上传前验证
  `[1,61]→[1,14]`（51 维遗留被点名拒绝）、合理输入试跑拒 NaN/常数输出、从 git+wandb
  填 `training` 块（task/commit/branch/dirty/run/checkpoint）、拒绝无 `--force` 覆盖
  已有 onnx、默认建私有仓库

### 部署预演：`scripts/infer_policy.py`（上真机前的最后一道验证）

CPU MuJoCo 里用**训练同款 BAM M6 执行器**跑导出的 ONNX，键盘驾驶（速度指令、
G 捡地/Y 坐站/R 翻滚/K/L 踢球），关键能力：

- **热切换预演**：`--walking walk.onnx --standing stand.onnx --sitstand sitstand.onnx
  --roulade roulade.onnx`——与运行时 walk/recover/trick 策略切换的行为一致
- `--vin/--vin-drop-gain/--kp-fw` 把训练 DR 范围固定为一个值（模拟特定电量/摩擦的
  个体差异）；`--no-bam` 退回 XML PD
- `--record/--save-csv` 支持 sim2real 对比

**契约的教训**（AGENTS.md 收尾条目）：posture flag 住在 twist 的 vx 槽里——输入
全零意味着"站"，看起来就像"策略无视按钮"。预演时要用**正确的指令槽写入**，这正是
它存在的意义。

### 技术栈分层详解

```
┌ ⑥ 部署契约层   ONNX 导出 · infer_policy CPU 预演 · duck-body 仿真躯体服务器
├ ⑤ 工具链层     uv · wandb · Hugging Face (Jobs+Hub) · matplotlib
├ ④ 算法层       PPO (rsl_rl) · Actor/Critic MLP
├ ③ 机器人层     BAM 执行器模型 · onshape-to-robot CAD→MJCF · rustypot 真机测试台
├ ② 训练框架层   mjlab 1.3.0 (Manager-Based RL)
└ ① 仿真内核层   MuJoCo 3.10 + mujoco-warp 3.8.1 (GPU 并行) · NVIDIA Warp 1.12
```

#### ① 仿真内核：MuJoCo Warp——GPU 并行的物理

- **mujoco-warp** 把 MuJoCo 物理搬到 NVIDIA **Warp**（GPU kernel 语言）上：4096 个环境
  在一个 GPU 上同步步进，这是"1–2 小时出一个步态"的算力来源
- **Warp 1.12 与 torch 的零拷贝互操作**是选型的隐藏约束：cu129（不是 cu130）索引正是
  为了和 warp 捆绑的 CUDA 工具链同大版本——错开大版本，张量在 GPU 间就得拷贝
- 物理特性跟版本走：mujoco 3.10 的 `mjDSBL_MULTICCD` 被 mujoco-warp 引用——曾有
  override 把 mujoco 强降到 3.4.0，import 即崩（所以 pyproject 明令"不要 override mujoco"）

#### ② 训练框架：mjlab——MuJoCo 官方团队的 Manager-Based RL

- Manager-Based 架构：奖励/事件（DR）/课程/观测各由一个 manager 管，任务 = 一份 cfg
  文件——所以 13 个任务全是 `microduck_*_env_cfg.py` 配置而非代码
- 自带地形生成器（台阶/网格/坡度）、任务注册表（`uv run list-envs`）、entry-point
  插件机制（`mjlab.tasks`）
- **本仓库本身就是 mjlab 的插件包**；`train` script 故意与 mjlab 同名碰撞
  （pyproject 注释记了为什么删不得：同名 console script 是 last-writer-wins，删了这行
  mjlab 的也不会回来——`uv sync` 卸掉我们的之后没人重建 mjlab 的，`uv run train`
  会找到 liblinear 的 `train`。`tests/test_hf_jobs_flag.py` 锁定该行为）

#### ③ 机器人层：三个"真源"工具

- **BAM**（`better-actuator-models`，Rhoban）：M6 XL330 模型。走 git 源
  （`mjlab_frictionloss` 分支），PyPI 版还没有 mjlab 接口
- **onshape-to-robot**：CAD 直接导 MJCF——几何和惯量从设计文件来，不是手抄
- **rustypot**：与**主仓库 daemon 同一个 Rust Dynamixel 库**（训练仓库的依赖里有它）
  ——`testbench_sim2real.py` / `validate_bam_testbench.py` 用真舵机验证 BAM 模型
  本身：sim2real 的可信度有实测背书，而非依赖假设

#### ④ 算法层：PPO 的具体配置

| 项 | 值 |
|---|---|
| Actor / Critic | 各自独立 MLP `(512, 256, 128)`，ELU |
| 学习率 / γ / λ | 1e-3 / 0.99 / 0.95 |
| 熵系数 | 0.01 |
| minibatch | 4；`NUM_STEPS_PER_ENV = 24` |
| 有效批量 | 4096 envs × 24 steps ≈ **10 万步/迭代** |
| 预算 | 一次性技能 ~1000 迭代；步态/恢复类 4000–6000 |

架构是最朴素的 MLP + PPO——这个项目证明了在 800 g 双足上，**sim2real 的胜负在
奖励设计和执行器保真度，不在算法花哨**。

#### ⑤ 工具链层：依赖治理本身是文档

`pyproject.toml` 的注释密度堪比设计文档，每条 override 都是一次真实故障：

| 故障 | 修法 |
|---|---|
| BAM 限定 `protobuf<4`（为它自己的 zmq 消息，训练根本不用）→ 级联把 onnx 降到 1.17.0（无 wheel 编不过） | override `protobuf>=4` + `onnx>=1.20.1` |
| BAM 依赖的 PyPI `zmq==0.0.0` stub 轮子损坏，**热缓存忍得了、HF Jobs 冷装就死** | `zmq ; python_version < '3.0'`（恒假 marker 踢掉） |
| Python 上限未限定 → HF Jobs 选了 3.13.14，与本地测的 3.12 不一致 | `>=3.12, <3.13` |
| mjlab 1.3.0 import scipy 但没声明 → 冷 `uv sync` 连 import 都过不了 | 显式补 `scipy>=1.16` |
| torch 是传递依赖 → `[tool.uv.sources]` 的 aarch64 路由**静默失效** | 提为直接依赖 + `==2.9.1` 精确锁定版本 |

核心哲学（pyproject 原注释）：**"fresh `uv sync` 是 ground truth（HF Jobs 跑的就是
它）——任何只靠本地手工装的包远程必死"**。

#### ⑥ 部署契约层：三段式，最后一段是仿真躯体

1. **导出**：`runner.export_policy_to_onnx` 产出 `actor(normalizer(obs))`，
   `[1,61]→[1,14]`
2. **预演**：`infer_policy.py` 在 CPU MuJoCo（训练同款 BAM 执行器）里热切换多个
   ONNX，键盘驾驶，`--record/--save-csv` 做 sim2real 对比
3. **仿真躯体（`sim/body_server.py`，`uv run duck-body`）**：

```bash
uv run duck-body --ducks 4        # MuJoCo 躯体服务器：一鸭一端口、
                                  # 共享一个世界、能互相撞
robotd --sim 127.0.0.1:7801       # 主仓库侧：daemon 连仿真躯体
```

- **关键设计**：`RobotIo` trait 之上的一切都是真代码——50 Hz 循环、ONNX 策略、
  安全、跌倒检测、里程计、运动学、全部 IPC，只有这个 TCP 进程知道没有真鸭子
- 协议是换行 JSON + 握手查版本（两端分属两个仓库，"你的模拟器旧了"和"你的 daemon
  旧了"不能是同一种症状）；TCP 而非 unix socket 的理由在主仓库 `duck_control::sim`
- `sim/tof.py`：仿真 VL53L5CX 连**每区状态字节**都仿真——"没东西"≠"测不到"，
  只报距离的模拟器会放走硬件才能抓到的 bug；噪声随距离增长（近处毫米级、
  4m 限附近厘米级），数字来自数据手册和桌面实测

**上游 duck-sim 栈**（本地 clone 的 tag `daemon-dev-sim-remote-io`，apirrone
2026-09-02，**截至本仓库 main `bc41fb5` 尚未合入**）已经长成完整多鸭系统：
`scripts/duck-sim up` 一鸭一容器、`DUCK_SIM_SCENE=apartment`（公寓房间）、
`DUCK_SIM_DUCKS=2`、`DUCK_SIM_CAMERAS=all`（每鸭一个 mediad + 仿真眼 + 独立控制台
`:8080/:8081`）、`tofd --sim`（仿真深度帧，`sensor: "sim"`，`tof.frame` 格式与真
传感器逐字节相同）——多只鸭子在同一世界中交互的完整机器人栈仿真。

> 对 Pi 验证的意义：`--fake` 是"无机器人"的最低档，`--sim` 是其更完整的形态——控制环、
> 策略推理、安全层全部对着仿真躯体真跑。切到该 dev tag（或等它进 main）后，Pi 上
> 可验证比 runbook 现在所写更接近真实的东西。

---

## 之四：策略网络本体——规格与真实 ONNX 解剖

> 依据：`microduck_velocity_env_cfg.py` 的 RL 配置、`duck-control/src/{obs,policy}.rs`、
> `robotd/src/control.rs`，以及对官方 `alpha_walking.onnx`（HF Hub，经 hf-mirror 下载，
> 776 KB）的实际图结构解析（2026-09-04，onnx Python 库）。

### 网络规格：一个小型前馈 MLP

| 项 | 值 |
|---|---|
| 结构 | `61 → 512 → 256 → 128 → 14`，纯全连接 |
| 参数量 | **约 19.8 万**（≈0.8 MB @ fp32；实测 alpha_walking.onnx 为 776 KB，吻合） |
| 激活函数 | ELU（隐层），输出层线性（无激活） |
| 策略分布 | `GaussianDistribution`，`init_std=1.0`，scalar std（训练期探索用；部署时取均值动作） |
| Critic | 同构 `(512,256,128)`，**不随部署携带**——只有 actor 上机 |
| 图形状 | ONNX `[1,61] → [1,14]`，导出时强制验证 |

没有 LSTM、没有 Transformer、没有注意力——"会走路"的全部知识压缩在约 20 万个
浮点数里。

### 输入 61 维逐位定义（`duck-control/src/obs.rs`，自称"本 crate 最高危代码"）

```
 0..3    陀螺仪（躯干坐标系，rad/s）
 3..6    重力投影（单位向量；直立 = [0,0,-1]）
 6..20   14 个关节位置 − home 姿态（嘴被排除）
20..34   14 个关节速度
34..48   上一步的动作（14）
48..61   指令块：twist(3) + head(4) + body(6)
```

- **纯本体感知**：没有视觉、没有深度、没有外部感知——策略仅凭陀螺仪和关节编码器
  行走
- 上一步动作作为输入构成短期记忆，用于补偿系统延迟
- 61 维宽度是训练与部署的**编译期共享契约**（`duck-ipc-proto` 持有相同常量，
  `assert!` 保证相等）

### 输出：相对 home 姿态的偏移

```
舵机目标角 = home 姿态 + action_scale × action
```

- `action_scale` 为运行时可调参数；站立使用更小的 scale 与软化增益，踢球窗口沿用
  站立调参（历史行为，踢球即按该组参数训练，有意保留）
- 输出经安全层（关节限位、温度、跌倒检测）钳位后才写入总线
- 14 维不含嘴，嘴由独立逻辑控制

### 训练超参（补充之二/之三未列全的部分）

| 项 | 值 |
|---|---|
| PPO clip | 0.2，`use_clipped_value_loss`，`value_loss_coef=1.0` |
| 学习率调度 | **自适应**，按 KL 散度目标 `desired_kl=0.01` 调节 |
| 每次更新的 epoch | 5 |
| 梯度裁剪 | `max_grad_norm=1.0` |
| 采样 | 4096 envs × 24 steps ≈ 10 万步/迭代，4 minibatches |

### 真实解剖：`alpha_walking.onnx` 的完整计算图

**完整图只有 9 个算子**——行走能力的全部计算路径：

```
输入 obs [1,61]
  │
  ├─ Sub   (obs − obs_normalizer._mean)     ┐ 观测归一化器
  ├─ Div   (÷ std)                          ┘ 烘焙进图的前两步
  │
  ├─ Gemm  [61→512]  + Elu                  ┐
  ├─ Gemm  [512→256] + Elu                  │ MLP 主体
  ├─ Gemm  [256→128] + Elu                  ┘
  ├─ Gemm  [128→14]                          （线性输出）
  │
  └─ 输出 actions [1,14]
```

10 个权重张量：

```
obs_normalizer._mean  [1,61]     ← 训练期统计的 61 个均值
onnx::Div_24          [1,61]     ← 61 个标准差
mlp.0.weight [512,61]  mlp.0.bias [512]
mlp.2.weight [256,512] mlp.2.bias [256]
mlp.4.weight [128,256] mlp.4.bias [128]
mlp.6.weight [14,128]  mlp.6.bias [14]
```

查看工具：`https://netron.app`（网页版）可直接可视化整个图与每层权重。

### "模型实现是数据文件，不是代码"

两个仓库中**均无手写的网络实现**，三个组成部分各司其职：

| 组成 | 位置 | 职责 |
|---|---|---|
| 网络定义 | 第三方包 **rsl_rl**（`leggedrobotics/rsl_rl` 的 `ActorCritic` + `GaussianDistribution`，pip 安装） | 定义 `nn.Module` 的搭建，训练时前向/反向 |
| 序列化 | `microduck_rl/export.py`（`torch.onnx.export`） | 把训好的模块导出为上图的 9 算子图，归一化器一并烘焙 |
| 执行 | 第三方库 **ONNX Runtime**（`duck-control/policy.rs` 经 `ort` dlopen `libonnxruntime.so`） | 运行时执行 9 个算子；`policy.rs` 仅为调用编排（加载/验证/喂输入/取输出） |

设计含义：**只要图形状为 `[1,61]→[1,14]`，训练侧任意改动网络结构（增大 MLP、增加
层数）都不影响部署侧代码**——这正是全族策略热切换、社区策略即插即用得以成立的
基础。
