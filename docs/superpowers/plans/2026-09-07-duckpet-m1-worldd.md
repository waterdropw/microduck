# DuckPet M1（worldd 会话网关）实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 交付 spec（`docs/superpowers/specs/2026-09-07-duckpet-mp-mvp-design.md`）的 M1 里程碑——一个 `worldd` 服务，能按需启动仿真世界容器，经 WebSocket 以 30Hz 下发关节状态流、接收驾驶意图；验收标准是**用 wscat 在终端里驾驶一只云鸭子**。

**Architecture:** 单个 Rust 二进制 `worldd`（axum）：HTTP 端点创建/回收会话，WebSocket 端点做协议翻译。每个会话对应一个 Docker 容器（`duckpet-world` 镜像：duck-body + robotd --sim + 官方策略集，即 duck-sim 已验证形态的产品化封装）。状态桥经 robotd 的 unix socket 订阅 `robot.subscribe {hz:30}`。

**Tech Stack:** Rust 2024 · axum 0.8 (ws) · tokio · serde_json · duck-ipc-proto（path 依赖）· Docker

## Global Constraints

- 代码在新仓库 `~/repos/duckpet`（与 microduck 平级）；每个任务一次提交
- `duck-ipc-proto = { path = "../microduck/duck-ipc-proto" }`——协议类型只用这个 crate 的，禁止手写 JSON 字段
- **多人预留（spec §6a）**：状态下行消息按代理**数组**编码（v1 恒为长度 1）；意图携带 `duck_id`
- 关节顺序契约 = `duck_ipc_proto::JOINT_NAMES`（15 元素），版本随 `API_VERSION`
- 仿真依赖的现成资产（本机已就绪）：`~/repos/microduck-sim`（robotd 二进制 + policies/）、`~/repos/microduck_rl`（sim 包源码）
- 30Hz 降采样用协议原生能力 `robot.subscribe {hz:30}`，**不自建节流**

---

### Task 1: 仓库脚手架 + /healthz

**Files:**
- Create: `~/repos/duckpet/Cargo.toml`（workspace + worldd crate）
- Create: `~/repos/duckpet/worldd/Cargo.toml`
- Create: `~/repos/duckpet/worldd/src/main.rs`
- Create: `~/repos/duckpet/worldd/src/lib.rs`

**Interfaces:**
- Produces: `worldd::app() -> axum::Router`（后续任务往里挂路由）；二进制监听 `127.0.0.1:7800`

- [ ] **Step 1: 建仓库与 crate**

```bash
mkdir -p ~/repos/duckpet/worldd/src && cd ~/repos/duckpet && git init
cat > Cargo.toml <<'EOF'
[workspace]
resolver = "3"
members = ["worldd"]
EOF
cat > worldd/Cargo.toml <<'EOF'
[package]
name = "worldd"
version = "0.1.0"
edition = "2024"
rust-version = "1.89"

[dependencies]
axum = { version = "0.8", features = ["ws"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tracing = "0.1"
tracing-subscriber = "0.3"
duck-ipc-proto = { path = "../microduck/duck-ipc-proto" }
async-trait = "0.1"

[dev-dependencies]
tokio-tungstenite = "0.24"
futures-util = "0.3"
tempfile = "3"
EOF
```

- [ ] **Step 2: 写失败测试（healthz）**

`worldd/src/lib.rs`：

```rust
use axum::{Router, routing::get, http::StatusCode};

pub fn app() -> Router {
    Router::new().route("/healthz", get(|| async { "ok" }))
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum::body::Body;
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    #[tokio::test]
    async fn healthz_answers_ok() {
        let res = app().oneshot(
            axum::http::Request::builder().uri("/healthz").body(Body::empty()).unwrap()
        ).await.unwrap();
        assert_eq!(res.status(), StatusCode::OK);
        let body = res.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"ok");
    }
}
```

注意：`tower::ServiceExt::oneshot` 需要 `axum` 的测试支持——在 `worldd/Cargo.toml` 的 `[dev-dependencies]` 加 `tower = { version = "0.5", features = ["util"] }` 和 `http-body-util = "0.1"`。

- [ ] **Step 3: 跑测试确认通过**

```bash
cd ~/repos/duckpet && cargo test
```
预期：1 passed。`main.rs` 先放最小 `fn main() { let _ = worldd::app(); }`。

- [ ] **Step 4: main.rs 起服务**

```rust
use worldd::app;

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();
    let listener = tokio::net::TcpListener::bind("127.0.0.1:7800").await.unwrap();
    tracing::info!("worldd on 127.0.0.1:7800");
    axum::serve(listener, app()).await.unwrap();
}
```

- [ ] **Step 5: 手动验证后提交**

```bash
cargo run &; sleep 1; curl -s localhost:7800/healthz   # 期望: ok; kill %1
git add -A && git commit -m "worldd scaffold with healthz"
```

---

### Task 2: RobotdLink——robotd 的订阅/意图客户端

**Files:**
- Create: `worldd/src/link.rs`
- Modify: `worldd/src/lib.rs`（`pub mod link;`）

**Interfaces:**
- Produces:
  - `pub trait RobotLink: Send + Sync`，方法 `async fn state(&self) -> tokio::sync::watch::Receiver<duck_ipc_proto::RobotState>`、`async fn send_move(&self, m: duck_ipc_proto::MoveParams) -> anyhow-free Result<(), String>`、`async fn send_do(&self, skill: &str) -> Result<(), String>`
  - `pub struct RobotdLink; pub async fn RobotdLink::connect(socket: &Path, hz: u32) -> Result<RobotdLink, String>`——连接、发 `robot.subscribe {hz}`、确认 ack、启动读循环把 `robot.state` 通知写进 watch
- Consumes: `duck_ipc_proto::{Request, Response, Call, RobotState, MoveParams, DoParams, SubscribeParams, Id}`；`Request::call(id, &Call)` / `Request::notify(&Call)`（构造消息只用这两个，禁止手拼 JSON）

**背景（实现者需要知道的协议事实）**：robotd 的 socket 是 NDJSON JSON-RPC：一行一对象。连接后无需 hello。`robot.subscribe {hz: N}` 请求会收到 ack，随后连接上开始收到无 `id` 的 `robot.state` 通知（`RobotState` 含 `t/joints(15,JOINT_NAMES 序)/odom{position,yaw}/…`）。`robot.move` 是**通知**（无应答），`robot.do {skill}` 是**请求**（有应答，拒绝时带原因）。

- [ ] **Step 1: 写失败测试——对着一个 stub robotd**

`worldd/src/link.rs` 测试内嵌一个 stub 服务端（tokio UnixListener），行为：收第一行（subscribe 请求）回 ack，然后推两条 `robot.state` 通知；随后读一行断言是 `robot.move` 通知：

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use duck_ipc_proto as proto;
    use std::os::unix::net::UnixListener;
    use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};

    fn sample_state(t: f64) -> proto::RobotState {
        proto::RobotState {
            t,
            movement: Default::default(),
            head: [0.0; 4],
            policy: "walk".into(),
            safety: Default::default(),
            control_loop: Default::default(),
            joints: vec![0.01; 15],
            targets: vec![0.0; 15],
            odom: Default::default(),
        }
    }

    #[tokio::test]
    async fn subscribes_streams_state_and_relays_move() {
        let dir = tempfile::tempdir().unwrap();
        let sock = dir.path().join("robotd.sock");
        let listener = UnixListener::bind(&sock).unwrap();
        let sock2 = sock.clone();

        let server = tokio::spawn(async move {
            let (stream, _) = listener.accept().unwrap();
            let stream = tokio::IntoAsyncRead::into_async_read(stream); // UnixListener 是同步的，测试里换 tokio::net::UnixListener
            todo!("见 Step 2 的说明：服务端用 tokio::net::UnixListener")
        });
        let _ = sock2;
        let _ = server;
        todo!()
    }
}
```

实现者注意：上面是骨架示意——**用 `tokio::net::UnixListener`** 写 stub（`accept().await`），循环 `read_line`：

```rust
let listener = tokio::net::UnixListener::bind(&sock).unwrap();
tokio::spawn(async move {
    let (mut stream, _) = listener.accept().await.unwrap();
    let (mut r, mut w) = stream.split();
    let mut r = BufReader::new(&mut r).lines();
    // 1) 第一行 = subscribe 请求 → 回 ack
    let line = r.next().await.unwrap().unwrap();
    let req: proto::Request = serde_json::from_str(&line).unwrap();
    assert_eq!(req.method, "robot.subscribe");
    let id = req.id.clone();
    w.write_all(
        format!("{}\n", serde_json::json!({"jsonrpc":"2.0","id":id,"result":{}})).as_bytes()
    ).await.unwrap();
    // 2) 推两条 state 通知
    for t in [1.0_f64, 2.0] {
        let n = proto::Request::notify_state(&sample_state(t));
        w.write_all(format!("{}\n", serde_json::to_string(&n).unwrap()).as_bytes()).await.unwrap();
    }
    // 3) 读一条 move 通知并断言
    let line = r.next().await.unwrap().unwrap();
    let n: proto::Request = serde_json::from_str(&line).unwrap();
    assert_eq!(n.method, "robot.move");
    let m: proto::MoveParams = serde_json::from_value(n.params.unwrap()).unwrap();
    assert_eq!(m.vx, 0.3);
});
```

主测试体：

```rust
let link = RobotdLink::connect(&sock, 30).await.unwrap();
let mut rx = link.state().await;
// watch 收到的是最新值——等第二条覆盖
tokio::time::sleep(std::time::Duration::from_millis(100)).await;
assert_eq!(rx.borrow().t, 2.0);
assert_eq!(rx.borrow().joints.len(), 15);
link.send_move(proto::MoveParams { vx: 0.3, vy: 0.0, vyaw: 0.0 }).await.unwrap();
tokio::time::sleep(std::time::Duration::from_millis(100)).await;
```

- [ ] **Step 2: 跑测试确认失败**

```bash
cargo test -p worldd link
```
预期：编译失败（`RobotdLink` 未定义）。

- [ ] **Step 3: 实现 link.rs**

```rust
use duck_ipc_proto as proto;
use std::path::Path;
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::UnixStream;
use tokio::sync::{watch, Mutex};

pub struct RobotdLink {
    latest: watch::Receiver<proto::RobotState>,
    writer: Mutex<OwnedWriteHalfAlias>,
}
```

实现要点（每条都写全，不要省略）：
- `connect`: `UnixStream::connect` → `split()`；写 `Request::call(Id::Number(1), &Call::RobotSubscribe(SubscribeParams{hz: Some(hz)}))` + `\n`；读一行解析 `Response`，`result.is_some()` 才算成功
- 读任务（`tokio::spawn`）：循环 `read_line`，`serde_json::from_str::<proto::Request>`，`method == "robot.state"` 时 `serde_json::from_value::<RobotState>(params)` 成功则 `watch::Sender::send`（接收方落后就 `send_replace` 语义，天然 latest-wins）
- `send_move`: `Request::notify(&Call::RobotMove(m))` 序列化 + `\n` 写出（锁 writer）
- `send_do`: `Request::call(自增 id, &Call::RobotDo(DoParams{skill: skill.into()}))`，读一行应答；`error` 字段存在则返回 `Err(message)`
- writer 半部包 `Mutex`；读任务持有 read 半部

- [ ] **Step 4: 跑测试确认通过**

```bash
cargo test -p worldd link
```
预期：1 passed。

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "worldd: robotd link — subscribe at hz, relay move/do"
```

---

### Task 3: WebSocket 协议与端点（对 stub link 可测）

**Files:**
- Create: `worldd/src/protocol.rs`（WS 消息类型）
- Create: `worldd/src/ws.rs`
- Modify: `worldd/src/lib.rs`（挂 `/ws/{world_id}` 路由）

**Interfaces:**
- Consumes: Task 2 的 `RobotLink` trait
- Produces:
  - 下行：`{"type":"state","world":"<id>","agents":[{"duck_id":"duck-a","t":f64,"joints":[15×f64],"x":f64,"y":f64,"z":f64,"yaw":f64}],"policy":"walk"}` —— **agents 是数组（多人预留）**
  - 上行：`{"type":"intent","duck_id":"duck-a","vx":f,"vy":f,"vyaw":f}` 与 `{"type":"skill","duck_id":"duck-a","name":"roulade"}`
  - `pub async fn ws_handler(State(link), WebSocketUpgrade, Path(world_id))`

**背景**：关节流 30Hz 来自 Task 2 的 watch（robotd 已按 hz 降采样，watch 又是 latest-wins，客户端变慢自然丢帧——与 robotd 自己的 STATE_BUFFER 哲学一致）。上行 intent 的 EMA 平滑**客户端负责**（M2 的事），worldd 原样转发。

- [ ] **Step 1: 写失败测试**

`worldd/src/ws.rs` 测试：构造 stub `RobotLink`（测试实现：watch channel 手动喂两个 state；`send_move` 记录到 `Arc<Mutex<Vec<MoveParams>>>`），把路由挂上，用 `tokio-tungstenite` 连 `/ws/w1`，断言：收到一条 `state` 消息、agents 长度 1、joints 15；发一条 intent，stub 侧收到 `vx=0.2`。

stub 实现（完整写出）：

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::link::RobotLink;
    use duck_ipc_proto::{MoveParams, RobotState};
    use std::sync::Arc;
    use tokio::sync::{watch, Mutex};

    struct StubLink {
        tx: watch::Sender<RobotState>,
        moves: Arc<Mutex<Vec<MoveParams>>>,
    }
    impl RobotLink for StubLink {
        async fn state(&self) -> watch::Receiver<RobotState> { self.tx.subscribe() }
        async fn send_move(&self, m: MoveParams) -> Result<(), String> {
            self.moves.lock().await.push(m); Ok(())
        }
        async fn send_do(&self, _skill: &str) -> Result<(), String> { Ok(()) }
    }
    // … 测试体：app = Router with ws route over Arc<StubLink>；
    // tungstenite connect → recv Text → parse protocol::WorldStateMsg；
    // send intent json → 断言 stub.moves == [MoveParams{vx:0.2,…}]
}
```

`protocol.rs` 类型（完整写出，serde camelCase 无需——字段名直接按下行 JSON）：

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
pub struct StateMsg {
    #[serde(rename = "type")]
    pub kind: &'static str, // "state"
    pub world: String,
    pub agents: Vec<AgentState>,
}

#[derive(Serialize)]
pub struct AgentState {
    pub duck_id: String,
    pub t: f64,
    pub joints: Vec<f64>,
    pub x: f64, pub y: f64, pub z: f64, pub yaw: f64,
    pub policy: String,
}

#[derive(Deserialize)]
#[serde(tag = "type", rename_all = "lowercase")]
pub enum ClientMsg {
    Intent { duck_id: String, vx: f64, vy: f64, vyaw: f64 },
    Skill  { duck_id: String, name: String },
}
```

`ws.rs` handler 逻辑：`socket.accept()` → spawn 下发任务（`watch::changed().await` 后组 `StateMsg{agents: vec![一个 agent]}` 发 Text）→ 主循环收上行：Intent → `link.send_move`；Skill → `link.send_do`；**duck_id 不等于本世界唯一的鸭子时忽略**（多人预留：v1 校验恒真，但校验逻辑存在）。

- [ ] **Step 2: 跑测试确认失败 → Step 3: 实现 → Step 4: 跑测试通过**

```bash
cargo test -p worldd ws
```

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "worldd: websocket endpoint — agent-array state downstream, intents upstream"
```

---

### Task 4: World 启动器（docker 命令构造 + 启动）

**Files:**
- Create: `worldd/src/launcher.rs`
- Modify: `worldd/src/lib.rs`

**Interfaces:**
- Produces: `pub struct WorldLauncher { image: String, state_root: PathBuf }`；
  - `pub fn argv_for(&self, world_id: &str, body_port: u16) -> Vec<String>`（纯函数，可单测）
  - `pub async fn start(&self, world_id: &str) -> Result<RobotdLink, String>`：`docker run -d --name duckpet-<id>` → 轮询 socket 文件出现（≤10s）→ `RobotdLink::connect(state_root/<id>/robotd.sock, 30)`
  - `pub async fn stop(&self, world_id: &str) -> Result<(), String>`：`docker rm -f`
- Consumes: Task 2 的 `RobotdLink::connect`

- [ ] **Step 1: 写失败测试（argv 构造，纯函数）**

```rust
#[test]
fn argv_mounts_state_dir_and_names_container() {
    let l = WorldLauncher { image: "duckpet-world:dev".into(),
                            state_root: "/tmp/duckpet".into() };
    let argv = l.argv_for("w1", 7801);
    let joined = argv.join(" ");
    assert!(joined.contains("--name duckpet-w1"));
    assert!(joined.contains("-v /tmp/duckpet/w1:/state"));
    assert!(joined.contains("duckpet-world:dev"));
}
```

- [ ] **Step 2-4: 失败 → 实现 → 通过**。`argv_for` 完整输出（顺序固定，便于断言）：

```
docker run -d --name duckpet-<id>
  -v <state_root>/<id>:/state
  --memory 512m --cpus 1.0
  <image>
```

（robotd socket 由容器内 entrypoint 建在 `/state/robotd.sock`；`/state` 属主在宿主侧预创建为 777 或映射 uid——M1 用 777，spec 安全项里注明。`start` 里 `std::fs::create_dir_all` 后 `docker run`，然后每 200ms 检查 `<state_root>/<id>/robotd.sock` 存在，最多 10s。）

- [ ] **Step 5: 提交** `git commit -m "worldd: world launcher over docker"`

---

### Task 5: duckpet-world 镜像

**Files:**
- Create: `~/repos/duckpet/world-image/Dockerfile`
- Create: `~/repos/duckpet/world-image/entrypoint.sh`
- Create: `~/repos/duckpet/world-image/robotd.toml.in`

**Interfaces:**
- Produces: 镜像 `duckpet-world:dev`——`docker run` 后约 5s 内 `/state/robotd.sock` 可连
- Consumes: `~/repos/microduck-sim/target/debug/robotd`、`~/repos/microduck-sim/policies/*.onnx`、`~/repos/microduck_rl/src/mjlab_microduck/sim/`、场景 `scene.xml`

- [ ] **Step 1: Dockerfile（完整）**

```dockerfile
# 一个世界 = duck-body + robotd --sim。基础镜像用已验证的 duck-dev-build
# （rust:bookworm + 必要库，本机已构建）。
FROM duck-dev-build

RUN apt-get update && apt-get install -y --no-install-recommends \
        python3 python3-pip && rm -rf /var/lib/apt/lists/*
RUN pip3 install --break-system-packages --no-cache-dir \
        mujoco numpy onnxruntime==1.29.0

# 仿真包（duck-body + tof 仿真器）：只需 src/mjlab_microduck/sim 与 robot 模型
COPY mjlab_microduck/ /opt/sim/mjlab_microduck/
RUN echo /opt/sim > /usr/lib/python3/dist-packages/mjlab_microduck.pth

# daemon 与策略（从构建好的 sim worktree 拷入，见下方构建命令）
COPY robotd /opt/duck/robotd
COPY policies/ /opt/duck/policies/
COPY robotd.toml.in /opt/duck/robotd.toml.in
COPY entrypoint.sh /opt/duck/entrypoint.sh
RUN chmod +x /opt/duck/robotd /opt/duck/entrypoint.sh

ENTRYPOINT ["/opt/duck/entrypoint.sh"]
```

- [ ] **Step 2: entrypoint.sh（完整）**

```sh
#!/bin/sh
# 一个世界：MuJoCo 躯体在 7801，真 robotd 对着它，socket 放到 /state 供网关连接。
set -eu
mkdir -p /state

# robotd.toml：每个策略点名（RELEASE_DIR 在容器里不存在，路径必须逐个指定）
sed -e "s|@POLICIES@|/opt/duck/policies|g" /opt/duck/robotd.toml.in > /state/robotd.toml

ORT="$(ls /usr/lib/python3/dist-packages/onnxruntime/capi/libonnxruntime.so.* 2>/dev/null \
    || ls /usr/local/lib/python3*/dist-packages/onnxruntime/capi/libonnxruntime.so.* | head -1)"

# 躯体（headless；无摄像头）
setsid nohup python3 -m mjlab_microduck.sim.body_server \
    --port 7801 --ducks 1 --keyframe SIT --headless \
    --scene /opt/sim/mjlab_microduck/robot/microduck/scene.xml \
    > /state/body.log 2>&1 &

# 等躯体端口
i=0; while [ $i -lt 50 ]; do
    python3 -c "import socket;s=socket.socket();s.settimeout(.3);exit(0 if s.connect_ex(('127.0.0.1',7801))==0 else 1)" \
        && break; i=$((i+1)); sleep 0.2; done

# 真 robotd——与机器人上运行的二进制相同，只有电机换成 TCP 对端的 MuJoCo
exec env ORT_DYLIB_PATH="$ORT" RUST_LOG=info \
    /opt/duck/robotd --sim 127.0.0.1:7801 \
    --params /state/robotd.toml --socket /state/robotd.sock
```

`robotd.toml.in`（策略段完整七行，字段名照抄 duck-sim 生成的写法——`[policy] enabled / walk / stand / sitstand / ground_pick / kick_left / kick_right / roulade`，路径 `@POLICIES@/alpha_walking.onnx` 等；另加 `[chorale] accept = true`）。

- [ ] **Step 3: 构建并手测**

```bash
cd ~/repos/duckpet/world-image
mkdir -p ctx && cp -r ~/repos/microduck_rl/src/mjlab_microduck ctx/
cp ~/repos/microduck-sim/target/debug/robotd ctx/
cp -r ~/repos/microduck-sim/policies ctx/
cp entrypoint.sh robotd.toml.in ctx/
docker build -t duckpet-world:dev -f Dockerfile ctx/

mkdir -p /tmp/wtest && docker run -d --name duckpet-wtest -v /tmp/wtest:/state duckpet-world:dev
sleep 6 && docker exec duckpet-wtest ls /state/   # 期望: robotd.sock body.log robotd.toml
docker rm -f duckpet-wtest
```

- [ ] **Step 4: 提交**（把 world-image 目录与构建说明提交；ctx/ 进 .gitignore）

---

### Task 6: 会话管理 + 端到端验收（M1 的验收）

**Files:**
- Create: `worldd/src/session.rs`
- Modify: `worldd/src/lib.rs`、`worldd/src/main.rs`

**Interfaces:**
- Produces: `POST /sessions` → `{"world_id":"w-<8hex>","duck_id":"duck-a","ws":"/ws/w-<8hex>"}`（201）；`GET /healthz` 不变；会话闲置 10 分钟自动 `launcher.stop` 并从注册表摘除
- Consumes: Task 3 ws 路由（改为从注册表取该世界的 link）、Task 4 launcher、Task 5 镜像

- [ ] **Step 1: 写失败测试（注册表纯逻辑）**

`session.rs`：`SessionRegistry { launcher, worlds: Mutex<HashMap<String, Session>> }`，`Session { world_id, duck_id, link, last_seen: Arc<AtomicU64> }`；`create()` 起容器建 link；`touch(world_id)` 由 ws 收到消息时更新；后台任务每 60s 扫描，`last_seen` 超 600s 的 `stop`。单测：用注入假时钟或直接测 `is_expired(last_seen_epoch, now)` 纯函数。

- [ ] **Step 2-4: 失败 → 实现 → 通过**（ws_handler 改为 `State<Arc<SessionRegistry>>` + `Path(world_id)`，查不到世界返回 close code 4404）

- [ ] **Step 5: 端到端验收脚本 `scripts/e2e.sh`（完整写出）**

```sh
#!/bin/sh
# M1 验收：wscat 驾驶一只云鸭子。
set -eu
cd "$(dirname "$0")/.."
cargo build -q
./target/debug/worldd &
WORLD=$!; trap 'kill $WORLD' EXIT
sleep 1

RES=$(curl -s -X POST localhost:7800/sessions)
echo "$RES"
WS=$(echo "$RES" | sed 's/.*"ws":"\([^"]*\)".*/\1/')
WORLD_ID=$(echo "$RES" | sed 's/.*"world_id":"\([^"]*\)".*/\1/')

# 30s 会话：前 5s 只看状态流，然后全速前进 8s，再停
( echo 'connected'; sleep 5; \
  for i in $(seq 1 200); do \
    printf '{"type":"intent","duck_id":"duck-a","vx":0.3,"vy":0,"vyaw":0}\n'; sleep 0.04; done; \
  sleep 3 ) | timeout 30 wscat -c "ws://localhost:7800$WS" | tee /tmp/e2e-ws.log | head -3

# 断言：状态流里有 policy 从 stand 变 walk 的帧（鸭子真的在走）
grep -q '"policy":"walk"' /tmp/e2e-ws.log
echo "E2E PASS: 云鸭子被 wscat 驱动了 8 秒"
```

跑通标准：`E2E PASS` 打印，且 `docker ps` 期间能看到 `duckpet-w-*` 容器、结束后被回收。

- [ ] **Step 6: 提交** `git commit -m "worldd: session lifecycle + e2e — wscat drives a cloud duck"`

---

### Task 7: README 与运行手册

**Files:**
- Create: `~/repos/duckpet/README.md`

- [ ] **Step 1: 写 README**——四节：这是什么（引用 spec 路径与 §6a 多人预留的落实位置）、前置条件（microduck-sim 构建、duck-dev-build 镜像、RL 仓库源码）、三步运行（build image → cargo run → scripts/e2e.sh）、协议速查（上行/下行 JSON 各一例，注明关节序 = JOINT_NAMES、多人预留字段）。
- [ ] **Step 2: 提交** `git commit -m "duckpet: readme"`

---

## Self-Review 记录

- **Spec 覆盖**：M1 的三项交付（仿真单元产品化=Task 5、状态桥=Task 2+3、最小网关=Task 1+4+6）全部有任务；验收（wscat 驾驶）= Task 6 Step 5；§6a 的四项预留（agents 数组/duck_id/world 寻址/归属校验）落在 Task 3 与 Task 6。
- **类型一致性**：`RobotLink::state()` 返回 `watch::Receiver<RobotState>` 在 Task 2 定义、Task 3 消费；`WorldLauncher::start` 返回 `RobotdLink` 即 `RobotLink` 实现——Task 6 注册表以 `Arc<dyn RobotLink>` 持有。
- **占位符**：Task 2 Step 1 的第一段骨架标明"示意"并附完整 stub 代码，非留白；robotd.toml.in 的字段清单指向 duck-sim 的现成写法（微 duck-sim 脚本 159-168 行）。
- **已知风险**（留给实现者）：`worldd/Cargo.toml` 里 tokio-tungstenite 版本以编译时最新 0.2x 为准；镜像内 onnxruntime 的 pip 路径（dist-packages vs site-packages）两条都写了 fallback；`RobotState` 结构体字段以 duck-ipc-proto 当前定义为准（Default::default() 的字段以编译器报错为准逐个补全——`MoveState/SafetyState/LoopState` 均有 Default）。
