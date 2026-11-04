# dsh-room 插件设计方案

> **一句话**：把 DSH 的 Agent 工作过程做成一个"胡闹厨房"式的等距小房间 —— 主角沿用 dsh-pet 的默认角色（Q 版蓝毛小女仆），她在这个房间里跑工位、敲键盘、结印分身、拿文件夹求签字、收工跑过来通知你，空闲时吃零食玩游戏。
>
> 文档状态：设计评审稿 **v0.3** · 不含实现 · 所有 API 均已在本机 profile 上用 `cordis_inspect_query` 实测（见附录 A），待实现期确认项见附录 B。
>
> **v0.2 → v0.3 变更**：
> ① 角色从"复用 dsh-pet 素材"改为 **自建同风格角色**（2D 骨架 + 挂点 + 序列帧）—— 成品视频无法与房间物件交互；
> ② 视角改为 **正俯视地板 + 侧视立绘**（星露谷 / 胡闹厨房语法），tile 网格、y 排序遮挡；
> ③ 新增 **附录 D · 美术素材清单**（规格、数量、优先级、工时、验收）；逐件清单另见
> [`dsh-room-美术素材清单.md`](dsh-room-美术素材清单.md)。v0.3.1 逐件核对后修正了三处小计：
> 角色层 45 → **47**、家具 34 → **33**、道具 23 → **22**（"托盘"曾被家具与道具重复计数），总数仍 **242**。
>
> **v0.1 → v0.2 变更**：新增第 9 章「房间是操作台」—— 把对话、中止、回答问题、确认审批四类交互落到了具体宿主 API 上，并定下"上行只有一条通道、实现只在 Host 半边"的边界；相应补了 phase `composing`、修饰位 `pendingAction`、配置项 `interaction.*`、风险 R10–R15。

---

## 0. 一页速览

| 维度 | 结论 |
|---|---|
| 插件名 / 包名 | `dsh-room`（与 `dsh-pet` 同构的 bundle：Host 半边 + Client 半边 + 素材路由） |
| 主渲染面 | ① `conversation.view` 注册的 **房间页**（等距全景，可看、可点、可深链）② `shell.overlay` 注册的 **迷你房间浮窗**（日常可见，约 260×180） |
| 状态源 | Host 半边订阅 `session/event` + `subagent/*` + `workflow/*` + `approval/*` + `api-session/*`，折叠成 **14 个房间状态 + 5 个修饰位**，再下发到 Client |
| 数据通道 | 房间页走 **Host projection + `wire.view`**（零轮询）；浮窗走 **Host 聚合器 + `ctx.webServer.register()` 长轮询**（dsh-pet 已验证的同一条路） |
| 角色 | **自建**「蓝毛小女仆」同风格角色：Q 版 3.5 头身、蓝发 + 女仆装，2D 骨架（47 层）+ 6 个挂点 + 26 个 clip。**不复制 dsh-pet 的任何素材**，只沿用同一套形象设定 |
| 场景 | **正俯视地板 + 侧视立绘**（星露谷 / 胡闹厨房语法）：64px tile 网格 16×10、单屏固定相机、按 `feetY` y 排序遮挡、10 个工位 + 传送带 + 门 + 庭院 |
| 关键取舍 | **角色不用成品视频**：骨架 + 挂点（socket）+ 接触帧事件，才让"抓/递/按/放"成立。20 来个 clip 覆盖 14 个状态，且道具是房间的一等对象可随时替换 |
| **交互定位** | 房间不只是看板，而是**操作台**：对话、中止、回答问题、确认审批都能在房间里发起。每个操作**只在 Host 半边实现一次**，Client 只做入口（见 §9） |
| 交互总线路由 | ①房间页优先用宿主白给的 `inputActions` / `useInput` / `session.pendingInteraction` ②浮窗与房间页共用一条**房间指令端点** `POST /dsh-room-<hash>/action` → Host 半边调 `sessionController.prompt\|cancel`、`userQuestions.answer`、`subagents.interrupt` |
| 审批的边界 | **提问/计划评审走公开的 `userQuestions.answer`（干净、无竞争）**；**审批默认只镜像 + 深链到官方卡**，房间内真签字是 opt-in 的 Host answerer（会与官方终局 answerer 竞争，见 §9.5 / R4） |
| 首发成本 | **P0 素材 8 人天**（附录 D.7.2）就能让交互跑通；期间角色用灰色剪影占位（R15），美术补齐只换图集 |
| 最大风险 | ①美术产能：自建角色 + 场景 242 件 / 23 人天（附录 D.7）②审批若在房间内认领，用户没看房间时会变成"谁都没法确认"的僵局 |

---

## 1. 目标与非目标

### 1.1 目标

1. **可读性**：不看对话流，只看房间就能知道"它现在在干什么、卡在哪、要不要我确认"。
2. **可解释性**：状态与真实事件一一对应，不是随机表演；点房间里的东西能跳到真实的 tool call / 审批卡。
3. **可操作**：房间里能直接**说话、喊停、回答、确认**，不用切回 Chat 才能推进流程。这是本插件区别于"纯装饰宠物"的核心价值。
4. **情绪价值**：忙的时候有节奏感（跑工位、蒸汽、传送带），完成时有回报（跑过来庆祝 + 通知卡），空闲时可爱（吃零食、玩游戏、擦桌子）。
5. **低耦合**：不改 DSH 核心，不改别人的插件，全部通过 slot + event + service 组合；**用户权限相关的操作只走宿主既有 API，不自己发明一套授权**。

### 1.2 非目标（v1 明确不做）

- **不做"抢审批"**：v1 的审批只在房间里展示与深链，真正的决策仍由官方审批卡完成；房间内真签是 M3 的 opt-in 开关（见 §9.5、R4）。
- 不做多会话同屏大字（v1 只渲染一个"焦点会话"，其余会话以门口小图标提示）；**所有交互指令都按 `sessionId` 校验，绝不误发到别的会话**。
- 不做 3D 引擎（Three.js/WebGL）。房间用 **Canvas 2D + 图集**实现（§5.3 / D12），UI 用 React，保证和 DSH 的主题、字体、性能预算一致。
- 不做房间编辑器的持久化（M3 视情况，改配置即生效）。
- 不做"和角色闲聊"（那是 dsh-pet 的地盘）；房间里发的每一句话都是**发给 Agent 的真实 prompt**，不是聊天玩具。
- 不做移动端全屏房间（窄屏退化为"角色卡 + 指令条"，见 §11）。

---

## 2. 核心设计决策（先给结论）

| # | 决策 | 理由 | 被否掉的选项 |
|---|---|---|---|
| D1 | 房间是 **Conversation 的一个 View**（`conversation.view`），不是独立页面 | 该 slot 是 session 作用域、owner 已渲染成 header 的 View tab（现有 chat/trajectory/context/fleet 四个占位证明第三方可进入），且 props 直接给 `sessionId` / `useConversation` / `useChat` / `useProjection` / `inspectCall` | ① 自建 `main` keyed 面板（需自己造导航，且拿不到 session 作用域）② Host 出 HTML + iframe（拿不到主题令牌与 locale，明令禁止） |
| D2 | 日常可见形态是 **`shell.overlay` 迷你房间**，房间页是"看大片"形态 | `conversation.view` 一次只渲染一个（进房间页就看不到 Chat），所以不能指望用户常驻；overlay 是 frame 级浮层，不抢焦点 | 只做房间页（用户不会常开）／只做浮窗（没有全景表现力） |
| D3 | 状态在 **Host 折叠**，Client 只做表现 | Client 侧全局事件只有 `connection/reset`、`locale/change`、`slots/changed`、`theme/change` 四个，**拿不到会话事件流**；Host 有 `session/event` 全量 | Client 自己 fold `useChat` 快照（拿不到 reasoning 是否在流式、拿不到审批原因、拿不到未落地子 agent） |
| D4 | 角色用 **2D 骨架 + 挂点（socket）+ 序列帧**，不用成品视频 | 成品视频没有可寻址的手部挂点、没有上/下朝向、不能参与 y 排序遮挡、也没有"接触帧"事件 —— 也就是**无法与房间物件交互**。骨架 + socket + clip 事件这三样才让"抓/递/按/放"成立 | 复用现成视频（dsh-pet 等）：省事，但角色只能"表演"不能"操作"，且 640×360 视频放进 2× 场景会糊 |
| D5 | 状态机采用 **phase + flags**，而不是笛卡尔积枚举 | 目标是续跑轮、多会话并发、上下文吃紧这类"修饰"不该另开状态 | 一状态一分支（必然是 30+ 分支的意大利面） |
| D6 | 分级降级：`prefers-reduced-motion` / 后台标签 / 窄屏 → 静态首帧 + 气泡 | 逐帧动画与粒子是唯一每帧在跑的负载，必须能整体关掉而信息不丢 | 只做开关（用户不知道为什么要关） |
| D7 | 交互"**实现一次在 Host，入口按渲染面选型**" | 房间页有宿主白给的 `inputActions` / `useInput` / `session.pendingInteraction`，浮窗没有；两条入口各写一套逻辑必然走样。所以操作语义只写在 Host 半边，页面能拿到 props 就先用 props，拿不到就走 `POST /action` | 两套实现（行为必然不一致）／只用 HTTP（页面明明有原生 API 却绕路） |
| D8 | 审批**不默认认领**，只镜像 + 深链 | 官方 `dsh-client-ui-approval` 是**终局 answerer**（"a deployment composes one terminal answerer"）；第三方再注册一个，要么被它挡掉、要么抢走它 —— 后者的后果是用户看不到官方卡、却也没意识到该回房间 | 用 `prepend` 抢审批（会造成"谁都没法确认"的僵局） |
| D9 | 每个交互带 `callId` + `seq` 做**陈旧校验**，Host 明确拒绝过期动作 | 房间点击到事件翻页之间有 100–300ms 竞态；审批点错是不可逆的副作用 | 乐观发送不校验（会确认一个已经消失的工具调用） |
| D10 | 房间与**当前会话**强绑定：切对话就换房间，浮窗只演当前会话，后台会话不得抢占 | `shell.overlay` 是 root 作用域、props 里没有 `sessionId`；`ctx.sessions` 的 `SessionTarget` 只有 `SessionId \| SubagentAddress`，**没有 current 哨兵**。不显式上报，就只能按"谁最后有事件"选 —— 后台子 agent 一跑，你看的就是别人的现场 | 按"最后活跃会话"自动选（会被后台任务抢）／让用户手选（多一次操作，忘了更糟） |
| D11 | 投影用 **正俯视地板 + 侧视立绘**（星露谷 / 胡闹厨房语法），**不用等距 isometric** | 正交网格的地板 + 立牌式家具 + 侧视角色，是"一屏厨房"读图效率的来源：家具只需画一个立面（等距要画两个面、成本翻倍），tile 寻路是标准四邻接，遮挡只靠一条 `feetY` 规则 | 等距 45°（成本翻倍、寻路变成六边形/菱形、和 DSH 的直角 UI 气质不搭）／真 3D（超出前端预算） |
| D12 | 地板/家具/角色/道具在 **一个 Canvas 2D 里单遍绘制**（含 y 排序），UI 仍是 React | 上百个 DOM 节点做逐帧位移与排序会掉帧；Canvas 一次 `drawImage` 每件，排序就是数组 sort。React 只负责气泡、面板、按钮这些"会变的 UI" | 全 SVG/DOM（当前骨架的做法，帧率与排序都吃力）／WebGL（收益不匹配复杂度） |

---

## 3. 系统架构

### 3.1 三个渲染面

```
┌───────────────────────────── DSH Web UI ─────────────────────────────┐
│  会话头部 View tabs:  Chat │ Trajectory │ Context │ Fleet │ 工作间 ←①  │
│  ┌──────────────────────────── 房间页（conversation.view）──────────┐ │
│  │  等距全景房间 + 角色 + 工位 + 传送带 + 气泡 + 详情侧栏              │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│                              ┌──────────────┐  ←② 迷你房间            │
│                              │ 260×180 浮窗 │     (shell.overlay)     │
│                              │  角色+1 行字 │                         │
│                              └──────────────┘                         │
│  ┌─────────────────────────┐  ←③ 通知卡（同属 overlay，瞬态）           │
│  │ ✅ 这一轮搞定啦，来看看  │                                          │
│  └─────────────────────────┘                                          │
└──────────────────────────────────────────────────────────────────────┘
```

- **①房间页**：`conversation.view`，`id: 'room'`，`order: 40`（排在 fleet 之后）。
- **②迷你房间**：`shell.overlay`，`id: 'room-mini'`，`order: 100`，可拖拽、可收起、位置/大小持久化在 `localStorage`（不写 session 日志）。
- **③通知卡**：同一个 overlay 内由状态机驱动的瞬态卡（3–5s 自动消失），点击 → 切到房间页并聚焦对应事件。

### 3.2 两条下行通道 + 一条上行指令通道

```
                    ┌──────────────── Host 半边 ─────────────────┐
session/event ──────┤                                            │
  turn / step / tool│   fold(state, event)   ← 纯函数、同一份代码  │
  approval / todo   │        │                                   │
                    │        ├──────────► projection "dsh-room"    │──► wire.view ──► 房间页
subagent/start|end ─┤        │              (+ wire.view)         │    (useProjection)
workflow/*         ─┤        │                                   │
                    │        └──────────► 全局聚合器（焦点会话）   │──► webServer
api-session/status ─┤                          │                 │    /state?since=&timeout
agent/status       ─┘                          │                 │         │
                                               └─────────────────┘         ▼
                                                                   迷你房间（长轮询）
   webServer.register('/dsh-room-<hash>/atlas/*') ──► char/room/props/fx 图集 ──► 两个渲染面

   房间页 / 浮窗 ── POST /dsh-room-<hash>/action ──► Host 半边 ──► sessionController.prompt|cancel
   { op, sessionId, callId, seq }                                    userQuestions.answer
                                                                    subagents.interrupt
                                                                    approval answerer（opt-in）
```

> 图的读法：**上半部分只有一份 fold**，两条下行通道都从它派生，所以"房间页显示的状态"和"浮窗显示的状态"不可能不一致；**下面那条是唯一的上行口**，房间里任何操作都从这里出去。

**为什么下行两条、上行一条**：

- `wire.view` 是 session 作用域的（跟着 session 订阅、断线重连自动重放），房间页天生适配，且"框架驱动、插件计算"符合实践规范。
- 迷你房间是 root 作用域、要跨会话知道"现在哪个会话在忙"，作用域不匹配；而 Host 侧 `webServer.register()` + 长轮询是 dsh-pet 已经在这套 Web 构建里跑通的路径（`/state?since=&timeout=` 的老宿主兼容处理也一并继承）。
- 两条下行共用**同一个 fold 函数**（同一个 `apply`），不存在两套语义。
- 上行**只有一条**：Client 永远不直接改宿主状态，只投递"意图"（intent），由 Host 半边校验（会话是否同一个、`callId`/`seq` 是否过期、agent 是否存活）后再转成宿主 API 调用。这样"页面能做什么"与"浮窗能做什么"天然一致，权限也只有一处可审。

### 3.3 包结构与清单

```
dsh-room/
├─ package.json          # dsh.bundle.patch + dsh.client{platform:'web', inject:['@deepseek-ai/dsh-client-connection']}
├─ cordis.patch.yml      # insert: [{ id: dsh-room, name: dsh-room, config: {...} }]
├─ index.js              # Host：apply(ctx, config)
├─ client.js             # Client：window.__ModuleLoader__.load({id:'dsh-room', factory(require){...}})
├─ lib/
│  ├─ room/fold.js       # 状态折叠（纯函数，可单测）
│  ├─ room/selectors.js  # 工具→工位、phase→动作/道具的映射表
│  ├─ room/actions.js    # 交互意图 → 宿主 API（prompt / cancel / answer / approve / interrupt）
│  └─ room/routes.js     # /state 长轮询 + /action 指令 + /assets 素材，三个路由一个 handler 表
├─ assets/
│  ├─ atlas/{char,room,props,fx}@2x.png + *.atlas.json   # 自建图集（附录 D 交付物）
│  ├─ rig/character.json                                 # 骨架：parts / pivot / socket / clip / event
│  ├─ tilemap.json                                       # 16×10 网格与家具占格（§5.2）
│  └─ icon.svg
├─ locale/{en,zh}.json   # meta.title / meta.description + 全部 UI 文案
└─ README.md
```

- Host 入口 `export function apply(ctx, config)`，`export const inject = ['webServer']`（可选 service 放 `inject` 保证别的 profile 里不炸）。
- Client 入口是**惰性工厂**，`id` 必须等于包名；所有注册（slot / timer / listener / style）在 `apply` 里用 `ctx.effect` / `ctx.on` 建立并返回清理函数；工厂体内禁止副作用。
- 素材通过 Host 路由提供（`/dsh-room-<hash>/atlas/...`），不内联 base64（图集内联会把 boot payload 撑爆）。

### 3.4 状态折叠的位置

`lib/room/fold.js` 导出唯一的纯函数：

```js
// 纯函数、同步、无副作用：不感兴趣的事件返回同一个 state 引用（下游零成本）
export function applyRoomState(state, event, ctxMeta) { /* ... */ }
```

同一份 `applyRoomState` 被四处复用：Host projection、Host 全局聚合、单元测试（把真实 session 事件序列喂进去断言 phase 序列）、回放模式（M4）。

**这条边界是硬的**：`fold` 只读事件、只写状态；`actions` 只写宿主、不碰状态。**状态的唯一真相永远是 session 日志** —— 房间里点了"中止"，UI 不许自己把 phase 改成 `idle`，而要等 `turn/end{aborted}` 到来自行变化；中间这 300–800ms 用 `aborting` 相表达"已发出、等落地"（见 §4.6）。同理，点了"发送"不预置 `thinking`，而是等 `user/message` / `turn/start` 真的落地。这样做的好处是：网络断了、宿主拒了、动作为过期被丢弃，界面都不会撒谎。

---

## 4. DSH 执行任务的状态全景（核心章节）

### 4.1 状态来源分三层

| 层 | 来源 | 回答的问题 | 粒度 |
|---|---|---|---|
| L0 生命周期 | `api-session/*`、`agent/status`、`session/created/disposed` | 这个会话活着吗、在跑吗、几个在跑 | session |
| L1 回合与步骤 | `turn/*`、`step/*`、`assistant/message`、`tool/call`、`tool/result`、`assistant/attempt`、`request/*` | 这一轮走到哪一步了、模型在产出什么 | turn/step/call |
| L2 特殊子流程 | `approval/asked`、`ask_user_question`、`subagent/*`、`workflow/*`、`todo/write`、goal 工具、compaction | 是不是在等用户、是不是在分身、是不是在压缩 | 事件 |
| 交互层 | `user/message` 入队、`agent/inbox/*` | 用户刚说话 / 排队中 | turn |

### 4.2 L0 · 会话生命周期

| 事件 | 语义 | 可观察点 | 房间含义 |
|---|---|---|---|
| `session/created` / `session/disposed` | 会话进入/离开 live store | Host | 房间"有人入住房东"／"退租关灯" |
| `api-session/added` / `removed` | 会话出现在列表／离开注册表 | Host | 门口信标：+1 / -1 |
| `api-session/status(sessionId, running)` | Agent 运行态变化 | Host | 全局"哪个房间亮着灯" |
| `agent/status {idle ⇄ running}` | Agent 状态机 | Host | 同上，更及时 |
| `agent/error {turn, step, error}` | 落在回合位置的失败 | Host | 红灯 + 冒汗 |
| `api-session/error(sessionId, message)` | 不在回合位置的失败 | Host | 门口挂"故障中"牌 |
| `agent/disposed` | Agent 离开注册表 | Host | 角色下班走人 |

> **注意**：`whenIdle()` ≠ "这一轮完了"（多个输入可能共享一个 running 区间），所以**不要用 idle 判断成功**，成功只认 `turn/end` 的 `reason.kind === 'completed'`。这条已经写进实践规范，也直接决定房间的"庆祝"必须挂在 `turn/end` 上。

### 4.3 L1 · 回合、步骤与流式产出

`SessionEventMap` 实测形状（节选，字段名为实际字段）：

```ts
'turn/start'   : { turn: number }
'turn/end'     : { turn: number; reason: TurnEndReason }
'step/start'   : { turn: number; step: number }
'step/end'     : { turn: number; step: number }
'user/message' : UserMessage                            // 含 source.kind（'user' | 'goal' | …）
'assistant/message': { turn; step; message; stream: AssistantStreamRecord[]; usage?; interrupted?: true }
'assistant/attempt': { turn; step; stream }             // 失败/重试/取消的一击，不进历史
'tool/call'    : { turn; step; callId; name; arguments }
'tool/result'  : { turn; step; message; error?: { name; code; reason? }; meta? }
'request/header': { header; reason: 'initial'|'resume'|'change'|'series'; startsSeries? }
'request/context': RequestContext
```

`TurnEndReason` 的 7 种 kind（**这是"结束"的全部语义，房间必须逐一区分**）：

| kind | 含义 | 房间反应 |
|---|---|---|
| `completed` | 正常答完 | 成功（但要先判是否目标续跑中间轮，见 §4.6） |
| `max-tokens` | 触顶截断 | 失败系：抱着头"话没说完" |
| `error { error: LlmFailure }` | 模型/请求失败 | 失败系：亮红灯 + 冒汗，气泡带 `error.message` |
| `blocked` | 被阻塞（等用户/等外部） | **归入等待**，不是失败 |
| `aborted { reason: user\|parent\|hook\|disposed\|legacy }` | 被打断 | **必须清理回 idle**，绝不残留"忙碌"（这是 dsh-pet 当年"这一步正在进行中哦"永久挂死的根因） |
| `interrupted` | 中断态 | 同 aborted |
| `forked` | 因 fork 收口 | 静默清理，新房间继承 |

流式产出（决定"思考"与"打字"要分开）：

```ts
AssistantStreamRecord =
  | { type:'text-chunks';      time0; index; dt[]; texts[] }
  | { type:'reasoning-chunks'; time0; index; dt[]; texts[] }   // ← 思考/推理在流
  | { type:'tool-call-chunks'; time0; index; dt[]; id; name?; args[] }  // ← 参数生成中，尚未调用
  | { type:'chunk'; time; chunk: StreamChunk }
```

- `reasoning-chunks` 有增量 → **思考相**（托腮、冒泡、白板乱画）
- `text-chunks` 有增量 → **表达相**（打字/说话）
- `tool-call-chunks` 有增量 → **蓄力相**（伸手去拿工具，还没真正开工）
- 实时帧从 `agent/assistant-stream`（`{ agent, frame }`）拿，**不要轮询 `agent/status`**；落地节点等 `assistant/message`。
- `FinishReason` 也是 5 种（`stop` / `tool-calls` / `max-tokens` / `aborted{failure}` / `error{failure}`），用于 attempt 层判定。

### 4.4 L2 · 特殊子流程

| 事件 / 工具 | 形状 | 房间含义 |
|---|---|---|
| `approval/asked` | `{ toolName, reason? }` | **拿文件夹请签字**（核心演出） |
| `approval/request`（host 服务瀑布） | `ApprovalRequestEvent { agent, toolName, callId?, reason?, displayReason?, signal? }`，返回 `'allowed-once'\|'rejected'\|'cancelled'\|'unavailable'` | M3 房间内签字的接入点（必须 opt-in，且会与官方终局 answerer 竞争，见 §9.5） |
| `tool/call` 且 `name === 'ask_user_question'` | `arguments` 里含 `questions[]` | **举手问问题**（不是普通工作）；房间里可用 `userQuestions.answer` 直接答复（§9.3 ③） |
| `subagent/start` | `SubagentRunInfo { runId, provider, id, local }` | **结印分身术**开始（父作用域只收到自己派出的，天然隔离） |
| `subagent/end` | `SubagentRunEndInfo` | 分身归位（成功/失败两版落地动作） |
| `workflow/start` / `workflow/phase` / `workflow/log` / `workflow/agent-start` / `workflow/agent-end` / `workflow/end` | `WorkflowRunInfo` + 各自 info | **流水线厨房**：传送带加速、多口灶同时起烟、幕布切场景（phase） |
| `todo/write` | `{ todos: Array<{ status, content }> }` | **白板贴便签**；取「最后一个 in_progress，否则第一个 pending」作为当前步骤（dsh-pet 踩过的坑） |
| `update_goal` / `get_goal` / `create_goal`（工具） | `arguments.action ∈ {complete, blocked, …}` | 目标收尾轮判定：**本轮 completed ≠ 整个任务完成** |
| compaction（`compaction/*`） | 压缩摘要流 | **收拾桌面 / 把纸箱封箱** |
| `request/header.reason === 'series'` / `change` | 请求头变更 | 换工具/换模型 → 角色"换一套装备" |
| `job_*`（工具） | 后台进程 | 角落慢炖锅冒气，`job_output` 才开盖 |
| `schedule_*`（工具） | 定时提醒 | 门口日历/闹钟抖一下 |
| `present`（工具） | 交付文件 | 传送带送出一份"成品外卖"到门口 |

### 4.5 工具 → 工位映射（"厨房"的灶台就是工位）

房间的"胡闹厨房感"来自 **一个工具 = 一个工位**：角色为了干活必须走过去，这就把抽象的 tool call 变成了可见的空间移动。

| 工具类别（实测可用的工具名全景见附录 A.3） | 工位 | 动作 |
|---|---|---|
| `read` `glob` `grep` `read_image` `session_search` `session_event_*` `lsp` | 书架 / 文件柜 | 抽书、翻页、把书塞回去 |
| `write` `edit` | 大工作台 + 显示器 | 敲键盘、写字、删改（有"Ctrl+Z 撤销"小动作） |
| `pwsh` `bash` `terminal_*` | 终端灶台（带烟囱） | 起锅、爆炒（火花）、糊了（报错黑烟） |
| `web_search` `web_fetch` | 窗口 + 望远镜 | 探头看窗外、记笔记 |
| `subagent` `subagent_fork` `spawn_teammate` `send_message` `wait_agent` `interrupt_agent` `list_agents` | 庭院空地（玻璃门外） | **结印 → 分出一模一样的小人 → 各自跑开 → 归位** |
| `workflow` | 流水线指挥台 | 拉杆、看仪表、传送带加速 |
| `ask_user_question` `approval` | 柜台（带铃铛与签字笔） | **抱文件夹、按铃、等签字** |
| `todo_write` | 白板 | 贴便签、连线、打钩 |
| `create_goal` `update_goal` `get_goal` | 目标大屏 | 抬头看大屏、按确认键 |
| `skill` | 技能柜 / 换装镜 | 换装、翻秘籍 |
| `job_*` | 慢炖锅 / 烤箱 | 塞进去、看火、开盖 |
| `schedule_*` | 门口日历 + 闹钟 | 写一笔、拨指针 |
| `present` | 传送带 + 门 | 放上托盘、按下启动 |
| `cordis_*` `plugin_manager` | 插件工坊（角落小桌） | 拧螺丝、焊锡 |
| MCP 工具（`mcp__*`） | 外卖窗口 | 递单子、接外卖 |
| 未知工具 | 房间中央 | 通用"干活"动作（兜底，绝不空转） |

### 4.6 归一化：14 个房间状态 + 5 个修饰位

**状态（phase，互斥，枚举）**

| # | phase | 中文 | 触发（权威条件） | 退出条件 |
|---|---|---|---|---|
| 0 | `offline` | 空房 | 无会话 / 连接未建立 | 有会话 |
| 1 | `idle` | 待机 | 无 running、无未闭合 turn | 任一事件进入其他相 |
| 2 | `attending` | 起身接单 | `user/message`（source.kind==='user'）落地、`turn/start` 之前 | 收到 `turn/start`（→thinking） |
| 3 | `thinking` | 思考 | `turn/start`、`step/start`、reasoning 流式增量、`request/header` | 出现 `tool/call` 或 `text-chunks` |
| 4 | `planning` | 排计划 | `todo_write` 落地、`pre-step` 决策为 plan | 出现下一次 `tool/call` |
| 5 | `working` | 干活 | `tool/call`（非 ask_user_question / 非审批类） | `tool/result`（→result） |
| 6 | `delegating` | 分身 | `subagent/start`、`workflow/start` | `subagent/end` 全部归位、`workflow/end` |
| 7 | `waitingApproval` | 求签字 | `approval/asked`、`turn/end.blocked` | 审批结果（`approval/settled` 或下一事件） |
| 8 | `waitingAnswer` | 等你回答 | `tool/call` 且 `name==='ask_user_question'` | 该 tool/result 落地 |
| 9 | `compacting` | 收拾压缩 | compaction 流程事件 | 压缩结束 |
| 10 | `result` | 清点归档 | `tool/result` 落地（0.6–1.4s 过渡态） | 下一次 `tool/call` 或 `turn/end` |
| 11 | `succeeded` | 收工庆祝 | `turn/end.completed` **且** 非目标续跑中间轮 | 用户下一句 / 4s 后回落 idle |
| 12 | `failed` | 出事 | `turn/end` ∈ {error, max-tokens, interrupted}、`agent/error` | 用户下一句 / 8s 后回落 idle |
| 13 | `composing` | 你说，我听着 | **用户侧事件**：房间输入框获得焦点且有内容（不是宿主事件） | 提交成功、失焦清空、或 Agent 又开始干活 |

**修饰位（flags，正交，不新增状态）**

| flag | 来源 | 表现影响 |
|---|---|---|
| `goalRound` | `user/message.source.kind === 'goal'` | 中间轮 `completed` → 落到 `result` 而非 `succeeded`；角色"看表继续干" |
| `closing` | `update_goal.action` | `blocked` → 失败相但文案是"卡住了"；`complete` → 允许庆祝 |
| `contextPressure` | `tokenMeter` / usage 累积 | 0/1/2 三档：桌面纸张变多 → 房间开始堆箱子 → 大幅压缩前预演"收拾" |
| `multiBusy` | 其他会话 `api-session/status` | 门口多一块门牌，迷你房间右侧显示"另外 2 个房间在忙" |
| `pendingAction` | 房间刚提交了某个指令（中止 / 发送 / 回答 / 确认），宿主还没回执、事件还没落地 | 0.6–0.8s 的"收到，我这就办"演出**叠加**在当前动作上（不换脚、不换工位）；回执失败则立刻换成"没办成"并给红字原因。**它不是状态**，因为底层 phase 不许被 UI 乐观改写（见 §3.4） |

### 4.7 状态机规则（从 dsh-pet 的真实坑里提炼）

1. **优先级**（并发状态收敛到一个展示）：`waitingApproval`(60) > `waitingAnswer`(58) > `failed`(50) > `delegating`(45) > `working`(40) > `planning`(35) > `thinking`(30) > `result`(25) > `succeeded`(20) > `attending`(15) > `composing`(14) > `idle`(10) > `offline`(0)。同优先级取 `seq` 更大者。
   > 原文依据：dsh-pet 用 `waiting > error > working > thinking > result > success`，其中"result 高于 success"是为了**防止别的会话的已完成态压过本会话的进行中态**，即"中途庆祝"。这条必须继承。
2. **同相不重播**：相同 phase 且 `seq` 不前进 → 不产生新状态、不重启动画（`apply` 返回同一引用）。
3. **去抖（滞回）**：工具秒回时会出现 `working → result → working` 的高频抖动。规则：`result` 相最短驻留 600ms；若在这 600ms 内又来了 `tool/call`，**不切 `result`**，只在当前工位叠加一个 0.4s 的"清点"小动作。（dsh-pet 的"忙碌档抖动抑制"同源问题。）
4. **aborted 必清**：`turn/end` 为 `aborted`/`interrupted` 时**立即**落回 `idle`，并写明原因气泡（"被打断了"）。任何"等某个 result 才复位"的写法都会造成永久卡死。
5. **超时漂移**：处于 `working`/`thinking` 且 8s 无事件 → 进入"漂移"变体（看表、张望、抖腿）；>60s → "长任务"变体（趴在工位、慢炖锅冒气）。**不改变 phase**，只改变动作选择（避免状态爆炸）。
6. **多会话聚合**：房间只演一个会话（焦点 = 迷你房间所在/最近活跃 running 会话）。其它会话只在门口以门牌指示灯反映，避免"两个角色抢一个工位"的混乱。
7. **终态回落**：`succeeded`/`failed` 由定时器在 4s/8s 后回落 `idle`；定时器必须挂在 owning effect 上，插件卸载即清。
8. **交互不推状态**：房间发出任何指令都只置 `pendingAction`，**不直接改 phase**；只有真实回执与后续事件能推进状态机。最坏情况是 `pendingAction` 超时淡出、phase 保持原样 —— 界面永远不会"因为点了按钮就假装任务变了"。
9. **等待相不可被自身动作打断**：`waitingApproval`/`waitingAnswer` 期间用户点"回答/确认"只更新 `pendingAction`，不要切到 `working` 假装已经继续；`tool/result` 落地才算走完。

### 4.8 状态 → 动作 → 场景 总表（设计核心交付）

| phase | 工位/位置 | 动作（clip，见 §6.4） | 道具（程序化） | 场景变化 | 气泡/通知 |
|---|---|---|---|---|---|
| `offline` | 房间中央趴着 | 深度睡眠/趴睡（静态帧即可） | — | 关主灯、窗外夜色、显示器熄屏 | 无 |
| `idle` | 沙发 / 游戏机 / 桌边 | 待机呼吸、偷吃零食、玩魔方、哼歌、擦桌（随机链） | 零食袋、手柄、抹布 | 暖光、窗景白天、水壶冒小气 | 偶发一句碎碎念（可关） |
| `attending` | 门口 → 工作台 | 起身小跑（左右镜像） | — | 顶灯由暖转白（一格） | 头顶"！" |
| `thinking` | 座椅（托腮） | `think_chin`（`goalRound` 时半段 `work_look`） | 头顶气泡、飘起的纸 | 显示器滚动灰色代码、主灯偏冷蓝、镜头轻微推近 2% | "正在想下一步…" |
| `planning` | 白板 | 贴便签 / 画线 | 便签（数量 = todo 条数）、马克笔 | 白板灯亮、便签随风轻摆 | 当前 todo 文本（截断 40 字） |
| `working` | 该工具的工位 | 按工位映射（敲键盘/翻书/爆炒/望远镜…） | 工位道具高亮、火花/蒸汽粒子 | 工位上方射灯亮、传送带停 | "正在 <工具名> …" |
| `delegating` | 庭院空地 | **结印 → 分身影 → 归位**（见 §6.5） | 影分身烟雾、每个小人头顶编号 | 庭院灯亮、玻璃门推开、镜头拉宽 4% | "派出 N 个子 agent" |
| `waitingApproval` | 柜台 | 抱着文件夹小跑 + 按铃 + 递纸 + 原地等 | **文件夹/纸 + 签字笔 + 铃铛**（手部锚点叠加） | 柜台射灯闪烁、铃铛摇、主灯降饱和 | 通知卡："这里要你确认一下" + `toolName`/`reason` |
| `waitingAnswer` | 柜台（举手） | 举手 + 问号气泡 + 踮脚 | 问号、选项纸条 | 同上但灯不闪（区别于审批） | 通知卡："有个问题要你选" |
| `compacting` | 工作台 | 搬纸箱 / 封箱 / 抖桌布 | 纸箱堆叠（高度 = 压缩比例） | 灯光变"收工感"、灰尘粒子 | "整理一下记忆…" |
| `result` | 工位 → 归档柜 | `work_read`（0.6–1.4s） | 文件夹飞进柜子 | 传送带轻转一格 | 无（避免噪声） |
| `succeeded` | 工位 → **镜头前** | `run_front` → `cheer` | 完成印章、"✓" | 主灯暖金、彩带粒子、窗外放烟花（M2） | **通知卡 + 系统通知（可选）**："这一轮搞定啦" |
| `failed` | 工位原地 | `sad` / `panic` | 报警灯、冒烟的小锅 | 主灯转红、闪烁 2 次、显示器红屏 | 通知卡：`error.message` 截断 / "输出被截断了" / "被打断了" |
| `composing` | 柜台前（面向镜头） | 抬头听 + 点头 | 话筒 / 便签 | 柜台柔光、其余不变 | 房间内输入框浮在角色旁边（见 §9.3） |

> 其中 `waitingApproval` / `waitingAnswer` / `composing` 三行是**可操作行**：它们不只是表演，还带真实控件与真实提交路径（§9）。其余行是纯表现。

---

## 5. 房间场景设计（星露谷 × 胡闹厨房）

### 5.1 风格与镜头

- **投影**：**正俯视地板 + 侧视立绘**（星露谷 / 胡闹厨房共用的那套语法，**不是 45° 等距**）：
  - 地板是纯正交俯视的方格，**没有透视收敛线**；
  - 家具/物件是"立牌"——正面立面 + 一条很薄的可见顶面（2.5D 感），不画侧面；
  - 角色是侧视立绘，脚底锚在某个 tile 的地面点上，身高约 1.75 格；
  - 这样一眼能读懂"谁在哪、谁挡在谁前面"，而且每件家具只需画一套立面（等距要画两面）。
- **相机**：**一屏固定**（胡闹厨房）：16:9、永不滚动、不缩放、不旋转；窗口尺寸变化时整屏等比 `scale`，不重排布局。
- **网格**：基准 tile = **64×64 px（@1x）**，画布 **16×10 格 = 1024×640**；家具、道具、站位、寻路全部落在 tile 上（见 §5.2）。
- **遮挡**：唯一规则 —— 按 **`feetY`（脚底 / 底边 y）** 排序；地板贴花（阴影、地毯、水渍）不参与排序。这就是"角色走到柜台后面被挡住"的全部实现，也是唯一需要遵守的图层纪律。
- **画风**：手绘平涂 + 2px 深色描边（**不做像素风**：DSH 的 UI 是高分屏矢量，像素风会显脏），2× 导出以适配高分屏；只一层椭圆软阴影，不用渐变。
- **配色**：房间自带色板（奶油墙 `#F7E9D7`、木地板 `#E0B98A`、青绿瓷砖 `#8FD3C1`、砖红 `#D9544B`、暖灯 `#FFD79A`），**但外壳（面板、按钮、气泡边框）必须用主题令牌**：`--dsw-alias-bg-layer-1`、`--dsw-alias-border-l1`、`--dsw-alias-label-primary`、`--dsw-alias-brand-primary`、状态色用 `--dsw-alias-state-{success,warn,error,idle}-primary`。这样浅色/深色主题下房间外的 UI 自动跟随，房间内部像"贴纸"一样自成一体。
- **尺寸**：房间页按 16:9 铺满可用宽度，内部坐标一律用 **tile**（`x * 64, y * 64`），不依赖相对单位；缩放只是一次画布 `scale`，布局永不变形。迷你房间是同一张画布的**截图式缩小**（整体 scale），不是另一套美术。

### 5.2 平面布局（tile 网格 16×10）

图例：`#` 墙 / 边界 · `.` 可走地板 · `S` 书架(2×4) · `D` 工作台(3×2) ·
`T` 灶台(2×3) · `C` 审批柜台(3×3) · `B` 白板(墙上 3×1) · `N` 窗(墙上 2×1) ·
`M` 目标大屏(墙上 2×1) · `V` 传送带(5×1) · `O` 沙发(2×2) · `X` 门(1×2) · `Y` 庭院空地

```
       0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
row 0  # # # N N # # # B B B  #  M  M  #  #
row 1  S S . . . . . . . .  .  .  .  .  .  #
row 2  S S . . . . . . . .  .  C  C  C  .  #
row 3  S S . . D D D . . .  .  C  C  C  .  #
row 4  S S . . D D D . . .  .  C  C  C  .  X
row 5  . . . . . . . . . .  .  .  .  .  .  X
row 6  T T . . . V V V V V  .  .  .  O  O  #
row 7  T T . . . . . . . .  .  .  .  O  O  #
row 8  T T . . . . . . . .  .  .  .  .  .  #
row 9  . . . Y Y Y Y Y Y Y  Y  Y  Y  #  #  #
```

工位表（**唯一锚点 = 角色站位**；家具占的格子不可走，所以锚点永远在相邻地板上）：

| station | 家具占格 (col,row,w,h) | 角色锚点 (col,row) | 朝向 | 挂点交互 | 联动道具 |
|---|---|---|---|---|---|
| `shelf` 书架 | (0,1,2,4) | (2,3) | `left` | handR 抽书 / 翻页 | 书、梯子 |
| `desk` 工作台 | (4,3,3,2) | (5,5) | `up` | handR 键盘、handL 纸 | 显示器、键盘、椅子 |
| `stove` 灶台 | (0,6,2,3) | (2,7) | `left` | handR 锅铲 | 锅、烟囱、灶左端的慢炖锅 |
| `window` 窗口 | (3,0,2,1) 墙上 | (4,2) | `up` | 视线（望远镜走 handR） | 望远镜、窗帘 |
| `board` 白板 | (8,0,3,1) 墙上 | (9,1) | `up` | handR 马克笔 + 便签 | 便签、马克笔 |
| `counter` 审批柜台 | (11,2,3,3) | (12,5) | `up` | handR 纸/文件夹、handL 铃 | 文件夹、铃、签字笔 |
| `conveyor` 传送带 | (5,6,5,1) | (7,7) | `up` | 双手托盘 | 托盘、成品盒 |
| `sofa` 沙发 | (13,6,2,2) | (12,7) | `right` | — | 零食、手柄、毯子 |
| `door` 门 | (15,4,1,2) | (14,4) | `right` | — | 门牌、门铃 |
| `yard` 庭院 | (3,9,10,1) | (7,8) | `down` | — | 影分身烟、木桩 |

**寻路（标准网格 A\*，不再用节点图）**：16×10 网格，被家具/墙占住的格子不可走，四邻接、不切角；
速度 3.2 格/秒（≈205 px/s）。到达锚点后按上表转向，再播对应 clip。**斜向移动不允许** ——
这既是星露谷的手感，也让"角色永远站在格子中心"这条对齐规则成立（否则脚底锚点在缩放时会漂）。

**为什么锚点唯一**：一个工位只允许从**一个**格子操作，避免"操作时角色站在哪"这类歧义；需要多格操作时
（例如柜台两侧），就用两个工位 id 表达，而不是给一个工位配一组锚点。

**交互距离**：站在锚点上即可操作（1 格内），不做"必须贴身"的额外判定 —— 房间只有 10 个工位，
玩家不会误解"我站这儿能不能按到"。

### 5.3 绘制顺序（一次 Canvas 2D 扫描，不是 DOM 层级）

每帧画三批，React 只画第 4 批：

```
① 地板层（不参与排序）：地板 tile → 地毯/水渍 → 每个可站立物件的椭圆软阴影
② 排序层：把 [家具, 角色, 手持道具, 分身] 合成一个数组，按 feetY 升序排序后逐个 drawImage
   · 家具 feetY = 底边 y；角色 feetY = 脚底 y；手持道具 = 持有者 feetY + 1px（保证压在手上）
   · 站在柜台下方 → 角色 y 更大 → 后画 → 压在柜台前面。这就是全部遮挡逻辑。
③ 覆盖层：蒸汽/火花/彩带/分身烟 → 光照（multiply 一层暖/冷/红闪）→ 暗角
④ React 层：气泡、工位标签、通知卡、问题板、输入框、按钮
   （这些要可聚焦、可选中文字、要吃主题令牌，必须留在 DOM）
```

**为什么这样切**：② 是唯一每帧变化的层，放进一次 Canvas 扫描既拿到 y 排序，也不触发 React 重渲染；
④ 变化低频，交给 React 才能保留键盘可达性与 `--dsw-alias-*` 令牌一致性。

### 5.4 灯光 / 时间 / 天气 = 任务节奏

把"抽象进度"翻译成环境：

| 输入 | 环境表现 |
|---|---|
| 刚开新一轮 | 顶灯从暖到白（0.6s），窗外天色亮一档 |
| 连续 3 次以上 tool call | 传送带开始转、灶台火变大、蒸汽变密（"厨房忙起来了"） |
| 长任务（>60s 无事件） | 天色转黄昏、灯变暖、角色影子拉长 |
| 上下文吃紧（contextPressure 2） | 桌子上的纸越堆越多，最后堆到地上 |
| 出错 | 主灯红闪 2 次（0.15s/次），灶台冒黑烟 |
| 成功 | 暖金主灯 + 彩带 + 窗外烟花（M2） |
| 空闲 >5min | 天色转夜、主灯调暗到 40%、角色开始打瞌睡 |

### 5.5 场景状态变化表（可玩物件的"状态联动"清单）

每个物件都必须**至少响应一个状态**，否则删掉（避免装饰噪声）：

| 物件 | 响应 | 具体变化 |
|---|---|---|
| 显示器 | `thinking` / `failed` / `succeeded` | 灰代码滚动 / 红屏错误 / 绿色对勾 + 烟花 |
| 目标大屏 | `goalRound` / `closing` | 显示 `turn 3/∞` / 收官时打勾 |
| 白板便签 | `planning` + todo 数 | 便签数量与当前 in_progress 打钩 |
| 灶台火 | `working`(pwsh) | 小→中→大，报错变黑烟 |
| 传送带 | `present` / `result` | 转动 + 托盘送出 |
| 铃铛 | `waitingApproval` | 摇摆 + 蓝色高亮环 |
| 沙发靠垫 | `idle` | 被压下去（角色坐下时） |
| 慢炖锅 | `job_*` | 锅盖轻跳、白气 |
| 门口日历 | `schedule_*` | 翻一页 |
| 门牌 | `multiBusy` | 多块小灯牌亮 |
| **写字台输入框** | `composing` / `pendingAction` | 聚焦时柔光亮起；提交时纸条飞进传送带（§9.3 ①） |
| **急停拉杆** | 仅在 `working`/`thinking`/`delegating` 可用 | 可用时红光微亮；拉下时全屋灯闪一下（§9.3 ②） |
| **选择板（提问）** | `waitingAnswer` | 举起、选项按钮可点、右上角倒计时环（§9.3 ③） |
| **待发篮（队列）** | `queue.length` | 纸条数量 = 排队条数，可抓回编辑/删（§9.1 附加项） |

---

## 6. 虚拟角色设计

### 6.1 自建角色：为什么成品视频不行

沿用 dsh-pet 的**形象设定**（Q 版 3.5 头身、蓝发、深色女仆裙 + 白围裙、温柔俏皮），但**资产全部自绘**。
理由不是"想换画风"，而是成品视频在物理上做不到与房间物件交互：

| 交互需要的能力 | 成品视频（webm 等） | 自建骨架 |
|---|---|---|
| 手部挂点（纸/铃/锅放进手里） | ❌ 只能事后叠一层"假贴图"：不跟手、会被自己挡住、无法交接给人 | ✅ socket 属于骨架，道具与手同帧变换 |
| 4 向朝向（上/下/左/右） | ❌ 只有左右镜像，没有"背影朝上" | ✅ 三个方向母版 + 镜像 |
| y 排序遮挡（被柜台挡住） | ❌ 视频是一整块矩形，永远压在最上层 | ✅ 以角色 `feetY` 参与排序 |
| 接触帧事件（"按铃那一下"） | ❌ 时间信息在像素里，拿不到 | ✅ clip 时间轴上的 `event` 标记 |
| 光照 / 变色 / 缩放 | ❌ 要重出素材 | ✅ 图层 tint + 程序化缩放 |
| 与场景画风、像素密度一致 | ❌ 640×360 实拍风在 2× 场景里会糊 | ✅ 同一套描边与色板 |
| 版权 | ❌ dsh-pet 素材明确"禁止商用" | ✅ 自有 |

> 一句话：**视频只能"表演"，骨架才能"操作"**。

**形象设定（可直接交付给美术）**

| 项 | 规格 |
|---|---|
| 头身比 | 3.5 头身（Q 版）；头宽 ≈ 0.55 tile |
| 身高 | 112 px @1x（1.75 tile），脚底 = `feetY` 锚点 |
| 配色 | 发 `#7FC8F8` / 发暗部 `#5FB4E8`、裙 `#2F3A48`、围裙 `#F7F3EC`、肤色 `#FFE3D0`、腮红 `#F6A9A9`、描边 `#26313F` |
| 描边 | 统一 2px（@2x 即 4px）；只描外轮廓与内侧转折，不逐层描 |
| 表情 | 6 个覆盖层：普通 / 开心 / 困惑 / 冒汗 / 红脸 / 星星眼（**只做 `down` 向**，`up` 向只有后脑） |
| 朝向 | `down` / `up` / `side` 三套母版；**`right = mirror(side)`，不重画** |
| 交付 | 分层 PNG @2x（512×512 画布，脚底与身体中线对齐）+ `rig.json`（pivot / socket / clip / event） |

### 6.2 骨架、挂点，与"道具是一等对象"

角色不是一个整图，而是 **47 个部件 + 6 个挂点**（逐件见美术素材清单 §2.1 / §2.2）：

```
后发 ─┬─ 头（6 表情覆盖）─ 前发/刘海
      │       └ socket: head / chest（领结）
  躯干 ─┤
      │   ├ 上臂 L/R ─ 前臂 L/R ─ socket: handL / handR   ← 拿东西的唯一入口
      │   └ 围裙 / 裙摆（2 骨摆动）
   腿 L/R（walk 用 2 帧替代骨骼弯曲，省一半工作量）
```

五条关键约定：

1. **道具是房间的一等对象**，不是角色素材。`attach('handR', propId)` 把房间里那个 `文件夹`
   实例挂到手上，`detach` 放回柜台 —— "纸被签了要换图"只换道具纹理，角色完全不动。
2. **手持道具继承持有者的 `feetY`（+1px）**，所以它跟角色一起参与 y 排序、一起被柜台挡住。
3. **接触帧**：clip 可带事件（`{clip:'ring_bell', t:0.24, event:'bell'}`）。房间收到 `bell` 才摇铃、
   才发声 —— "手到了才响"，而不是"一进等审批状态就响"。
4. **朝向由目标决定**：`faceToward(tileCenter)`，转身是 ≤80ms 交叉淡入（不做转身动画，省 3 个 clip）。
5. **位移与动画解耦**：走路 = 格子寻路逐帧算坐标 + 播 `walk` clip；clip 内不含位移。
   于是速度可调、`prefers-reduced-motion` 可直接瞬移，而动画不用重做。

### 6.3 动作清单（26 个 clip；12 循环 + 14 一次性，手绘仅 50 帧）

分两类：**12 个循环**（由 rig 程序化驱动，不额外画帧）+ **14 个一次性演出**（手绘序列帧，见附录 D.2）。

**循环类（rig 程序化）**

| clip | 帧 | 用途 | 事件标记 |
|---|---|---|---|
| `idle_breathe` | 4 | 待机呼吸（默认循环） | — |
| `idle_look` | 4 | 东张西望 | — |
| `walk_down` / `walk_up` / `walk_side` | 各 6 | 四向行走（`side` 镜像出 right） | `step` 每 2 帧 |
| `think_chin` | 4 | 托腮思考 | `bubble` @0.5 |
| `work_type` | 6 | 键盘打字 | `key` @0.1 循环 |
| `work_read` | 6 | 翻书 / 翻文件 | `page` @0.5 |
| `work_cook` | 6 | 颠锅 | `spark` @0.33 |
| `work_look` | 4 | 望远镜张望 | — |
| `wait_stand` | 4 | 踮脚张望（等审批 / 等回答） | — |
| `answer_raise` | 4 | 举手（在回答问题） | — |
| `sad` | 4 | 垂头（失败停留） | — |
| `sleep` | 4 | 趴睡（无会话 / 空闲 > 5min） | — |

**一次性类（手绘序列帧）**

| clip | 帧 | 用途 | 事件标记 |
|---|---|---|---|
| `pin_note` | 6 | 白板贴便签 | `pin` @0.4 |
| `pick_book` | 5 | 从书架抽书 | `take` @0.3 |
| `carry_walk` | 6 | 抱着东西走（循环） | — |
| `offer_paper` | 6 | 双手递纸 | `offer` @0.5 |
| `ring_bell` | 3 | 按铃 | `bell` @0.24 |
| `stamp_paper` | 3 | 盖章 | `stamp` @0.2 |
| `seal_clone` | 8 | 结印 | `seal` @0.6 |
| `merge_clone` | 6 | 分身归位 | `merge` @0.4 |
| `cheer` | 6 | 雀跃庆祝 | `confetti` @0.2 |
| `run_front` | 6 | 跑向镜头 | — |
| `panic` | 4 | 抱头蹲下 | — |
| `pack_box` | 8 | 搬箱 / 封箱 | `seal_box` @0.6 |
| `snack` | 8 | 吃零食 | `bite` @0.3 |
| `play_cube` | 8 | 玩魔方 | — |

复用率高是这套设计的关键：同一个 `work_type` 同时服务"写文件 / 改代码 / 敲配置"三种工具；
`carry_walk` + `offer_paper` 同时服务"递审批纸"和"交付托盘"。

### 6.4 phase → clip / 朝向 / 手上道具

| phase | clip | 朝向 | handR / handL | 房间联动 |
|---|---|---|---|---|
| `offline` | `sleep` | `down` | — | 主灯灭、窗外夜色 |
| `idle` | `idle_breathe` ↔ `idle_look` ↔ `snack` ↔ `play_cube`（随机链） | `down` | 零食 / 魔方 | 暖光 |
| `attending` | `walk_<dir>` | 朝工作台 | — | 顶灯亮一档 |
| `thinking` | `think_chin` | `up` | — | 显示器灰线、冷蓝光 |
| `planning` | `pin_note` → `idle_breathe` | `up` | 便签 | 白板便签 +1 |
| `working` | 按工具类别：`work_type` / `work_read` / `work_cook` / `work_look` | `up` / `left` | 对应工具 | 工位射灯、火花 / 蒸汽 |
| `delegating` | `seal_clone` → `wait_stand` → `merge_clone` | `down` | — | 最多 4 个分身小人跑向庭院 |
| `waitingApproval` | `carry_walk` → `offer_paper` → `ring_bell`（>12s） | `down` | **文件夹**（真道具） | 柜台射灯、`bell` 事件摇铃 |
| `waitingAnswer` | `answer_raise` | `up` | 选择板 | 紫环 + 问号气泡 |
| `compacting` | `pack_box` | `up` | 箱子 | 纸箱堆叠 |
| `result` | `work_read`（0.6–1.4s） | 工位方向 | 文件夹 | 传送带轻转一格 |
| `succeeded` | `run_front` → `cheer` | `down` | 完成印章 | 暖金 + 彩带粒子 |
| `failed` | `sad` / `panic` | 工位方向 | — | 红闪 2 次 + 黑烟 |
| `composing` | `idle_look` | `down` | — | 柜台柔光 |

**修饰位对 clip 的影响**（不换 phase，只换 clip / 参数）：

| 修饰位 | 影响 |
|---|---|
| `goalRound` | `succeeded` **降级为 `result`**（不庆祝）；工作循环里插入 0.4s"瞄一眼大屏"（`work_look` 半段） |
| 长任务（>60s 无事件） | 循环速度 ×0.8 + 插入 `wait_stand` 与 `sad` 各 1 次 |
| `contextPressure` 2 | `compacting` 变体：`pack_box` 连续播 2 次 |
| `reducedMotion` | 只播每个 clip 的**首帧**（静态立绘）；位移瞬移；事件标记仍按时间触发 |

### 6.5 特殊演出编排（分镜级说明）

**① 分身术（`delegating`）**

```
0.0s  角色抬头（预兆：头顶"！"）
0.3s  结印起手（`seal_clone` 的前 6 帧）
1.1s  1.5s 结印：双手合十 + 影分身烟雾（3 团烟）
1.6s  每个 subagent 生成一个小人：同一角色资源，scale 0.35、饱和度 0.8、
      头顶编号 #1 #2，从烟雾里弹出（easeOutBack），各自跑向庭院不同位置
      —— 小人用**同一套 rig 的精简版**（头/躯干/腿三层，scale 0.45），能真的跑动、做小动作、被遮挡
进行中 主线角色留在原地"遥控"（双手前伸、指挥动作），
      庭院里小人各自做"干活"小循环；每归位一个，小人跑回主线合体
结束  最后一个 subagent/end → 全员归位 → 拍灰 → 回到上一个工位
```
- 并发上限：同时最多渲染 4 个小人（第 5 个起只显示 "+N" 徽标）。
- 失败分支：某个子 agent 失败 → 对应小人变红 + 抱头，归位时主线角色安慰式拍头。

**② 求签字（`waitingApproval`）**

```
0.0s  柜台射灯亮 + 铃铛蓝环
0.2s  角色从当前工位小跑到柜台（走位图）
1.4s  抱起文件夹（道具叠加在 handAnchor），转身面向镜头
2.0s  踮脚递纸 + 期待表情；纸面用 DOM 渲染"待确认：<toolName>" + 两个按钮
      主按钮「去确认」= 深链到真实审批卡
      次按钮「就在这里签」只在 approveMode === 'inRoom' 时出现（M3，见 §9.3 ④）
>12s  按铃催促（每 12s 一次，最多 3 次，之后改为静静等）
结果  allowed-once → 盖章"✓"飞走 → 跑回工位继续
      rejected → 纸被退回，角色垂头 2s → 回 idle
```
- 与 `waitingAnswer` 的区分：审批=**铃 + 纸**；提问=**举手 + 问号**（不同颜色高亮环，蓝 / 紫）。

**③ 收工通知（`succeeded`）**

```
0.0s  当前动作收尾（0.2s 硬切到"站直"）
0.2s  跑向镜头（亮度提升 10%，镜头推近 3%）
1.2s  停在镜头前，雀跃庆祝 + 彩带粒子 + 暖金主灯
1.4s  通知卡从右上滑出（贴纸感），带"看看成果"按钮
4.0s  卡片自动收起；角色转身回沙发进入 idle
```

**④ 压缩记忆（`compacting`）**：纸箱随压缩比例堆高 → 封箱（胶带音效 M3）→ 搬进柜子 → 桌子变清爽（视觉上"干净了"）。这是唯一会**减少**房间道具数量的状态，正好和 `contextPressure` 的单向增长形成节奏。

### 6.6 播放器（clip 混合 + 事件 + 性能）

```
输入：{ phase, flags, station, holding }
  │
  ├ 选 clip：phase → §6.4 表；修饰位只改播放参数（速度 / 降级 / 首帧冻结）
  ├ 选朝向：station.face → down/up/left/right；交互途中可 faceToward(目标)
  ├ 过渡：两 clip 交叉淡入 120ms（骨骼插值，不是整图叠化）
  ├ 事件：clip 时间轴事件进房间总线（bell / offer / key / spark / confetti…）
  └ 兜底：素材缺失 → 画灰色剪影 + 保留足底锚点，绝不白屏、绝不漂移
```

- **每帧只做一件事**：更新 transform 与帧索引，然后整体重绘一次 Canvas；**不 setState**。
- **状态与表现分离**：React 只在 `phase / station / bubble / pending` 变化时重渲染 UI 层；
  逐帧动画完全跑在 rAF 里 —— 这就是 D12 选 Canvas 的收益。
- **事件去重**：同一 clip 的同一事件在一次播放内只触发一次；循环 clip 每圈触发一次。
- **预加载**：进入某 phase 前预取它需要的图集分片（角色 1 + 家具 1 + 道具 1）。
- **后台标签**：`document.hidden` → 停 rAF、停粒子、只算状态；回到前台按 `since` 直接快进到当前 clip 的正确帧。

---

## 7. 视觉规范

| 元素 | 规范 |
|---|---|
| 面板/卡片 | 圆角 12px，`--dsw-alias-bg-layer-1`，1px `--dsw-alias-border-l1`，阴影只一层硬投影 |
| 气泡 | 圆角 14px + 小尖角指向角色；正文 `--dsw-alias-label-primary`，底色 `--dsw-alias-bg-overlay`；最大宽度 = 角色宽的 0.5（dsh-pet 的实测值，约 11 个汉字/行） |
| 状态标签 | 工位上方小胶囊，图标 + 2–4 字（"搜索中/写文件/等确认"），字号跟随主题 `font-size` |
| 通知卡 | 右上滑入 320×88，带图标 + 一句话 + 一个动作按钮 |
| 字体 | 复用 DSH 字体栈；**不要**自带字体文件（dsh-pet 自带软糖体是特例，房间页要保持宿主一致性） |
| 图标 | 自绘 SVG 24×24 线性图标（工位/状态各一套），描边 2px，`currentColor` |
| 深色模式 | 房间内部色板提供 dark 变体（墙 `#2A2622`、地板 `#3A2E24`、灯光更暖更暗），通过 `ctx.theme.getTheme()` 快照切换；外壳永远用令牌 |

---

## 8. 数据契约

### 8.1 `RoomState`（wire 与本地面板共用）

```ts
type RoomPhase =
  | 'offline' | 'idle' | 'attending' | 'thinking' | 'planning' | 'working'
  | 'delegating' | 'waitingApproval' | 'waitingAnswer' | 'compacting'
  | 'result' | 'succeeded' | 'failed' | 'composing';

type StationId =
  | 'desk' | 'shelf' | 'stove' | 'window' | 'board'
  | 'counter' | 'yard' | 'sofa' | 'conveyor' | 'door';

type ToolCategory =
  | 'fs-read' | 'fs-write' | 'shell' | 'web' | 'delegate'
  | 'workflow' | 'ask' | 'plan' | 'goal' | 'skill'
  | 'job' | 'schedule' | 'deliver' | 'plugin' | 'mcp' | 'other';

interface RoomState {
  v: 1;                        // 契约版本，改动形状必进位（老 Client 要能优雅降级）
  sessionId: string;
  phase: RoomPhase;
  station: StationId | null;   // 当前站位（走位目标）
  tool: { name: string; category: ToolCategory; callId: string; startedAt: number } | null;
  plan: { current: string | null; done: number; total: number } | null;
  children: Array<{ id: string; label?: string; depth: number; status: 'running'|'done'|'failed' }>;
  /** 待用户动手的那件事：审批 / 提问 / 计划评审。null = 没有要你干的（见 §9.3） */
  pendingInteraction:
    | { kind: 'approval'; toolName: string; reason?: string; callId?: string }
    | { kind: 'question'; callId: string; count: number; first?: string; timed: boolean }
    | { kind: 'plan-review'; callId: string }
    | null;
  /** 传送带上的待发订单（queue 模式入队的用户消息），供房间内编辑/撤回（见 §9.1 附加项） */
  queue: Array<{ itemId: string; text: string; mode: 'queue' | 'steer' }>;
  /** 房间被允许做哪些事：Host 按 config 算好下发，Client 照此渲染，不自作主张 */
  actions: {
    cancel: boolean; prompt: boolean; answer: boolean; queue: boolean; interruptChild: boolean;
    approve: 'off' | 'link' | 'inRoom';
  };
  lastEnd: { kind: TurnEndReasonKind | 'unknown'; message?: string } | null;
  flags: { goalRound: boolean; closing: 'complete'|'blocked'|null; contextPressure: 0|1|2; multiBusy: number };
  seq: number;                 // 最近吸收的事件 seq（同相去重的判据）
  since: number;                // phase 进入时间（用于长任务变体/超时漂移）
}

// 注意：`pendingAction`（0.6–0.8s 的"收到，我这就办"）是 **Client 本地瞬态**，
// 不进 wire state、不参与同相去重 —— 它属于"我按了按钮"，不属于"任务发生了什么"。
```

### 8.2 `wire.view` 与快照

- 投影 key：`dsh-room`，`wire: { view: (state) => state }`——**必须返回同一引用**当内容未变，否则会持续推送。
- 快照语义：投影片段（projection snapshot）随 session 持久化，冷启动可从尾部回放重建；`stateVersion` 进位策略：字段/折叠语义变更时 +1，缓存自动失效。
- 迷你房间快照：`GET /dsh-room-<hash>/state?since=<counter>&timeout=<ms>`，返回 `{ counter, sections: { focus: {...}, rooms: {...} } }`；**老宿主不带 counter 要退化为定时轮询**（否则立刻返回变狂打）。

### 8.3 节流与背压

| 通道 | 节流 |
|---|---|
| 投影推送 | 由框架按 `state` 引用变化推送；fold 内部保证"无变化不新建对象" |
| 长轮询 | 服务端在 `counter` 未变时挂起 ≤ 25s；客户端同一时刻只挂 1 个请求 |
| 动画切换 | ≤ 2 次/秒（交叉淡入 200ms，硬限制 500ms 冷却） |
| 气泡 | ≤ 1 条/2s，同类 5s 内不重复 |

---

## 9. 交互设计（房间是操作台）

> 这一章是 v0.2 新增的主体。核心原则：**每个操作在 Host 半边只实现一次**，房间页/浮窗只是两个入口；**用户权限类操作只用宿主既有 API**，绝不自己发明一套授权。

### 9.1 交互清单（四类核心 + 三类附加）

| 交互 | 宿主 API（唯一实现） | 房间页入口 | 浮窗入口 | 可逆性 |
|---|---|---|---|---|
| **对话**（发消息 / 插话） | `sessionController.prompt({ sessionId, requestId, mode: 'queue'\|'steer', content, clientTimeZone })` | 优先 `inputActions`（宿主白给）；不可用时 `POST /action` | `POST /action` | 可撤回（队列里可删） |
| **中止当前回合** | `sessionController.cancel({ sessionId })` → `{ accepted: true }` | 急停拉杆 / `inputActions` 的 stop 路径 | 拉杆按钮 | 不可逆（但**不丢 inbox**，见 R11） |
| **回答问题**（含计划评审） | `userQuestions.answer(agent, callId, { answers: [{ id, selected[], custom? }] })` | 房间里的"选择板" | 小卡片 | 不可逆（提交即定） |
| **确认审批** | 官方 answerer（v1 不接）／自建 answerer（M3 opt-in） | 纸张 + 「去确认」主按钮（深链）；M3 可「在房间里签」 | 纸牌/徽标 + 深链 | 不可逆 |
| *附加*：队列管理 | `sessionController.updateQueue({ sessionId, itemId, action: { kind: 'edit'\|'remove'\|'steer' } })` | 传送带上还没送走的"待发订单" | 同 | 可逆 |
| *附加*：打断子 agent | `subagents.interrupt(childSessionId, { kind: 'user', parentSessionId })` | 点小人的头 | 点徽标 | 不可逆（子 agent 停止） |
| *附加*：换模型 / 换预设 | `sessionController.selectModel({ sessionId, provider, model })` | 技能柜 / 换装镜 | 设置菜单 | 可逆 |

> 为什么"对话"不直接用宿主的 composer：房里的输入是**独立草稿**（§9.7），提交走的是同一条 Host API，所以消息在队列/transcript 里的记录与宿主完全一致，不会出现两份。

### 9.2 房间指令协议（唯一上行口）

```
POST /dsh-room-<hash>/action
content-type: application/json
```

```jsonc
// 请求
{
  "op": "prompt" | "cancel" | "answer" | "approve" | "queue" | "interrupt-child" | "select-model",
  "requestId": "uuid-v4",     // 幂等键：同一 UI 动作重试不会重复生效（Host 60s LRU 去重）
  "sessionId": "…",           // 必须是当前焦点会话，不符直接拒（防串会话）
  "callId": "…",              // answer / interrupt-child 必填，用于陈旧校验
  "seq": 1284,                // 客户端渲染时的最近 seq；小于宿主当前状态即视为陈旧
  "payload": { … }            // 按 op 定：prompt={text, mode} / answer={answers} / queue={itemId, action}
}
```

| 响应 | 含义 | 房间反应 |
|---|---|---|
| `{ ok: true }` | 已受理 | `pendingAction` 转"已递出"，等真实事件落地 |
| `{ ok: false, code: 'stale' }` | `seq` 或 `callId` 已过期 | 收回控件 + 蓝字"这一步已经过去了" |
| `{ ok: false, code: 'not-running' }` | 没有活动回合却要 cancel | "其实已经停了"，顺手刷一次状态 |
| `{ ok: false, code: 'wrong-session' }` | 焦点会话已切换 | 静默丢弃 + 提示"房间已换人" |
| `{ ok: false, code: 'no-agent' }` | 会话没有 live agent（冷会话） | 提示"需要先打开这个会话" |
| `{ ok: false, code: 'question-gone' }` | 问题已被别处回答 / 超时 | 控件禁用，显示"问题已过期" |
| `{ ok: false, code: 'rate-limited' }` | 单位时间提交过多 | 按钮冷却 2s |

**Host 侧校验顺序（固定，不许跳步）**：① 会话匹配 → ② Agent 存活（`ctx.agents.get`）→ ③ `callId`/`seq` 新鲜度 → ④ 幂等 `requestId` → ⑤ 速率限制 → ⑥ 调宿主 API → ⑦ 记一条房间本地遥测（不写 session 日志）。

### 9.3 三个可操作相的房间 UI 与演出

**① 对话（`composing`）**

```
空闲时：柜台边常驻一个"写字台"小输入框（房间页右下，迷你房间是底部一行）
聚焦：柜台柔光亮起，角色走过来抬头听（composing 相）
提交：写好的纸条被塞进传送带 → 传送带转动 → 纸条消失；
      queue 通道 = 纸条落进"待办篮"（还在队里，可抓回来改）
      steer 通道 = 纸条被"啪"一下插进正在跑的活里（角色抖一下继续）
失败：纸条退回手里 + 红字原因；草稿不丢
```
- 投递方式默认由配置 `interaction.promptMode` 决定（`ask` = 两个按钮让用户选）；`Ctrl/Cmd+Enter` 用另一种。
- 提交**不预置** `thinking`：等 `user/message` 落地才切相（§3.4）。
- `requestId` = 房间侧生成的 uuid，重试复用同一个 → 不会重复入队。

**② 中止（急停拉杆）**

```
位置：房间左下（灶台旁）一根红色拉杆，只在 working/thinking/delegating 相可用
交互：点击 → 拉杆下落 0.2s → 角色"撒手扔下工具"（0.3s 硬切）→ 立刻提交 cancel
反馈：pendingAction 期间角色站在原地看着镜头喘气，房间主灯闪一下；
      turn/end{aborted} 落地 → 走到沙发坐下，气泡"我停下了"
      turn/end{completed} 竞态赢了（刚好结束）→ 不报错，静默转成功
不做二次确认：cancel 不丢队列、不删数据，误触代价低；按钮 hover 文案写明"保留排队消息"
```

**③ 回答问题（`waitingAnswer`）**

```
角色举手 + 头顶问号 → 走到柜台，把一块"选择板"举到你面前
板面：问题标题（header） + 选项按钮（label + description） + 「自己填」输入
     多选（multiSelect）支持勾选；每题一块板页，右上角显示"第几题 / 共几题"
提交：板子飞向角色 → 角色点头 → 举手放下
倒计时：若宿主用了前台等待（`attachWait` 有帧）→ 板子右上角显示剩余时间；
      到点 → 控件禁用 + "问题已过期（可以让我重新问）"
计划评审：同一块板子，标题加"计划待批准"角标（`AskUserQuestionIntent.kind === 'plan-review'`）
```

**④ 确认审批（`waitingApproval`）——v1 只做"镜像 + 深链"**

```
纸张正面：待确认的工具名 + 原因（reason / displayReason 已本地化）
按钮 1（主，蓝色）：「去确认」→ 切到 Chat View 并聚焦该审批卡（焦点走 useConversation/inspect）
按钮 2（次，仅 M3 且 approveMode='inRoom'）：「就在这里签」
      需要长按 400ms 才生效（防误触），签完纸面盖"✓"飞走
>12s：角色按铃催促；不做自动放弃
```

### 9.4 幂等、陈旧与超时（三张保险）

| 场景 | 保险 |
|---|---|
| 用户双击"确认" | `requestId` 幂等键 → 第二次返回 `{ok:true, deduped:true}`，不重复生效 |
| 点完 200ms 后事件翻页 | `seq` + `callId` 陈旧校验 → `stale`，控件变灰并说明 |
| 焦点会话被切走 | `sessionId` 校验 → `wrong-session`，旧控件全部禁用 |
| 网络抖动 / Host 卡住 | 800ms 无回执 → `pendingAction` 转黄"没动静"；3s → 红字 + 可重试（同一个 `requestId`） |
| 问题过期 | 服务端 `attachWait` 帧驱动倒计时；过期后禁用并提示 |
| 房间页与浮窗同时打开 | 同一 `requestId` 去重；两边都从同一份状态收敛（不会一个显示"已签"、另一个还能点） |

### 9.5 权限边界（照抄宿主规范，不自创）

宿主规范明确：**"授予或确认权限的动作 —— 批准工具调用、回答 agent 的提问、放宽策略 —— 必须只由用户执行"**。因此：

1. 房间里的这些按钮是**用户动作**，合法；但**绝不给 Agent 一个能点它们的工具**（不注册 `room_approve` 之类的工具）。房间只提供 UI，不提供 agent 通道。
2. 审批的默认路径是**深链**：房间不做决策者，官方卡才做。这样 `ask` 策略的 fail-closed 语义（没有 answerer 就 `unavailable`）完全不被改变。
3. `approveMode: 'inRoom'` 是**显式 opt-in**，实现方式是 Host 侧注册一个 `approval/request` 监听器：
   - 只有当 ①配置开启 ②房间客户端在线且订阅了该 session 的待确认 ③请求的 `toolName` 在白名单里 —— 三者同时满足时**认领**（返回一个等房间提交的 Promise）；
   - 否则立即 `next()`，把请求完整交给官方终局 answerer；
   - 认领后**不设自动放弃**：等不到就继续等，同时房间弹强提示；宁可慢，不可让一个"已认领"的审批变成 `unavailable`（那会让工具直接失败）。
4. 已想清楚不做的事：**不做"批量允许/永久放宽策略"**（那是权限预设的活，抢过来只会绕过审计）。

### 9.6 键盘与焦点

| 键 | 作用域 | 行为 |
|---|---|---|
| `Enter` | 房间输入框聚焦 | 发送（按当前投递方式） |
| `Shift+Enter` | 同上 | 换行 |
| `Ctrl/Cmd+Enter` | 同上 | 用另一种投递方式发送 |
| `Esc` | 房间内有输入/面板 | 收起输入/面板（**不**中止任务，避免与宿主的 Esc Esc 语义冲突） |
| `Space` | 房间页无焦点 | 暂停/继续动画 |
| `Tab` | 房间页 | 在工位与可操作控件间顺序移动（全部带 `aria-label`） |

- 房间输入框只在**自身聚焦**时拦截按键；任何情况下不注册全局快捷键（宿主的 `Esc Esc` 停任务、快捷键插件仍在位）。
- 等待相（`waitingAnswer`）里选项按钮是标准 `button`，不用 `Y/N` 单键 —— 单键确认太容易误触。

### 9.7 与宿主 composer 的关系

- **草稿独立**：房间输入不回填、不读取宿主 composer 的 draft，避免双写同一份草稿。
- **消息不重复**：提交成功后不在房间再渲染一条本地 echo，等 `user/message` 落地后由状态机表达（会走 `attending` 相）。
- **可选联动（M3）**：读 `useInput` 快照，在房间显示"对面正在输入…"，只是提示，不参与控制。
- **多端**：房间页与浮窗同时在线时，两者共用同一个 Host 去重表，界面各自收敛到同一份状态。

### 9.8 纯展示类交互（保留）

| 交互 | 行为 |
|---|---|
| 悬停角色 | 气泡显示"正在 <工具名>，已 12s"（悬停不打断动作） |
| 点击角色 | 气泡展开：当前 phase + 当前 todo + 当前队列条数；再次点击收起 |
| 点击工位/物件 | 打开该 `callId` 的 tool 卡片（房间页有 `inspectCall` prop） |
| 点击纸张/铃铛 | 深链到真实审批卡（= §9.3 ④ 的主按钮） |
| 右键角色 | 缩放（S/M/L）、主题、暂停动画、重新载入素材 |
| 拖拽角色 | 抓起，松手落回最近节点（M2；`prefers-reduced-motion` 关闭） |
| 迷你房间头部 | 拖拽移动、双击收起为"只显示角色"、右键切到房间页 |
| 命令 | `/room` 打开房间页；`/room off` 关浮窗（M3） |

### 9.9 三个渲染面的能力矩阵

同一个"操作语义"在不同渲染面上的**表达完整度**不同，但**结果一致**（都走同一条 Host API）：

| 能力 | 房间页 | 迷你浮窗（260×180） | 窄屏角色卡 |
|---|---|---|---|
| 看状态与动作 | ✅ 全景 | ✅ 角色 + 一句话 | ✅ 角色 + 文字状态条 |
| 发消息 | ✅ 完整输入框 + 投递方式切换 | ✅ 一行输入（点击展开为弹层） | ✅ 弹层输入 |
| 中止回合 | ✅ 拉杆 | ✅ 小按钮 | ✅ 小按钮 |
| 回答提问 | ✅ 选择板（多题翻页 + 自由输入） | ⚠️ 只放"单选 / 两题以内"，多题给"去房间里回答" | ❌ 只给"去回答"按钮（深链） |
| 确认审批 | ✅ 纸张 + 两种按钮 | ⚠️ 徽标 + 深链（不显示全文原因） | ❌ 深链 |
| 队列编辑 / 撤回 | ✅ | ❌ | ❌ |
| 打断子 agent | ✅ 点小人的头 | ⚠️ 小人列表点选 | ❌ |
| tool 卡片详情 | ✅ `inspectCall` | ❌（只显示工具名 + 已用时长） | ❌ |
| 缩放 / 主题 / 暂停 | ✅ 右键菜单 | ✅ 头部菜单 | ✅ 设置 |

**降级原则**：能力缺一格时，**永远给一个"去房间里做"的出口**，而不是把按钮藏掉让用户怀疑插件坏了。

---

## 10. 配置项（`cordis.patch.yml` 的 `config`，全部进 `Config` schema）

| 键 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `placement` | `'page' \| 'overlay' \| 'both'` | `'both'` | 渲染面 |
| `overlay` | `{ corner, marginX, marginY, width, collapsed }` | `top-right, 24, 96, 260, false` | 迷你房间位置尺寸 |
| `theme` | `'kitchen-day' \| 'kitchen-night' \| 'auto'` | `'auto'` | `auto` = 跟随宿主明暗 |
| `character` | `'maid' \| 'custom'` | `'maid'` | 角色资源根（`custom` 走 pet pack 式目录） |
| `animationSet` | `Record<RoomPhase, string[]>` | 内置映射 | 每相候选动画（数组=等概率抽） |
| `phaseWeights` | `Record<string, number>` | 内置 | idle 随机链权重（复用 dsh-pet 的 rollKind 思路） |
| `bubble` | `{ enabled, maxChars, durationMs }` | `true, 40, 8000` | 气泡 |
| `notify` | `{ card, system, onSuccess, onFailure, onApproval, onQuestion }` | `true,false,true,true,true,true` | 通知（系统通知默认关，避免和 dsh-pet 撞） |
| `interaction` | `{ enabled, prompt, cancel, answer, approve, queue, interruptChild }` | `true`（`approve` 除外，默认 `false`） | 逐项开关房间内交互：本想当纯看板的用户可一键关掉，不必卸载插件 |
| `interaction.approveMode` | `'link' \| 'inRoom'` | `'link'` | 审批只深链 / 允许房间内签（opt-in，见 §9.5） |
| `interaction.approveTools` | `string[]` | `['pwsh','bash','write','edit']` | `inRoom` 认领哪些工具的审批（白名单，宁可窄，别一上来就 *） |
| `interaction.promptMode` | `'ask' \| 'queue' \| 'steer'` | `'ask'` | 房间提交的默认投递方式（`ask` = 两个按钮让用户选） |
| `interaction.confirmDangerous` | boolean | `true` | 中止/审批是否要二次确认（长按 400ms） |
| `interaction.queueCap` | number | `20` | 传送带上最多显示几条排队消息 |
| `motion` | `{ honorReducedMotion, fpsCap, particles }` | `true, 30, true` | 动效预算 |
| `offlineAfterMs` | number | `900000` | 空闲多久进"夜晚/打瞌睡" |
| `driftAfterMs` / `longTaskAfterMs` | number | `8000` / `60000` | 漂移 / 长任务变体阈值 |
| `resultHoldMs` | number | `600` | `result` 相最短驻留（去抖） |

配置非法时**大声报错**（红色错误条 + 重试），绝不静默兜底成"看不出来发生了什么"——这是 dsh-pet 的一条硬经验。

---

## 11. 性能与降级

| 项 | 策略 |
|---|---|
| 绘制 | 一次 Canvas 扫描：地板 1 层 + 排序层 ≤ 60 个 drawImage + 覆盖层；分身 ≤ 4 |
| 首屏预算 | 图集分片按需加载：首帧只等 `char` + `room`（各 ≤ 1.2 MB，四张合计 ≤ 3 MB） |
| DOM 预算 | 房间页 ≤ 600 节点；粒子 ≤ 40；分身影 ≤ 4 |
| 后台 | `document.hidden` → 暂停媒体、停 rAF、只算状态 |
| 低端机 | `fpsCap: 30` + 关闭粒子 + 动画降级为静态首帧轮播（每 1s 换一帧，视觉上仍"活"） |
| 断线 | `connection/reset` → 房间进 `offline`，显示"重连中"，恢复后从投影快照重建（不重放旧庆祝） |
| 首帧 | 先画地板 + 家具剪影 + 角色首帧，图集就绪后重绘，避免"白房间" |
| `prefers-reduced-motion` | 位移归零、动画换静态帧、粒子关闭、**信息不丢**（气泡/标签/通知照常） |
| 交互请求 | 同一时刻 ≤ 1 个在飞；按钮提交后禁用 800ms；`/action` 速率限制 8 次/分钟（**Host 侧强制**，不靠前端自觉） |
| 交互失败 | 一律显式报错（`code` + 人话），绝不静默吞掉；草稿/选项保留可重试 |

---

## 12. 无障碍与国际化

- 房间是可理解的信息图：每个工位/状态都有 `aria-label`（"工作台，正在写文件，已进行 12 秒"），并提供**文字版状态条**（房间页右上角一行纯文本，屏幕阅读器优先读它）。
- 不依赖颜色单独传达状态（红绿同时有图标与文字）。
- 全部 UI 文案走 `ctx.locale`：Client 侧 `register(ns, dicts)` + `bind(ns)`；`meta.title`/`meta.description` 写 `locale/en.json` 与 `locale/zh.json`。
- 动画可暂停（`Space`），且暂停时状态照常更新。
- 素材命名中文化（与 dsh-pet 一致），但**文件名不参与展示**，展示名来自 locale 字典（避免中英混排）。

---

## 13. 风险与开放问题

| # | 风险 | 影响 | 对策 |
|---|---|---|---|
| R1 | **美术产能**（v0.3 起取代原来的"素材版权"风险） | 自建角色 + 场景 **242 件 / 23 人天**（附录 D.7）；没有美术就没有角色 | ①先做 P0 最小可跑集（8 天，附录 D.7.2），期间用灰色剪影就能验证交互；②`character` / `theme` 都做成可插拔资源根，社区可换皮；③**不再引用 dsh-pet 素材**（其许可禁商用，且技术上不可交互） |
| R2 | 房间页占用 View tab，进房间就看不到 Chat | 用户可能"进去看两眼就出来"，日常可见性靠浮窗 | 浮窗为主、房间页为辅；房间页顶部提供"回到对话"一键；查看审批等操作一律深链回 Chat |
| R3 | Client 只有 4 个全局事件，状态必须从 Host 来 | 若 Host 半边被禁/在某些 profile 缺失，房间只剩空壳 | `dsh.client.inject` 声明依赖、Host `inject: ['webServer']`；Host 不可用时房间显示"仅静态展示"而不是报错白屏 |
| R4 | **审批认领的僵局风险**：官方 `dsh-client-ui-approval` 是**终局 answerer**；我的监听器若排在前面并认领，官方卡就不会出现 —— 用户没看房间时任务会一直等 | 用户以为"卡住了"，最终可能超时/失败 | ①默认 `approveMode:'link'` 只深链；②`inRoom` 必须 opt-in + 工具白名单 + 认领后房间强提示（通知卡+可选声音）；③**认领后不自动放弃**（放弃=`unavailable`=工具直接失败）；④设置页写明这个开关的后果 |
| R5 | `conversation.view` 的 hooks 由宿主提供，跨版本签名可能变 | 升级后组件崩 → slot 显示"entry crashed" | 只用稳定子集（`sessionId`、`useConversation`、`useProjection`、`inspectCall`）；对每个 hook 做存在性检查；崩了降级为"纯 Host 快照渲染"（不依赖任何 hook） |
| R6 | 图集内存：四张 @2x 各 ~1.2 MB + Canvas 后备缓冲 | 低端机卡顿、显存吃紧 | 图集分片按需加载 + 单张长边 ≤ 2048 + `fpsCap` 降级 |
| R7 | 状态精度：`result` 与 `working` 的高频交替 | 角色疯狂抽搐 | 滞回（600ms 最短驻留）+ 同相不重播 |
| R8 | 多会话并发时"谁代表房间" | 语义混乱 | 焦点会话规则（最近活跃 running，浮窗所在会话优先）；其余以门牌指示 |
| R9 | 与 dsh-pet 同时开启时的**视觉打架**（桌面上两只一样的角色） | 用户困惑 | 检测 `dsh-pet` 是否已装载（`ctx.get`/service 探测）；若同时启用，默认把房间浮窗收起为"迷你门牌"，并在设置里提示 |
| R10 | 交互误触与陈旧：点"确认"时事件已翻页 | 可能确认一个已经消失的调用，或中止掉刚结束的回合 | §9.2 的 `seq`/`callId` 校验 + `requestId` 幂等 + 危险动作长按 |
| R11 | `cancel` **不丢 pending inbox**（"without dropping its pending inbox"） | 用户点急停以为清空了，结果下一条消息又自动跑起来 | 房间明确显示"还有 N 条排队"，并提供逐条 `remove`（`updateQueue`） |
| R12 | 房间草稿与宿主 composer 草稿 | 双写同一份 draft → 互相覆盖、丢字 | 房间草稿完全独立（§9.7），不回填、不读取 |
| R13 | 提问的前台等待会过期（`askTimed` / `attachWait`） | 用户慢慢选，问题已经作废 | 倒计时 + 过期禁用 + "可以让我重新问" |
| R14 | 静态 bundle 能否用 `host.call`（只有动态 client half 明确有） | 若可用，可省掉自建 HTTP 路由 | 先按 HTTP 路由实现（已验证路径），`host.call` 作为可选优化（附录 B.8） |
| R15 | 自建角色意味着 **M0 阶段没有现成素材** | 骨架跑起来但"不好看"，容易被误判为做不成 | 用**灰色剪影 + 足底锚点**跑通全部交互与布局（附录 D.7.2 的 P0 清单）；美术到位只换图集，逻辑一行不动 |

**开放问题（需产品决策）**

1. 房间页是否要在 header 的 View 里显示为**默认**视图（而不是用户手动切换）？
2. 分身小人是否需要显示子 agent 的**名字/任务**（会在小人身旁加小标签，信息密度上升）。
3. 空闲时角色是否允许"漫游出房间"（走出画面）？建议否，房间是舞台，出去就丢了。
4. 是否需要音效（打字声、铃铛、收工音）？建议 M3 且默认关。
5. 房间里的"中止"要不要二次确认（默认长按 400ms）？还是一击即停（更快，但更容易误触——尤其下拉菜单里）。
6. `approveMode: 'inRoom'` 要不要做成设置页里的**实验开关**，还是干脆不开放（永远深链）？
7. 房间输入默认 `queue` 还是 `steer`？（`steer` 更有"插话"感，但会打断当前步；`queue` 更安全）
8. 要不要支持"重发上一条 / 回到这一步之前"（涉及 fork 语义）？建议 v1 不做，M4 再说。

---

## 14. 里程碑与验收

| 里程碑 | 内容 | 验收标准 |
|---|---|---|
| **M0**（3–4 天） | Host fold + 迷你房间浮窗 + 14 phase + **角色用灰色剪影占位（骨架先立起来）** + 程序化道具/灯光 + 长轮询 | 手工跑一轮真实任务，14 个 phase 中至少 12 个能被真实事件触发并正确回落；`aborted` 不残留；工具秒回不抽搐 |
| **M1**（5–7 天） | 房间页 View（等距 SVG 场景 + 走位 + 6 工位 + 传送带）+ `inspectCall` 深链 + 通知卡 **+ 交互总线 `/action` + 对话（queue/steer）+ 急停拉杆** | 视觉复核通过；「房间里发一句话能真的发出去、并看到消息落地」；「中止后 `turn/end{aborted}` 落地且队列仍在」；失败有明确原因 |
| **M2**（5–7 天） | 分身术／求签字／回答问题／收工跑过来／压缩搬箱 五段演出 **+ 角色骨架与 P0 图集（附录 D.7.2）** **+ 回答提问（`userQuestions.answer`）+ 子 agent 打断** | 演出可被打断且不串戏；选项板能真实提交并驱动 `tool/result`；打断子 agent 后小人归位；审批深链能打开真实卡片 |
| **M3**（4 天） | 主题（白天/夜晚）、配置项全量、音效（可选）、命令、与 dsh-pet 共存 **+ 队列编辑 + 换模型 + 房间内签字（opt-in answerer）** | 配置非法有红色错误条；`inRoom` 只在白名单工具上认领且不自动放弃；locale 双语无硬编码 |
| **M4**（可选） | 轨迹回放（用 trajectory 事件驱动房间时间轴重演）、重发/回到某一步（fork） | 回放不产生副作用、不写 session |

**验收锚点（任何里程碑都必须满足）**

- 状态永远可从日志重建（重启/刷新后房间状态与真实一致，不重放旧的庆祝）。
- 任何一个"进入某相"都**必然**存在对应的退出条件（无死锁相）。
- 关掉动画（`reducedMotion`）后，房间仍然能读出现在在干什么。
- **每个房间内操作都能追溯到一条真实宿主 API 调用**，失败必有明确原因；没有"点了没反应"。
- 把 `interaction.enabled` 关掉后，房间退化成纯看板，行为与 v0.1 完全一致（不留半死的控件）。

---

## 15. 实现状态（v0.2 骨架 · 已落地）

M0 + M1 已实现并安装进本机 profile `desktop`（包名 `dsh-room`，源码 `dsh-room/`，`README.md` 有完整自查步骤）。

| 已落地 | 未落地（M2+） |
|---|---|
| Host 折叠（14 phase + `pendingAction` 语义在客户端）、`/state` 长轮询、`/action` 意图通道、`/anim` 素材 | 分身术演出、审批真签、队列编辑 UI、子 agent 打断 UI |
| 房间页 `conversation.view#room` + 迷你浮窗 `shell.overlay#room-mini` | 主题切换、音效、`/room` 命令、装修模式 |
| 等距 SVG 场景（10 工位 + 传送带 + 庭院 + 灯光层）、走位、4 工位标签 | 遮罩式家具分层遮挡（现在是单层绘制）、粒子特效 |
| 对话（queue/steer）、急停、回答问题（选择板）、审批深链 | 拖拽角色、多会话同屏、窄屏角色卡 |

**与设计的 4 处务实偏差**（都是"少一个未知 API"换来的稳定）：

1. **两条下行通道收敛成一条**：房间页也用 `/state` 长轮询，没走 `sessionProjections` + `wire.view`。少一个跨版本可能变的 API，代价是页面也吃轮询。
2. **交互统一走 `/action`**：没用宿主白给的 `inputActions` / `session.pendingInteraction`（它们的形状未公开），所以两个面行为完全一致。
3. **素材运行时借用（v0.2 占位，v0.3 废止）**：当时的做法是按 profile 目录找到已安装的 dsh-pet、借它的 webm。
   v0.3 改为自建骨架 + 图集（附录 D），运行时不再引用任何第三方美术。这条改动的**触发原因是功能性的**：
   成品视频没有挂点/朝向/接触帧，角色无法与房间物件交互。
4. **动画由 Host 算**：`anim.name/alt` 随状态下发，客户端不持有 phase→动画映射表，避免两份表走样。

**实测结论**：`read` 工具调用被正确映射成 `phase=working / station=shelf / anim=工作状态-忙碌点按`（见 `dsh-room/README.md` 的实测记录表）。

> **v0.3 与当前代码的差距（有意保留）**：现在跑着的骨架（`dsh-room/`）仍是 v0.2 形态 ——
> "借 dsh-pet 的 webm + SVG 立绘"。它的作用是**占位**：验证状态折叠、会话绑定、交互通道与布局，
> 不代表最终美术。v0.3 定稿后按三步替换，**都不动 Host 的 fold 与交互协议**（那两层已与美术无关）：
> ① 角色换成 §6 的骨架 + 挂点（`anim.name` 字段随之变成 `clip + dir`）；
> ② 场景换成 §5.2 的 tile 网格 + §5.3 的 Canvas 单遍绘制；
> ③ 素材服务从"运行时借 dsh-pet"改成"自带上图集"。

### 15.1 会话绑定（v0.3 增补，D10 的落地）

三块拼起来才成立：

1. **零视觉绑定器**：`SessionLink` 渲染 `null`，只有一句 `useEffect`，把 `props.sessionId` 写进模块级
   `currentSession`。**只挂一处**（`conversation.composer.dock`）：曾经同时挂它与会话头部 utilities，
   两处在切换瞬间各报一个 id，`currentSession` 被按帧翻转；每次翻转 → Host `active` 变 → `bump` →
   立刻唤醒长轮询 → **120–160 次/秒的自激热循环**（实测 `counter` 两分钟 5 → 18000）。止住它需要三道闸：
   **单点绑定 + 180ms 尾随去抖 + 轮询最小间隔 350ms**，并且 `?active=` 只在值变化的那一次请求里携带。
2. **本地优先切换**：两个面都先读 `currentSession`（房间页 `props.sessionId` 优先），所以切会话是
   **瞬时**的 —— 角色、站位、动作、气泡、排队条数全部换，不等网络。
3. **上行搭车而非另开通道**：`POST /action {op:'focus'}` 实测**没有到达** Host（探针 `lastAction` 恒为
   `null`），改在已验证可用的 `GET /state` 上带 `?active=<id>`；`currentSession` 变化时 `feed.poke()`
   打断在途长轮询立即重发。Host 侧 `setActive` **必须幂等**（只在会话变化/房间首现时 `bump`）。

> **部署约束（实操记下来）**：客户端半边的 `.js` 直接改了**不会**随页面热换 —— 那需要 `pnpm run dev:web`
> 重新构建，或走一次插件重装/重启应用换模块代。改完必须在插件管理里关→开，或重启应用，否则浏览器仍在跑
> 旧 bundle（本轮就因此让热循环多跑了几分钟）。
>
> **热循环的完整教训**（实测数据在 `dsh-room/README.md`）：客户端上报 `active` + Host `bump` 唤醒长轮询
> 这条链是有反馈回路的。三道闸缺一不可 —— 单点绑定、180ms 去抖、**订阅归零即停**（否则每次换模块代都留一条
> 孤儿轮询）；Host 侧再加**抖动闸**：2s 内交替超过 3 次就只采纳不通知。加闸前 `dbg.active=992/bump=996`、
> 137 次/秒；加闸后 <1 次/秒。

配套的三条一致性规则（都在 `dsh-room/README.md` 里）：切会话要重置幂等键（否则新会话的操作会被
当成重复请求静默吞掉）、清空草稿、重置通知卡基线（不重播上一轮的庆祝）。

---

## 附录 A · 已验证 API 清单（本机 profile 实测）

### A.1 Slot

| slot | kind / scope | 用途 |
|---|---|---|
| `conversation.view` | list / session | 房间页；注册参数 `{ id, order, label }`；owner props `inspectCall`、`viewRequest`、`openView`、`completeViewRequest`；标准 props 含 `sessionId`、`useConversation`、`useChat`、`useProjection`、`useSession`、`useSessionStatus`、`useTrajectory`、`inputActions` |
| `shell.overlay` | list / root | 迷你房间；注册参数 `{ id, order, label }` |
| `conversation.session.header.actions` / `utilities` | list / session | （可选）一个"打开工作间"按钮 |
| `settings.section` | list / root | （可选）房间设置页 |

`conversation.view` 现有占位：`chat(0)`、`trajectory(10)`、`context(20)`（`dsh-context`）、`fleet(30)`（`dsh-context`）—— 证明第三方注册可用。

### A.2 Host 事件（相关子集）

`session/event`、`api-session/status|added|removed|error`、`agent/status|error|created|disposed|assistant-stream`、`approval/request`（waterfall）、`user-questions/request`（waterfall）、`subagent/start|end`、`workflow/start|phase|log|agent-start|agent-end|end`、`tools/execute|pre-execute|post-execute|result`、`goal/changed`、`compaction/summary-error`、`connection/request`。

### A.3 工具名（当前 profile 的 `tool.call.toolview` keyDomain）

`ask_user_question` `bash` `cordis_define` `cordis_inspect_list` `cordis_inspect_query` `cordis_inspect_self` `cordis_run` `cordis_stop` `cordis_undefine` `create_goal` `edit` `get_goal` `glob` `grep` `interrupt_agent` `job_kill` `job_list` `job_output` `list_agents` `list_subagent_models` `lsp` `present` `ralph` `read` `read_image` `schedule_create` `schedule_delete` `schedule_list` `schedule_update` `send_message` `session_event_read` `session_event_search` `session_event_trace` `session_search` `session_trace` `skill` `spawn_teammate` `subagent` `team_task_*` `terminal_*` `todo_write` `update_goal` `wait_agent` `web_fetch` `web_search` `workflow` `write` + 运行时 MCP 工具（`mcp__*`）。

### A.4 Service

`sessionProjections`（`register` / `stateOf` / `onChanged` / `snapshot` / `cachedSnapshot` / `checkpoint`）、`webServer`（`register` / `registerUpgrade` / `tapIndex`）、`subagents`（`listChildren` / `listDescendants`）、`approval`（`request` / `setPolicy`）、`userQuestions`、`workingDirectory`、`tokenMeter`（`measure` → 用于 `contextPressure`）、`agents`（`get` / `roots` / `currentInitiator`）。

### A.5 主题令牌

`--dsw-alias-bg-base|bg-layer-1|bg-layer-2|bg-overlay|border-l1|border-l2|brand-primary|label-primary|label-secondary|state-error-primary|state-idle-primary|state-success-primary|state-warn-primary|specific-sidebar-fill`。

### A.6 Client 模块格式

`window.__ModuleLoader__.load({ id: '<包名>', factory(require){ … return { inject:['slots'], apply(ctx){ ctx.slots.inject(key, () => ctx.slots.register({name,id,order}, Component)) } } } })`；React 从模块表 `require('react')`。

### A.7 交互相关 API（v0.2 新增实测）

| 用途 | 精确签名 |
|---|---|
| 发消息 | `sessionController.prompt({ requestId, sessionId, mode: 'queue'\|'steer', content: PromptContentPart[], clientTimeZone? }) → { accepted: true }`（`@Remote('prompt')`，会先显式 resume 会话） |
| 中止 | `sessionController.cancel({ sessionId }) → { accepted: true }`（`@Remote('cancel')`；文档明确**不丢 pending inbox**） |
| 队列编辑 | `sessionController.updateQueue({ sessionId, itemId: MessageId, action: {kind:'edit',content} \| {kind:'remove'} \| {kind:'steer'} }) → { accepted: true }` |
| 换模型 | `sessionController.selectModel({ sessionId, provider, model }) → { selected }` |
| 回答提问 | `userQuestions.answer(agent, callId, { answers: [{ id, selected: string[], custom? }] }) → boolean`（`@Remote`；错误 `BAD_ANSWER` / `REPLY_QUEUED`） |
| 提问倒计时 | `userQuestions.attachWait(agent, callId, signal) → AsyncIterable<{ remainingMs }>`（`@Remote({mode:'stream'})`，关闭流即释放认领） |
| 打断子 agent | `subagents.interrupt(childSessionId, { kind: 'user', parentSessionId })`；配套 `listChildren` / `listDescendants` / `sendMessage` |
| 审批（读） | `approval/request` waterfall：`{ agent, toolName, callId?, reason?, displayReason?, signal? } → 'allowed-once' \| 'rejected' \| 'cancelled' \| 'unavailable'`；**一个部署只组一个终局 answerer** |
| 审批（写） | **没有**公开的"提交决策"方法 —— 决策只能由 answerer 在瀑布里返回。这就是 §9.5 那条边界的根因 |
| 待交互信号 | 房间页 props 的 `session.pendingInteraction`（与 `ComposerChainProps` 同源）是"要用户动手"的权威信号 |
| 输入动作 | `InputActions.captureInsertion()` 已确认存在；`inputActions` 由 `ctx.uiSession.provide()` 供给，**不是**可 `ctx.get` 的服务 |

### A.8 Client 侧可 `ctx.get` 的服务（决定交互入口）

只有 8 个：`layout` / `locale` / `sessions` / `slots` / `theme` / `timer` / `uiWorkspace` / `workspaces`。

**所以**：`prompt` / `cancel` / `answer` / `approve` 在 Client 侧**没有**直连服务 —— 必须走 ① 宿主白给的 props（房间页）或 ② 自建 `/action` 路由（浮窗）。这条直接决定了 §3.2 那条上行通道的存在。

---

## 附录 B · 待实现期确认项

1. `conversation.view` 的 `useConversation` / `useChat` / `useProjection` **精确返回形状**（决定房间页能否不依赖 Host 补齐 plan/question 细节）。
2. `SessionEventMap` 的完整类型尾部（实测被截断于 `session/end-seed`），需补齐 compaction / goal / todo / approval 相关事件的确切形状，用于 fold 的穷尽 switch。
3. `subagent/end` 的 `SubagentRunEndInfo` 字段（决定"归位"动画能否区分成功/失败/取消）。
4. `workflow/*` 各 info 的字段（`WorkflowRunInfo` / `WorkflowAgentInfo` / `WorkflowResultInfo`）。
5. `shell.overlay` 现有占位情况（避免和 dsh-pet 抢同一角落）。
6. `ctx.sessionProjections.register` 的 `ProjectionDefinition` 中 `wire.view` 的确切签名（是否 `(state) => view` 且需返回稳定引用）。
7. `dsh.client.inject` 对 `@deepseek-ai/dsh-client-connection` 以外依赖的必要性（浮窗做长轮询时是否需要）。
8. **静态安装的 client bundle 是否也能拿到 `host.call`**（文档只对"动态 client half"明确保证）。若能，`/action` 路由可换成 `host.call`，少一条自建通道；不能则维持现设计。
9. `inputActions` 的完整方法集（是否含 submit / stop，还是只有 `captureInsertion` 这类编辑器能力），以及 `useInput` 的 `InputState` 形状。
10. `session.pendingInteraction` 的类型 —— 能否区分 approval / question / plan-review（决定房间该弹纸还是弹问题板）。
11. `ask_user_question` 的 `tool/call.arguments.questions` 精确 JSON 形状（Host 侧 `AskUserQuestionItem` 已确认，落地的 `arguments` 字符串需按其同构解析）。
12. `PromptContentPart` 的确切联合（文本块是否就是 `{ type: 'text', text }`）。
13. Client 侧是否有可 `ctx.get('conversation')` 的队列服务（用于房间内队列编辑；否则全走 `/action`）。

---

## 附录 C · 命名规范

| 对象 | 规则 | 示例 |
|---|---|---|
| 插件/包 | kebab-case，前缀 `dsh-` | `dsh-room` |
| 路由前缀 | `/dsh-room-<4 位 hash>/` | `/dsh-room-9f2a/atlas/char@2x.png` |
| phase | 小写驼峰，见 §4.6 枚举 | `waitingApproval` |
| 工位 | 小写，见 §5.2 | `stove`、`counter` |
| clip | snake_case，动词在前 | `work_type`、`offer_paper`、`seal_clone` |
| 素材帧 | `<域>/<对象>_<状态>_<方向>_<帧号>` | `char/head_down_front_00`、`prop/paper_blank` |
| 图集 | `<域>@2x.png` + `<域>.atlas.json` | `char@2x.png` / `props.atlas.json` |
| 事件动画键 | `events.<phase>`，数组索引即档位（**只允许末尾追加**，中间插入会造成老配置错位） | `events.workStatus[3] = 等工作状态` |
| 道具 | `prop-<物>-<状态>.svg` | `prop-paper-waiting.svg` |
| CSS 类 | 前缀 `dsh-room-` | `.dsh-room-station--lit` |
| 交互 op | 小写 kebab，动词在前 | `prompt` `cancel` `answer` `approve` `queue` `interrupt-child` `select-model` |
| 幂等键 | uuid v4，带 `room-` 前缀（日志里一眼能认出来源） | `room-6f1c…` |
| 主题令牌 | 只用 `--dsw-alias-*` / `--dsw-specific-*` | `var(--dsw-alias-bg-layer-1)` |
