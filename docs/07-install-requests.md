# 安装请求：从提交到装好

这一章讲插件「决定安装」到「真正跑起来」之间发生了什么，以及你作为企业管理员能做什么、会看到什么。

> **读者**：在企业里点「安装」、盯进度、或需要中途取消的人。
> **前置**：插件包已经上传暂存（第 5 章）或来自一个锁定的地址（GitHub / Gitee / `.tgz`）。

## 1. 一个「安装请求」是什么

你点「安装」时，平台**不会**立刻把插件装好。它会先登记一条**安装请求**（install request），马上回给你一个 `request_id`，然后在后台一步步推进。你可以随时查它到哪一步、看实时进度、或在还没开始时取消。

这就像寄快递：你拿到单号（`request_id`），包裹（安装）由平台分阶段运送，你凭单号查物流（进度），在还没揽收前可以撤回（取消）。

**为什么不是同步装完？** 因为安装可能要做拉取代码、构建、校验等耗时操作，让你的浏览器一直转圈等既慢又不可靠。拆成「先登记、再推进、可查询」后，即使页面关掉，安装也照常进行，你回来凭 `request_id` 接着看。

## 2. 发起安装：你要提供什么

调 `POST /api-gateway/plugin-registry/v1/market/install-requests`（或在市场页面点「安装」，页面替你调）。要准备三样：

| 字段 | 必填 | 说明 |
|---|---|---|
| 来源 `source` 或 `package_id` | 二选一 | **地址安装**：`{type:"git"\|"tgz", url, commit, path}`，`commit` 必须是**完整 40 位**（锁死版本，防止地址内容被悄悄换掉）；**上传包安装**：直接给上传时拿到的 `package_id`（形如 `plg_demo@1.0.0`） |
| `enabled` | 可选，默认 `false` | 装好后是否立即启用 |
| `accept_permissions` | **必填** | 必须为 `true`：表示你看过清单申请的权限并明确同意。这是权限确认门，平台不接受「默认勾选」 |

地址安装的 `url` 只接受 https，且主机必须在允许列表里（GitHub、Gitee，或平台另行配置的信任主机），不带账号密码、不指向 IP 地址。

成功返回 HTTP **201** 和一个 JSON，里面最重要的是 `request_id`（形如 `ireq_...`）和初始 `status: "pending"`（已受理、排队中）。**记下 `request_id`**，后续查询、看进度、取消都用它。

**重复提交**：如果你用完全相同的参数再发一次，平台认出是同一个请求，返回 HTTP **200** 和同一个 `request_id`，不会重复安装。如果参数几乎一样但内容变了，会返回 **409**（冲突）。

## 3. 安装会经过哪些状态

一条安装请求的 `status` 会按顺序往前走：

```
pending → queued → inspecting → fetching → building → installing → installed
   (已受理)  (排队)   (预检)      (拉取)     (构建)     (安装)     (装好)
                                                              ↘ failed (失败)
   任意未开始状态 --取消--> cancelled (已取消)
```

| 状态 | 含义 | 你能做什么 |
|---|---|---|
| `pending` / `queued` | 已受理，还没真正开始 | 可以**取消** |
| `inspecting` | 预检：版本兼容、权限、签名、依赖 | 等待 |
| `fetching` | 按地址拉取代码/包 | 等待（已开始，**不能取消**） |
| `building` | 在隔离流水线里构建 Wasm 组件 | 等待 |
| `installing` | 写入安装状态、启用（若 `enabled`） | 等待 |
| `installed` | 装好 | 终态 |
| `failed` | 某一步失败 | 终态；看 `failure_code` / `failure_message` 定位，修正后可重新发起 |
| `cancelled` | 已取消 | 终态 |

查当前状态：`GET /api-gateway/plugin-registry/v1/market/install-requests/{request_id}`。

## 4. 看实时进度（SSE 事件流）

除了轮询状态，还能订阅实时事件流：`GET /api-gateway/plugin-registry/v1/market/install-requests/{request_id}/events`，返回 `text/event-stream`（SSE）。浏览器 `EventSource` 或 `curl -N` 都能接。

事件有两类：

| 事件 | 内容 | 什么时候出现 |
|---|---|---|
| `install-log` | `{phase, message, ts_unix_ms}`：某一阶段的一句话日志 | 推进过程持续产生 |
| `install-state` | `{status, phase, ts_unix_ms}`：状态发生变化的节点 | 每次 `status` 跳转 |

每个事件带一个递增序号 `seq`（SSE 的 `id` 字段）。**断线续传**：网络断了重连时带上 `Last-Event-ID: <上次收到的 seq>`（或查询参数 `after_seq`），平台只发这之后的新事件，不会重发一堆。

流不会无限挂着：到达终态（`installed`/`failed`/`cancelled`）或超时后平台会主动结束连接，客户端按 SSE 惯例自动重连即可（重连时带上最后的 `seq`）。

## 5. 取消安装

只能在**还没开始**时取消：`POST /api-gateway/plugin-registry/v1/market/install-requests/{request_id}/cancel`。

- 请求还在 `pending` / `queued`：取消成功，返回 200，`status` 变 `cancelled`。
- 已经进入 `fetching` 或更后：返回 **409** `install_request_not_cancellable`——覆水难收，安装会继续走完（之后你可以走卸载流程）。

取消是幂等的：对一个已 `cancelled` 的请求再取消，仍返回 200，不报错。

## 6. 制品可信与一致性（为什么「装好的」就是「你审过的」）

平台在你确认安装到真正启用之间，用**内容寻址存储（CAS）**保证你看到的和最终运行的是同一份字节：

- 上传/拉取到的制品按 **SHA-256 摘要**寻址，摘要不一致就算同名校验也不过；
- 真正落「可运行制品」那一步是**原子**的：平台会同时核对发布绑定、版本、提交这三处的最新修订号，三者都与当前权威记录完全一致才写入；任何一处在你操作期间被别人改动，整步拒绝（返回 409 `source_revision_changed`），不会写出半成品；
- 写入后会**读回**权威字节重新校验摘要和长度，双确认才算成功。

所以你不用担心「我审的是 A，结果线上跑的是 B」。这套一致性保证目前随平台版本逐步启用；在它覆盖的路径之外，安装请求会明确失败（503），而不是带病继续。

## 7. 出错怎么办

| HTTP | `code` | 含义与处理 |
|---|---|---|
| 400 | `install_source_commit_unpinned` | 地址没锁完整 40 位 commit：补上 `#<完整 commit>` 重新发起 |
| 400 | `install_source_url_invalid` | 地址不是允许的 https 信任主机，或带了账号密码 / IP：换合规地址 |
| 400 | `accept_permissions_required` | 没显式确认权限：把 `accept_permissions` 设为 `true` 再发 |
| 404 | `install_request_not_found` | `request_id` 不存在，或不属于你当前企业（跨企业一律 404） |
| 409 | `install_request_conflict` | 同一请求标识但内容不同：确认参数后用原 `request_id` 查 |
| 409 | `install_request_not_cancellable` | 已开始，取消不了：让它走完，之后走卸载 |
| 503 | `install_request_store_unavailable` | 平台的安装台账依赖暂不可用：稍等重试；持续出现带 `request_id` 找平台 |

排查时给平台提供：接口路径、HTTP 状态码、`code`、`request_id`（响应里的请求编号）。**不要**提供会话、密码或令牌。

## 8. 之后会发生什么

安装请求装好（`installed`）只是「可用」的起点。后续版本会补上：启用/停用、按清单 config schema 渲染的配置表单、升级（只能升更高版本、失败自动回退）、回滚、卸载（连同菜单、接口、配置、制品引用一起清理）。这些能力上线时本章会同步补完整操作步骤。
