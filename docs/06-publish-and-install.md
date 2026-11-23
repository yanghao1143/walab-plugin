# 发布与安装

这篇文档带你完成**第一次上传插件包**：从"确认自己有权限"到"看懂平台返回的每一个字段"。当前版本支持把 `.paasplugin` 包上传到你的企业暂存；按 GitHub / Gitee 地址安装、安装前预检、启停/升级/回滚/卸载会在后续版本提供。

> 对照说明：本章只描述**已上线**的能力。先记住一句：接口返回 `state: "uploaded"` 只表示包已经暂存成功，**不代表**插件已安装、已可信或已可运行。

## 1. 你需要准备什么

| 要件 | 说明 |
|---|---|
| 一个 `.paasplugin` 文件 | 按[打包与签名](05-packaging-and-signing.md)第 2 节生成 |
| 平台账号 | 在 `https://developer.walab.ai` 注册并**能登录** |
| 企业管理员的身份 | 上传要求企业管理员权限，见下 |
| 目标企业和站点 | 登录后进入你要发布插件的**那个企业**（顶部可以切换组织/企业） |

**怎么确认自己是企业管理员？** 用你的账号登录 `https://developer.walab.ai`，能看到该企业的管理功能入口即是；拿不准就先试一次上传——普通用户会收到 `403 tenant_admin_required`，此时换管理员账号，或找你的企业管理员。

> 插件市场的前端页面上线后，上传会是"把文件拖进页面"那么简单。当前版本先走接口，下面的每一步都给了完整可复制的请求。

## 2. 上传到底在做什么（30 秒白话）

你把文件用 `multipart/form-data` 格式（网页里"选择文件后点提交"时浏览器用的就是这种格式）发给平台的正式接口。**不需要**、也**不要**自己拼任何身份头或签名：你的企业、站点、身份都来自你在 `developer.walab.ai` 的登录会话，网关会自动识别。你要做的只是"带着会话发文件"。

## 3. 发起上传

接口（正式入口，经平台网关）：

```http
POST https://developer.walab.ai/api-gateway/plugin-registry/v1/market/packages
Content-Type: multipart/form-data; boundary=<客户端自动生成>
```

表单只有两个字段：

| 字段 | 是否必需 | 填什么 |
|---|---|---|
| `plug` | 必需 | 你的 `.paasplugin` 文件，**只能出现一次** |
| `visibility` | 可选 | 固定填 `private`（不填默认也是 `private`；市场上架审核开放前，`market` 会被拒绝） |

### 3.1 用 curl 的完整写法

macOS / Linux：

```sh
curl -X POST "https://developer.walab.ai/api-gateway/plugin-registry/v1/market/packages" \
  -b "会话Cookie=<你登录后的会话>" \
  -F "plug=@example.paasplugin" \
  -F "visibility=private"
```

Windows PowerShell（注意 PowerShell 5.1 的 `curl` 是 `Invoke-WebRequest` 的别名，先换用 `curl.exe`）：

```powershell
curl.exe -X POST "https://developer.walab.ai/api-gateway/plugin-registry/v1/market/packages" `
  -b "会话Cookie=<你登录后的会话>" `
  -F "plug=@example.paasplugin" `
  -F "visibility=private"
```

关于"会话"：就是你登录 `developer.walab.ai` 之后浏览器里持有的那个会话。它等同你的登录凭证——**只自己用，不要发给任何人、不要截图、不要提交进代码仓库**。如果你不知道怎么带会话：等插件市场页面上线后，直接在页面拖拽上传即可，浏览器会自动处理一切。

## 4. 看懂返回：逐字段解释

第一次上传成功返回 HTTP **201**，响应是一个 JSON，每个字段的意思：

| 字段 | 是什么 | 你拿它做什么 |
|---|---|---|
| `package_id` | 形如 `plg_my_first_plugin@0.1.0`，插件 ID + 版本 | 记下来，后续安装/管理都用它指代这个包 |
| `state` | 固定 `uploaded`：已暂存 | 提醒自己——只是暂存，不是安装 |
| `visibility` | `private`：只有你这个企业能看见 | 上架审核开放前都是 private |
| `package_digest` | 整个压缩包的 SHA-256 指纹 | 和你本地算的对照，一致说明传输无损 |
| `content_digest` | 包内容的指纹（签名绑定用的那个） | 核对签名教程第 5.4 步脚本算出的值 |
| `manifest_preview` | 平台读到的清单摘要：插件 ID、类型、版本、显示名、运行类型 | 逐项和你写的清单对一遍，防手滑 |
| `requested_permissions` | 清单里声明的权限（`required` + `optional`） | 将来权限确认页展示的就是它 |
| `files` | 包内每个文件的路径、大小、指纹（`signature.json` 不列出） | 逐个核对，确认包里没有多余东西 |
| `signature` | 签名状态，见下一节 | 决定要不要回头补第 5 章的签名流程 |

**同一个文件再传一次**：返回 HTTP **200**，含义是"幂等重试成功"——平台认出这是同一个包（同一插件、同一版本、同一字节），什么都没改。这不是错误。

**同一版本换了内容**：返回 HTTP **409**（`package_version_conflict`）。哪怕你只是重新格式化了清单或重新 `tar` 了一次，只要整包字节变了就算"换了内容"。处理办法二选一：用**第一次生成的那个文件**重试；或递增 `version` 上传新版本。

## 5. `signature.state` 三个值分别代表什么

| 值 | 含义 | 你要做什么 |
|---|---|---|
| `absent` | 包里没有 `signature.json` | 当前版本允许；想成为可信作者就回第 5 章补签名 |
| `present_unverified` | 有格式正确的签名，但平台还没有可验证的作者公钥 | 正常现象。作者注册/公钥登记开放前，所有签名都是这个状态 |
| `verified` | 签名与平台登记的作者公钥对上了 | 作者体系上线后才会出现 |

`verified` 之外，带签名的包还可能给出 `present_unverified` 加一个机器可读原因 `signature.reason`，逐条对照：

| `signature.reason` | 含义 | 你要做什么 |
|---|---|---|
| `signing_key_unknown` | 你企业的名下查不到这把 `key_id` 对应的已登记公钥 | 公钥登记开放后先登记；没登记过的签名一律是这个原因 |
| `signing_key_author_unbound` | 清单里没有写 `vendor_id`，签名不知道该对哪个作者 | 在清单里补 `vendor_id`（你的作者身份），重新打包签名 |
| `signing_key_inactive` | 这把公钥已登记但已被停用 | 换一把有效的 `key_id` 重新签名，或联系平台恢复 |
| `signing_key_not_yet_valid` | 公钥还没到生效时间（`valid_from` 在未来） | 等到生效时间，或登记一把当前已生效的 key |
| `signing_key_expired` | 公钥已过有效期（`valid_until` 已过） | 登记新 key 重新签名；过期公钥不会再判 verified |
| `signing_key_window_invalid` | 登记的有效期数据本身不合法 | 联系平台核对这条公钥登记记录 |
| `signature_invalid` | 找到了可信公钥，但包的字节和签名对不上 | **视为被改动**：用定稿后的原包重新签名打包，别直接重传改过的包 |
| `author_key_projection_unavailable` | 平台的作者公钥读取暂不可用 | 不是包的问题，稍等重试；持续出现就带 `request_id` 找平台 |

这三个值**都不能**说明"插件已安装/已可运行"。安装与运行是后续版本的能力。也不要用无签名包冒充可信包：`vendor_id` 只是包内署名，不转移所有权，发布者永远是上传它的企业。

## 6. 出错怎么办：逐条自查表

所有错误响应都长这样：HTTP 状态码 + JSON 里的 `code`（机器码）和 `details`（人话）+ `request_id`（这次请求的编号）。先对表自查：

| HTTP | `code` | 含义与自查 |
|---|---|---|
| 400 | `plug_missing` / `multipart_invalid` | 表单里没有文件或格式不对：字段名必须是 `plug`，只出现一次，用 `-F "plug=@文件"` 的写法 |
| 400 | `visibility_unsupported` | `visibility` 只能是 `private`；上架审核开放前不支持 `market` |
| 400 | `package_not_gzip_tar` | 这不是 gzip 的 tar 包：确认扩展名背后真的是 `.tar.gz`（打包命令见第 5 章第 2 节） |
| 400 | `member_path_unsafe` / `duplicate_member` | 包里有绝对路径、`..`、链接、目录或重复成员：重新打包时**只写文件名** |
| 400 | `package_kind_unsupported` | 包里有 `ui/` 文件，或清单 `plugin_type`/`runtime.kind` 不是后端 Wasm 组件 |
| 400 | `manifest_invalid` | 清单不合规矩：对照第 5 章 1.2 的字段表逐项检查（最常见：多余逗号、中文引号、自造字段、`plugin_id` 大写） |
| 400 | `wasm_missing` / `wasm_not_component` | 没有 `plugin.wasm`，或它不是 Wasm 组件：用第 5 章 1.3 的方法看第 8 个字节 |
| 400 | `signature_invalid` / `signature_content_mismatch` | 签名信封格式错误或和内容对不上：重跑签名脚本、确认用的是定稿后的包 |
| 403 | `tenant_admin_required` | 当前身份不是该企业的管理员：换管理员账号或找企业管理员 |
| 409 | `package_version_conflict` | 同一版本已存在不同内容：用原文件重试，或递增版本号 |
| 413 | `package_too_large` / `member_too_large` / `unpacked_too_large` | 超出第 5 章第 3 节的大小限制：裁剪资源或拆分功能 |
| 415 | `unsupported_media_type` | 请求体不是 multipart：不要手写 JSON 上传，用 `-F` |
| 424 | `package_store_unavailable` | 平台暂存依赖暂不可用：稍等几分钟重试；仍失败就带着 `request_id` 找平台 |

**自查两轮还没头绪时再求助**，求助时提供四样东西：请求的接口路径、HTTP 状态码、`code`、`request_id`。**永远不要**把会话 Cookie、密码或令牌发给任何人（包括平台工作人员——他们不需要这些就能凭 `request_id` 查到请求）。

## 7. 接下来会发生什么

按平台后续版本的计划，本章节之后会陆续补上：

- 按 GitHub / Gitee / `.tgz` 地址安装（锁定完整 40 位 commit）；
- 安装前预检（把版本兼容、权限、签名结果先展示给企业确认）；
- 安装进度事件、取消；
- 启用、停用、配置、升级、回滚、卸载；
- 作者注册、公钥登记、上架审核。

这些能力上线时，本章和[打包与签名](05-packaging-and-signing.md)会同步更新出对应的完整操作步骤。当前请只用上传暂存接口；不要绕过签名信任、权限确认或运行隔离去"模拟"安装成功。
