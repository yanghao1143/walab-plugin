# 发布与安装

这篇文档先带你完成**上传一个已经编译好的插件包**。当前版本支持离线 `.paasplugin` 包上传暂存；按 GitHub、Gitee 或 `.tgz` 地址安装等能力会在后续版本提供。

## 你现在可以做什么

| 目标 | 当前状态 |
|---|---|
| 上传已编译的 `.paasplugin` | 支持 |
| 查看包摘要和清单预览 | 支持 |
| 按 GitHub / Gitee 地址安装 | 后续版本 |
| 安装前预检和进度事件 | 后续版本 |
| 启停、升级、回滚、卸载 | 后续版本 |
| 作者注册、公钥信任和上架审核 | 后续版本 |

请先记住：接口返回 `state: uploaded` 只表示包已暂存，不表示插件已安装或可以运行。

## 第一次上传

### 1. 准备文件

先按[打包与签名](05-packaging-and-signing.md)生成一个 `.paasplugin` 文件，例如：

```text
example.paasplugin
```

### 2. 登录目标企业

使用 `https://developer.walab.ai` 的正式登录会话，并确认当前会话位于目标企业和目标站点。

上传需要企业管理员权限。普通用户会收到 `403 tenant_admin_required`。

### 3. 上传文件

向下面的正式接口发送 `multipart/form-data` 请求：

```http
POST /api-gateway/plugin-registry/v1/market/packages
Content-Type: multipart/form-data; boundary=<客户端生成>
```

multipart 字段只有两个：

| 字段 | 是否必需 | 填写内容 |
|---|---|---|
| `plug` | 是 | `.paasplugin` 文件，只能出现一次 |
| `visibility` | 否 | 当前填写 `private`；不填时默认也是 `private` |

浏览器或 HTTP 客户端会自动生成 `boundary`，不要手写企业身份头或签名上下文。租户和站点信息来自正式登录会话。

### 4. 检查返回结果

首次上传成功返回 HTTP `201`，响应会包含：

- `package_id`
- `state: uploaded`
- `visibility: private`
- `manifest_preview`
- `requested_permissions`
- `files`
- `package_digest`
- `signature`

私有包只属于当前企业。清单里的 `vendor_id` 不会把包转移给其他作者或企业。

同一个插件、同一个版本和同一个压缩包原始字节再次上传时返回 HTTP `200`，可以把它当作幂等重试成功。

如果同一个版本换了压缩包内容，会返回 HTTP `409`。即使只重新格式化清单、重新打包，压缩包摘要发生变化也会冲突。此时请递增版本号；重试原上传时继续使用第一次的原文件。

## 签名状态怎么看

响应里的 `signature.state` 目前可能是：

- `absent`：没有签名文件。
- `present_unverified`：有签名文件，但作者公钥尚未完成验证。

这两个状态都不能说明作者已经可信，也不能说明插件已经可以安装。签名格式见[打包与签名](05-packaging-and-signing.md)。

## 常见错误怎么处理

| HTTP | 错误 code | 先做什么 |
|---|---|---|
| 400 | `plug_missing`、`multipart_invalid` | 检查字段名、文件数量和 multipart 格式 |
| 400 | `visibility_unsupported` | 当前只使用 `private`，市场上架尚未开放 |
| 400 | `package_not_gzip_tar`、`member_path_unsafe`、`duplicate_member` | 按打包规范重新生成压缩包 |
| 400 | `manifest_invalid`、`wasm_missing`、`wasm_not_component` | 检查清单和 Wasm 是否为 Component |
| 400 | `signature_invalid`、`signature_content_mismatch` | 检查签名信封和成员摘要 |
| 403 | `tenant_admin_required` | 换用目标企业管理员的正式会话 |
| 409 | `package_version_conflict` | 使用原文件重试，或递增版本号 |
| 413 | `package_too_large`、`member_too_large`、`unpacked_too_large` | 缩小压缩包、单文件或解压后总大小 |
| 415 | `unsupported_media_type` | 使用 multipart，不要发送 JSON |
| 424 | `package_store_unavailable` | 保存 `request_id` 联系平台排查暂存依赖 |

排错时请提供请求路径、HTTP 状态、错误 code 和 `request_id`。不要提交密码、Cookie、会话令牌或其他凭据。

## 后续安装方式

后续版本计划支持：

- GitHub、Gitee 和可信 Git 主机地址；
- `.tgz` 发布包地址；
- Git 来源锁定完整 40 位 commit；
- 安装前预检、进度事件和取消；
- 安装后的启停、升级、回滚和卸载。

这些入口上线后会补充可直接复制的命令和示例。当前不要用上传暂存接口模拟安装成功，也不要绕过签名信任、权限确认或运行隔离。