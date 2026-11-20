# 打包与签名

这篇文档解决两个问题：**怎样生成一个可以上传的 `.paasplugin` 文件**，以及**怎样理解包里的摘要和签名**。

如果你只是想先把插件上传到市场，先看[发布与安装](06-publish-and-install.md)里的“第一次上传”流程。签名是可信作者和后续上架审核的一部分，可以在包能上传后再接入。

## 先记住这件事

上传成功只代表平台把包暂存下来，状态是 `uploaded`。它不代表插件已经安装、签名已经可信，或者插件已经可以运行。

当前版本只接收后端 Wasm 插件包，不接收前端 UI 包。按地址安装、安装前预检、安装进度、启停、升级、回滚、卸载和作者上架审核会在后续版本提供。

## 1. 准备两个文件

一个最小的后端插件包包含：

```text
plugin-manifest.json
plugin.wasm
```

- `plugin-manifest.json` 使用 `paas.plugin-manifest.v1`。
- `plugin_type` 必须是 `backend-wasm-plugin`。
- `runtime.kind` 必须是 `wasm-component`。
- `plugin.wasm` 必须是 Wasm Component Model 组件，普通 Wasm core module 会被拒绝。

签名文件是可选的：

```text
signature.json
```

当前阶段可以先不放签名文件；后文会说明它的格式和当前状态。

不要把下面这些内容放进当前上传包：

```text
ui/asset-manifest.json
前端资源
```

这些路径为后续 UI 包格式预留，当前上传入口会拒绝它们。

## 2. 生成 `.paasplugin` 文件

在 `plugin-manifest.json` 和 `plugin.wasm` 所在目录执行：

```sh
tar -czf example.paasplugin plugin-manifest.json plugin.wasm
sha256sum example.paasplugin
```

生成的 `example.paasplugin` 就是要上传的文件。`sha256sum` 输出的是整个压缩包的 SHA-256 摘要，上传后可以和平台返回的 `package_digest` 对照。

同一个插件版本重试上传时，必须继续使用第一次生成的压缩包。重新执行 `tar` 可能改变文件时间、压缩头或成员顺序，导致摘要变化；如果内容确实变了，请递增版本号。

## 3. 包必须满足的限制

| 项目 | 限制 |
|---|---:|
| 压缩包大小 | 不超过 32 MiB |
| 单个文件大小 | 不超过 16 MiB |
| 解压后总大小 | 不超过 48 MiB |
| 文件数量 | 不超过 512 个 |
| `plugin-manifest.json` | 不超过 256 KiB |
| `signature.json` | 不超过 16 KiB |

包内成员必须使用安全的相对路径。绝对路径、`..`、重复成员、符号链接、硬链接、目录和未定义成员都会被拒绝。

## 4. 三种摘要分别是什么

| 字段 | 对什么计算 | 用来做什么 |
|---|---|---|
| `package_digest` | 整个 `.paasplugin` 压缩包的原始字节 | 暂存、回读和判断同版本是否上传了不同内容 |
| `artifact_digest` | `plugin.wasm` 文件字节 | 单独标识后端 Wasm 组件 |
| `content_digest` | 包内成员的规范化摘要列表 | 绑定签名和包内容 |

`package_digest` 和 `artifact_digest` 通常不同：前者覆盖整个压缩包，后者只覆盖 `plugin.wasm`。

## 5. 签名文件怎么工作

除 `signature.json` 本身以外，对每个文件计算 SHA-256，按成员路径排序，然后拼接成下面的文本。摘要使用小写十六进制，每行以换行符结束：

```text
<64 位文件 sha256> <成员路径>\n
```

再对这段文本计算一次 SHA-256，得到 `content_digest`。签名信封包含：

- `alg: ed25519`
- `key_id`
- `content_digest: sha256:<64 位摘要>`
- base64 编码的 64 字节 `signature`

当前上传入口只检查签名信封的结构和内容摘要，返回 `absent` 或 `present_unverified`：

- `absent`：包里没有签名文件。
- `present_unverified`：有签名文件，但还没有完成作者公钥验证。

`present_unverified` 不等于签名可信，也不等于作者已认证、签名已通过或插件可以安装。作者注册、公钥登记和真正的 Ed25519 验证会在后续步骤上线。

下一步请看[发布与安装](06-publish-and-install.md)。