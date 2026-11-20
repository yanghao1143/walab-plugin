# 打包与签名

这篇文档解决两个问题：**怎样从零做出一个可以上传的 `.paasplugin` 插件包**，以及**怎样给包签名**。

写给你的前提只有一个：会打开命令行、会复制粘贴。每个概念第一次出现时都会先用大白话解释；每条命令都给出可直接复制的完整版本。如果某一节暂时用不上（比如签名），可以先跳过，先完成第一次上传，再回来补。

> 对照说明：本章描述的是**当前已经上线**的平台行为。按地址安装、安装前预检、启停/升级/回滚/卸载、作者注册和上架审核会在后续版本提供，上线后本章会同步更新，并在每一节写明对应的接口和版本。

## 0. 先弄懂四个概念

**插件包（`.paasplugin`）是什么？**
它就是一个改了扩展名的 `.tar.gz` 压缩包（习惯叫 tar 包），里面装着你的插件的一切：清单、程序文件、可选的签名。平台不挑扩展名背后的东西，只挑里面的内容是否合规。

**清单（manifest）是什么？**
`plugin-manifest.json`，一张"插件身份证 + 说明书"：插件叫什么、什么版本、需要哪些权限、入口在哪。平台用它来认识你的插件，所以格式要求很严格。

**Wasm 组件（`plugin.wasm`）是什么？**
你的插件的程序本体，一个编译好的 WebAssembly 文件。注意平台要的是 **Wasm 组件（Component Model）**，不是普通的 Wasm 模块——两者文件开头不同，普通的会被拒绝（后面教你一眼分辨）。

**摘要（digest）是什么？**
对一段内容算出的"指纹"（SHA-256），内容哪怕变一个字节，指纹就完全不同。平台用摘要判断"你这次上传的包和上次是不是同一个东西"。

## 1. 准备文件：一个最小的包长什么样

一个最小的后端插件包包含两个文件，放在同一个文件夹里：

```text
plugin-manifest.json   插件的"身份证 + 说明书"
plugin.wasm            插件程序本体（Wasm 组件）
```

`plugin.wasm` 来自第 2 章《快速开始》的构建步骤（那一章上线后会有从写代码到产出的完整流程）。本章先把它当作一个**已经存在**的文件。

### 1.1 清单：完整的最小示例

把下面整段保存为 `plugin-manifest.json`（用任何文本编辑器，不要用 Word，推荐 VS Code 或记事本）：

```json
{
  "manifest_schema_version": "paas.plugin-manifest.v1",
  "plugin_id": "plg_my_first_plugin",
  "plugin_type": "backend-wasm-plugin",
  "version": "0.1.0",
  "display_name": "我的第一个插件",
  "vendor_id": "my-company",
  "summary": "一句话介绍这个插件做什么",
  "runtime": {
    "kind": "wasm-component",
    "entry": "paas:plugin@0.1.0#plugin.run"
  },
  "wit_package": "paas:plugin@0.1.0",
  "wit_world": "plugin",
  "capabilities": {
    "required": ["platform.log"],
    "optional": []
  }
}
```

### 1.2 每个字段是什么意思、有什么规矩

| 字段 | 填什么 | 规矩（违反会被拒绝） |
|---|---|---|
| `manifest_schema_version` | 固定值 `paas.plugin-manifest.v1` | 必须一字不差 |
| `plugin_id` | 插件的唯一 ID | 必须以 `plg_` 开头，后面只能用**小写字母、数字、下划线**，最长 64 个字符。同一个企业里它必须唯一，装了以后改不得 |
| `plugin_type` | 当前固定填 `backend-wasm-plugin` | 后端 Wasm 插件目前是唯一可上传的类型 |
| `version` | 你的版本号 | 必须是三段数字，如 `0.1.0`；可以带预发布后缀如 `1.2.3-beta.1`；最长 64 字符；**不支持** `+build` 这类构建元数据 |
| `display_name` | 给人看的名字 | 1～256 个字符 |
| `vendor_id` | 你（作者/厂商）的标识 | 1～256 个字符。它只是"包里声明的作者署名"，**不会**把包的所有权转给别人；上传的企业才是发布者 |
| `summary` | 一句话介绍 | 可选 |
| `runtime.kind` | 固定值 `wasm-component` | 平台只运行 Wasm 组件 |
| `runtime.entry` | 插件入口 | 形如 `paas:plugin@0.1.0#plugin.run`，和 `wit_package`/`wit_world` 对应 |
| `wit_package` / `wit_world` | 插件接口包与世界名 | 与第 2 章构建时使用的保持一致 |
| `capabilities.required` | 必须拥有的权限列表 | 格式见示例；可用能力清单在第 4 章逐个说明（随能力上线补齐） |
| `capabilities.optional` | 可选权限列表 | 可以为空数组 `[]` |

两个容易踩的坑：

- JSON 里**不要出现多余的逗号**、不要把双引号写成中文引号；除菜单字段外**不允许 `null`**。
- 清单里不许出现没有列在上表之外的自造字段，写了会被拒绝（平台按封闭清单校验）。

### 1.3 怎么确认 `plugin.wasm` 是"组件"而不是"普通模块"

用文件头 8 个字节判断。组件的第 8 个字节是 `01`，普通模块是 `00`。

macOS / Linux：

```sh
head -c 8 plugin.wasm | xxd
```

Windows PowerShell：

```powershell
Format-Hex -Path plugin.wasm -Count 8
```

正确的话开头 8 个字节是：

```text
00 61 73 6d 0d 00 01 00
```

如果第 8 个字节（最后的 `01`）是 `00`，说明你构建成了普通 Wasm 模块，请回到第 2 章按"组件"目标重新构建，上传这个文件会被拒绝（错误码 `wasm_not_component`）。

## 2. 打包：三条命令

把 `plugin-manifest.json` 和 `plugin.wasm` 放在同一个文件夹，在**那个文件夹里**执行：

macOS / Linux：

```sh
tar -czf example.paasplugin plugin-manifest.json plugin.wasm
sha256sum example.paasplugin
```

Windows PowerShell：

```powershell
tar -czf example.paasplugin plugin-manifest.json plugin.wasm
Get-FileHash example.paasplugin -Algorithm SHA256
```

> Windows 10/11 自带 `tar`，直接用即可；`sha256sum` 在 Git Bash 里可用。

得到的 `example.paasplugin` 就是成品，那一长串十六进制是它的 `package_digest`（整包指纹），上传后可以和平台返回的值对照。

**重要：同一个版本重试上传时，必须继续用第一次生成的这个文件。** 重新执行 `tar` 可能因为文件时间、压缩参数不同产出不同字节、指纹就变了，平台会当成"同一版本换了内容"拒绝（HTTP 409）。如果内容确实要改，请递增 `version` 再打包。

## 3. 包的硬性限制（上传前自己先查一遍）

| 项目 | 限制 | 超了会收到 |
|---|---:|---|
| 整个压缩包大小 | ≤ 32 MiB | `package_too_large` |
| 单个文件大小 | ≤ 16 MiB | `member_too_large` |
| 解压后所有文件加起来 | ≤ 48 MiB | `unpacked_too_large` |
| 文件个数 | ≤ 512 个 | 见错误表 |
| `plugin-manifest.json` 大小 | ≤ 256 KiB | `manifest_invalid` |
| `signature.json` 大小 | ≤ 16 KiB | 见错误表 |

查大小：macOS/Linux 用 `ls -l`，Windows 用 `Get-Item <文件> | Select-Object Length`（单位字节，32 MiB = 33554432）。

包内路径的规矩：只允许**普通的相对路径文件**。绝对路径（`/tmp/...`）、`..`、重复的同名成员、符号链接、硬链接、目录条目都会被拒绝。所以打包命令里**只写文件名、不要写文件夹**，就天然合规。

当前上传入口只接收后端插件包：不要放 `ui/` 开头的任何文件（包括 `ui/asset-manifest.json`），那是将来前端资源包的预留位，现在放了会收到 `package_kind_unsupported`。

## 4. 三种指纹分别是什么、拿来干什么

| 字段 | 对什么计算 | 用来做什么 |
|---|---|---|
| `package_digest` | 整个 `.paasplugin` 文件的原始字节 | 暂存、回读、判断"同一版本是否传了不同内容" |
| `artifact_digest` | 只对 `plugin.wasm` 的字节 | 单独标识后端组件，运行时校验的就是它 |
| `content_digest` | 包内（除 `signature.json` 外）每个文件的指纹清单，再整体算一次指纹 | 把签名和包内容绑定在一起 |

`package_digest` 和 `artifact_digest` 通常**不相等**：前者覆盖整个压缩包（含清单），后者只覆盖 `plugin.wasm`。平台响应里会同时给你这两个值。

`content_digest` 的计算规则（第 5 节的脚本会替你做，这里写给想核对的人）：

1. 对包里**除 `signature.json` 外**的每个文件算 SHA-256，得到小写十六进制；
2. 每个文件写成一行：`<指纹> <文件路径>`，注意中间**恰好一个空格**，行尾**必须有换行符**；
3. 所有行按**文件路径**排序后拼接成一段文本；
4. 对这段文本再算一次 SHA-256，前缀 `sha256:`，就是 `content_digest`。

## 5. 签名：从生成密钥到产出 signature.json

> 这一节可以整个跳过：当前版本**无签名的包也能上传**，响应里显示 `absent`。签名是"可信作者"体系的一部分（作者注册、公钥登记、上架审核在后续版本上线），想先把流程跑通的可以等平台开放作者注册后再来补签。

### 5.1 白话解释

Ed25519 是一种"一对钥匙"的数字签名方案：

- **私钥**：只有你有的一个文件，用来签名，**绝不给任何人**（包括平台客服）；
- **公钥**：从私钥算出来的一串字符，可以公开。平台登记你的公钥后，用它验证"这个包确实是你私钥签过的、而且签完没被改过"。

签名覆盖的是 `content_digest`——也就是包里所有内容。改了包里任何一个字节，签名就失效。所以**签名要在包内容定稿后做**。

### 5.2 准备工具（一次性）

装 Python 3.9+（官网下载，安装时勾选 "Add to PATH"），然后装一个库：

```sh
pip install cryptography
```

### 5.3 一个脚本搞定全部

把下面整段保存为 `paasplugin-sign.py`，和你的包放同一个文件夹：

```python
#!/usr/bin/env python3
"""为 .paasplugin 包生成作者密钥对和 signature.json。

用法:
    python3 paasplugin-sign.py keygen
    python3 paasplugin-sign.py sign example.paasplugin author-key-1
"""
import base64
import hashlib
import json
import sys
import tarfile

from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey

KEY_FILE = "author-private-key.pem"
PUB_FILE = "author-public-key.b64"


def keygen():
    key = Ed25519PrivateKey.generate()
    pem = key.private_bytes(
        serialization.Encoding.PEM,
        serialization.PrivateFormat.PKCS8,
        serialization.NoEncryption(),
    )
    with open(KEY_FILE, "wb") as f:
        f.write(pem)
    raw = key.public_key().public_bytes(
        serialization.Encoding.Raw, serialization.PublicFormat.Raw
    )
    with open(PUB_FILE, "wb") as f:
        f.write(raw)
    print(f"私钥已写入 {KEY_FILE}（保密！绝不外发、绝不提交到 git 仓库）")
    print(f"公钥已写入 {PUB_FILE}")
    print("公钥 base64（将来登记到平台时用这一串）:")
    print(base64.b64encode(raw).decode())


def members_of(path):
    members = {}
    with tarfile.open(path, "r:gz") as tar:
        for member in tar.getmembers():
            if not member.isfile():
                sys.exit(f"包里有不支持的成员: {member.name}（只允许普通文件）")
            members[member.name] = tar.extractfile(member).read()
    return members


def content_digest(members):
    entries = [
        (name, hashlib.sha256(data).hexdigest())
        for name, data in members.items()
        if name != "signature.json"
    ]
    entries.sort(key=lambda entry: entry[0])
    text = "".join(f"{digest} {name}\n" for name, digest in entries).encode()
    return "sha256:" + hashlib.sha256(text).hexdigest()


def sign(package, key_id):
    digest = content_digest(members_of(package))
    with open(KEY_FILE, "rb") as f:
        key = serialization.load_pem_private_key(f.read(), password=None)
    if not isinstance(key, Ed25519PrivateKey):
        sys.exit(f"{KEY_FILE} 不是 Ed25519 私钥")
    envelope = {
        "alg": "ed25519",
        "key_id": key_id,
        "content_digest": digest,
        "signature": base64.b64encode(key.sign(digest.encode())).decode(),
    }
    with open("signature.json", "w", encoding="utf-8") as f:
        f.write(json.dumps(envelope, indent=2) + "\n")
    print(f"已生成 signature.json（content_digest = {digest}）")


if __name__ == "__main__":
    if len(sys.argv) >= 2 and sys.argv[1] == "keygen":
        keygen()
    elif len(sys.argv) >= 4 and sys.argv[1] == "sign":
        sign(sys.argv[2], sys.argv[3])
    else:
        print(__doc__)
```

### 5.4 三步走

**第一步（只做一次）：生成你的密钥对**

```sh
python3 paasplugin-sign.py keygen
```

会生成 `author-private-key.pem`（私钥，保密）和 `author-public-key.b64`（公钥）。终端打印的那串 base64 就是你的公钥，抄下来——将来平台开放"作者公钥登记"时要用它。

**第二步：对未签名的包签名**

```sh
python3 paasplugin-sign.py sign example.paasplugin author-key-1
```

`author-key-1` 是你给这把钥匙起的名字（`key_id`），建议只用小写字母、数字和连字符。当前文件夹里会出现 `signature.json`，内容形如：

```json
{
  "alg": "ed25519",
  "key_id": "author-key-1",
  "content_digest": "sha256:9f86d0818...共64位十六进制",
  "signature": "标准 base64 的 64 字节签名"
}
```

**第三步：把签名放进包里，重新打包**

```sh
tar -czf example-signed.paasplugin plugin-manifest.json plugin.wasm signature.json
```

注意：加进 `signature.json` 再打包，`package_digest` 会变（整包字节变了），但 `content_digest` 不变（规则明确排除 `signature.json`），所以第二步算出的签名依然有效——这正是设计意图。

### 5.5 签名之后会看到什么

平台开放作者注册和公钥登记**之前**，带签名的包上传后 `signature.state` 会是 `present_unverified`——意思只是"检测到了格式正确的签名，但还没法验证它可信"，不是签名错了，也不是被处罚。开放之后：签名对得上登记公钥的包会显示 `verified`；对不上会给出具体原因（比如 `signing_key_unknown`、`signature_invalid`）。这些状态值的含义见[发布与安装](06-publish-and-install.md)。

## 6. 上传前自查清单

逐条打勾再上传，能避掉绝大多数 400 错误：

- [ ] `plugin-manifest.json` 能被 JSON 解析（无多余逗号、无中文引号、无 `null`）
- [ ] `plugin_id` 以 `plg_` 开头且只有小写字母/数字/下划线
- [ ] `version` 是三段数字（要发新内容时已递增）
- [ ] `plugin.wasm` 开头第 8 个字节是 `01`（是组件）
- [ ] 包里只有 `plugin-manifest.json`、`plugin.wasm`、可选的 `signature.json`，没有 `ui/`、没有文件夹、没有链接
- [ ] 各项大小都在第 3 节的限制内
- [ ] 保留了第一次打包生成的那个 `.paasplugin` 文件（重试上传要用它）

下一步：按[发布与安装](06-publish-and-install.md)把包传上去。
