# WALAB 插件开发者中心

这里是给**外部插件开发者**用的开发文档、SDK、示例和联调指南。你可以在自己的 GitHub / Gitee 仓库里开发插件，发布后通过 WALAB 插件市场安装到企业里：
- 在插件市场填仓库地址，由平台服务端拉取、校验、构建、安装；
- 或者把打好的插件包直接拖进插件市场安装。

> 状态：第一版目录。各章节随平台后端能力上线逐步补齐，每一章都写明对应的接口和版本。

## 目录
| 章节 | 内容 |
|---|---|
| [docs/01-overview.md](docs/01-overview.md) | 插件是什么、能做什么、运行在哪里（wasm 组件 + 可选前端资源）、生命周期 |
| [docs/02-quickstart.md](docs/02-quickstart.md) | 十分钟写出第一个插件：建仓、写代码、本地跑通、打包 |
| [docs/03-manifest.md](docs/03-manifest.md) | 插件清单 `manifest.json`（`paas.plugin-manifest.v1`）全部字段 |
| [docs/04-platform-capabilities.md](docs/04-platform-capabilities.md) | 怎么用平台能力：数据（数据库）、缓存、消息、对象存储、模型调用、身份与租户上下文。插件**不直连**任何中间件，一律经平台网关和数据网关，并按清单里声明的权限放行 |
| [docs/05-packaging-and-signing.md](docs/05-packaging-and-signing.md) | 打包格式（清单 + wasm + 前端资源）、签名、版本号与兼容范围 |
| [docs/06-publish-and-install.md](docs/06-publish-and-install.md) | 在 GitHub / Gitee 发版（锁定提交或 Release 资产）；在插件市场按地址安装或拖拽安装；升级、回滚、卸载 |
| [docs/07-install-requests.md](docs/07-install-requests.md) | 安装请求：从提交到装好（发起、进度、SSE、取消、制品一致性） |
| [docs/08-debugging.md](docs/08-debugging.md) | 联调：本地模拟宿主（不连平台也能跑通全部接口）＋ 开发沙箱企业（真实安装、真实调用、看日志） |
| [docs/09-security-and-review.md](docs/09-security-and-review.md) | 安全模型：租户隔离、权限确认、签名与可信来源、上架审核 |
| [docs/10-faq.md](docs/10-faq.md) | 常见错误码和排查 |

## 目录结构
- `docs/`：开发文档
- `sdk/`：Rust 插件 SDK（从平台 SDK 整理出公开版本）
- `examples/`：可以直接编译运行的示例插件
