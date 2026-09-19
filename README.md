# 排练提词同步 API

基于 **Python + FastAPI + SQLite** 的排练提词同步服务：房间导入有序台词并设置
**推进人数** 与 **离线超时**；演员上报参与者 ID、单调递增事件序号和已到达的
台词位置；服务幂等处理重复或乱序上报，公共位置只推进到达到指定人数的最高台词
且 **绝不回退**；超时参与者自动退出推进人数计算。排练可 **暂停/恢复**
（基于时钟版本的乐观并发控制）：暂停期间冻结超时判定并拒绝位置上报，恢复时
扣除整段暂停时长，剩余在线时长原样延续。台本支持 **带版本号的增量修订**：
按当前行号执行 insert/replace/delete，整批原子生效，参与者与公共位置
按台词身份自动重排。

## 快速启动（单条命令）

```bash
docker compose up --build
```

服务地址：**http://localhost:8000**

- 交互式 API 文档（Swagger UI）：http://localhost:8000/docs
- 健康检查：http://localhost:8000/health

依赖安装随镜像构建完成，SQLite 建表随应用启动自动完成，无需任何手工初始化。

## 配置

| 环境变量   | 默认值                    | 说明                     |
| ---------- | ------------------------- | ------------------------ |
| `DB_PATH`  | `/data/teleprompter.db`（Compose 内） | SQLite 数据库文件路径 |

端口映射在 `docker-compose.yml` 中（默认 `8000:8000`）；数据库文件通过
`teleprompter-data` 卷持久化。

## 快速体验

```bash
# 1. 创建房间：推进人数 2，离线超时 30 秒
curl -X POST http://localhost:8000/rooms \
  -H 'Content-Type: application/json' \
  -d '{"id":"demo","name":"雷雨第一幕","advance_threshold":2,"offline_timeout_seconds":30}'

# 2. 导入有序台词（列表顺序即台词顺序，位置从 0 开始）
curl -X POST http://localhost:8000/rooms/demo/lines \
  -H 'Content-Type: application/json' \
  -d '{"lines":["周朴园：你姓什么？","鲁侍萍：我姓鲁。","周朴园：你是谁？"]}'

# 3. 演员上报：参与者 ID + 单调递增序号 + 已到达的台词位置
curl -X POST http://localhost:8000/rooms/demo/events \
  -H 'Content-Type: application/json' \
  -d '{"participant_id":"actor-zhou","seq":1,"position":2}'
curl -X POST http://localhost:8000/rooms/demo/events \
  -H 'Content-Type: application/json' \
  -d '{"participant_id":"actor-lu","seq":1,"position":2}'

# 4. 查询当前活跃成员、公共位置及推进原因
curl http://localhost:8000/rooms/demo/state

# 5. 台本增量修订：基于 revision 1 整批执行插入/替换/删除（原子生效）
curl -X PATCH http://localhost:8000/rooms/demo/lines \
  -H 'Content-Type: application/json' \
  -d '{"base_revision":1,"operations":[
        {"op":"insert","position":1,"text":"周朴园：（沉思）哦。"},
        {"op":"replace","position":2,"text":"周朴园：你到底是谁？"},
        {"op":"delete","position":0}
      ]}'
# → {"room_id":"demo","revision":2,"lines_count":3,"public_position":2,"participants_remapped":0}

# 6. 台词与状态都会携带当前 revision；以过期 base_revision 修订将返回 409
curl http://localhost:8000/rooms/demo/lines

# 7. 暂停排练（携带当前 clock_version）：冻结超时判定，期间位置上报返回 409
curl -X PUT http://localhost:8000/rooms/demo/clock \
  -H 'Content-Type: application/json' \
  -d '{"expected_clock_version":0,"paused":true}'

# 8. 恢复排练：扣除整段暂停时长，暂停前剩余在线时长原样延续
curl -X PUT http://localhost:8000/rooms/demo/clock \
  -H 'Content-Type: application/json' \
  -d '{"expected_clock_version":1,"paused":false}'
```

## API 一览

| 方法   | 路径                            | 说明                                       |
| ------ | ------------------------------- | ------------------------------------------ |
| POST   | `/rooms`                        | 创建房间（推进人数、离线超时）             |
| GET    | `/rooms/{room_id}`              | 房间配置（含当前台本版本 `revision`）      |
| POST   | `/rooms/{room_id}/lines`        | 导入有序台词（整体替换，重置进度并开启新版本） |
| GET    | `/rooms/{room_id}/lines`        | 查看台词（含当前 `revision`）              |
| PATCH  | `/rooms/{room_id}/lines`        | 台本增量修订（版本校验 + 整批原子生效）    |
| POST   | `/rooms/{room_id}/events`       | 演员上报位置（幂等；暂停期间 409） |
| GET    | `/rooms/{room_id}/state`        | 当前活跃成员、公共位置、最近一次推进原因（含暂停/时钟状态） |
| PUT    | `/rooms/{room_id}/clock`        | 暂停/恢复排练（`expected_clock_version` 乐观并发控制） |
| GET    | `/rooms/{room_id}/advancements` | 完整推进历史（含每次推进原因）             |
| GET    | `/health`                       | 健康检查                                   |

## 核心语义

- **幂等上报**：同一参与者的 `seq` 必须严格递增。重复（`seq` 相同）或乱序
  （`seq` 更小）的上报会被忽略，返回 `applied: false`，不产生任何状态变化，
  客户端可安全重试。
- **推进规则**：公共位置 = 活跃参与者中第 `advance_threshold` 高的位置，即
  “达到指定人数的最高台词”。活跃指在 `offline_timeout_seconds` 内有上报。
- **绝不回退**：公共位置只升不降。参与者超时离线、演员自己回报更低位置，
  都不会使公共位置下降。
- **离线超时**：超过超时未上报的参与者退出活跃成员列表与推进人数计算；
  重新上报后自动回到活跃集合。
- **排练暂停/恢复**：`PUT /rooms/{room_id}/clock` 提交 `expected_clock_version`
  与 `paused`：
  - 目标状态与当前一致时返回当前状态且 **不增版**（重复请求安全）；目标状态
    不同但版本不符返回 **409**、状态不变；成功切换后 `clock_version` 递增；
  - **暂停**：记录 `paused_at`，冻结活跃成员的超时判定（暂停再久也不会有人
    超时离线），并拒绝一切位置上报（返回 **409**，含重复上报）；
  - **恢复**：把整段暂停时长（`now - paused_at`）顺延到所有参与者的最后
    上报时间上，暂停前剩余的在线时长 **原样延续**；暂停前已超时离线的成员
    依然离线，**不会被重新激活**（只有本人重新上报才能回到活跃集合）；
  - `GET /state` 返回 `paused`、`clock_version`、`paused_at`（恢复后为
    `null`）；台本 `revision` 与时钟 `clock_version` 相互独立，暂停期间仍可
    修订台本，推进历史也不因暂停/恢复而改变。
- **推进原因**：每次推进记录促成的参与者名单与可读原因，可在
  `/state`（最近一次）与 `/advancements`（全部历史）中查询。
- **台本版本（revision）**：房间创建时为 0，每次整体导入或增量修订后递增；
  `GET /rooms`、`GET /rooms/{room_id}/lines` 与 `GET /rooms/{room_id}/state`
  均返回当前版本。
- **台词导入**：整体替换台本并重置房间进度（公共位置、参与者、推进历史），
  同时开启新版本（`revision` 递增）。
- **台本增量修订**：`PATCH /rooms/{room_id}/lines` 携带 `base_revision` 与一组
  按**当前行号**执行的 `insert` / `replace` / `delete` 操作，整批在事务中校验、
  原子生效：
  - `base_revision` 与当前版本不一致 → **409**，台本不变；
  - 任一操作越界（`insert` 允许 `position == 台词总数` 表示追加）或同一行上
    操作冲突（重复删除、重复替换、替换又删除）→ **422**，**全部不落库**；
  - 修订成功后 `revision` 递增，参与者与公共位置按原先指向的**台词身份**
    重排：保留（含被替换）的台词跟随新位置，被删除的回退到最近仍保留的
    前一行，无前一行则置为 `-1`；修订本身**不产生推进记录**。
- 时间字段均为 epoch 秒（浮点）；`state.server_time` 可用于对齐超时判断。

## 本地开发（不用 Docker）

```bash
pip install -r requirements-dev.txt
uvicorn app.main:app --reload          # http://localhost:8000
pytest                                 # 运行测试
```

## 项目结构

```
app/
  main.py       # FastAPI 路由与推进/幂等核心逻辑
  db.py         # SQLite 连接与启动时建表
  schemas.py    # 请求体校验
tests/test_api.py
Dockerfile
docker-compose.yml
requirements.txt / requirements-dev.txt
```
