# ipatool-gui

[English](README.md) | **简体中文**

> 给 [ipatool](https://github.com/majd/ipatool) 套上中文图形界面 —— 不用敲命令，点几下就能登录 App Store、搜索应用、下载 IPA。

![Go](https://img.shields.io/badge/Go-1.25%2B-00ADD8?logo=go&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-blue)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey)

---

## 这是什么

[ipatool](https://github.com/majd/ipatool) 是一个非常棒的 App Store IPA 处理命令行工具。本项目在它之上加了一层**图形界面**，并且界面与提示**全中文**，登录、搜索、看已购、下载都能点着完成。

原来的命令行功能**完全没有改动**，所有既有命令照旧可用；图形界面只是一个新增的 `gui` 子命令。

## 功能

- **全中文界面**：UI 文案以及后端报错提示都做了中文化
- **Apple ID 登录**：支持双重验证（需要验证码时会自动出现输入框）
- **应用搜索**：支持选择平台（iPhone / iPad / Apple TV / visionOS / macOS）与结果条数
- **我的已购**：分页浏览账号已购（含免费领取）的应用
- **历史版本**：列出某个应用的全部版本标识，可指定版本下载
- **下载进度**：实时进度条、已下载 / 总大小、取消下载、一键打开所在文件夹
- **自动补救**：免费应用会自动获取许可；下载途中登录失效会自动重新登录
- **界面完全本地**：前端打进 exe 里，由 `127.0.0.1` 提供，不加载任何外部资源

## 实现方式

```
ipatool-gui.exe
   └── cmd/gui.go        本地 HTTP 服务（net/http）+ JSON 接口
         ├── /api/*      对已有 pkg/appstore 服务层的薄封装
         └── cmd/webui/  单页界面，通过 go:embed 内嵌进二进制
```

图形界面没有重写 App Store 逻辑，而是直接调用命令行用的同一套 `pkg/appstore` 服务。前端是纯 HTML/CSS/JavaScript 并嵌入可执行文件，所以最终只是**一个自包含的 exe** —— 不需要装 Node.js、Electron 或 WebView2 运行时。

## 环境要求

- **Go 1.25 或更高**（仅编译时需要）
- 不需要 CGO、不需要 C 编译器：全项目纯 Go，`CGO_ENABLED=0` 也能正常编译

## 编译

```bash
git clone https://github.com/AndyDEV233/ipatool-gui.git
cd ipatool-gui

# Windows
go build -o ipatool-gui.exe .

# macOS / Linux
go build -o ipatool-gui .
```

跑测试：`go test ./...`

> 因为这是 fork，Go 模块路径仍保留为 `github.com/majd/ipatool/v2`，所以请按上面的方式克隆后编译；`go install github.com/AndyDEV233/ipatool-gui@latest` 是不可用的。

## 使用

### 图形界面

双击 exe，或者直接不带参数运行：

```bash
./ipatool-gui
```

程序会在本地随机端口启动服务，并自动打开默认浏览器。**使用期间请保持那个命令行窗口不要关闭** —— 关掉窗口服务就停了。

要带参数启动，请显式调用 `gui` 子命令：

```bash
./ipatool-gui gui --port 8123 --no-browser
```

| 参数 | 说明 |
| --- | --- |
| `--port <n>` | 使用固定端口，默认自动分配 |
| `--no-browser` | 启动后不自动打开浏览器 |

### 首次使用

1. 打开 **设置**，先设置一个**钥匙串密码**。
   这是**本机**用来加密保存 Apple ID 凭据的口令，**不是你的 Apple ID 密码**，也不会发送到任何地方。请务必记住，下次启动还要用它。
2. 点 **登录 Apple ID** 登录。如果账号开了双重验证，会出现验证码输入框，填入设备上收到的 6 位验证码后再点一次登录。
3. 搜索应用，或打开 **我的已购**，点 **下载** 即可。
4. 下载文件默认保存到程序当前目录，也可以在设置里改成别的文件夹。

### 命令行（原样保留）

```bash
./ipatool-gui auth login --email you@example.com --password secret
./ipatool-gui search "telegram" --limit 10
./ipatool-gui download --bundle-identifier org.telegram.Telegram-iOS --output ./ipa
./ipatool-gui list-purchases --limit 50
```

完整命令说明见 `./ipatool-gui --help`。注意帮助信息里的命令名仍显示为上游的 `ipatool`。

## HTTP 接口

界面由一套只监听 `127.0.0.1` 的 JSON 接口驱动。

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/api/state` | 钥匙串状态、当前账号、下载目录、版本号 |
| `POST` | `/api/keychain` | 设置本机钥匙串密码 |
| `POST` | `/api/login` | 登录（需要验证码时返回 `need2FA: true`） |
| `POST` | `/api/logout` | 注销并清除已保存的凭据 |
| `GET` | `/api/search` | 参数：`term`、`limit`、`platform` |
| `GET` | `/api/versions` | 参数：`appId` 或 `bundleId`、`platform` |
| `GET` | `/api/purchases` | 参数：`page`、`limit`、`platform` |
| `POST` | `/api/purchase` | 为免费应用获取许可 |
| `POST` | `/api/download` | 开始下载（同一时间只允许一个任务） |
| `GET` | `/api/download` | 轮询当前任务：阶段、百分比、字节数、目标路径、错误 |
| `POST` | `/api/download/cancel` | 取消正在进行的下载 |
| `POST` | `/api/outputdir` | 修改下载目录 |
| `POST` | `/api/reveal` | 在系统文件管理器中定位路径 |

## 目录结构

```
cmd/gui.go        GUI 子命令：本地 HTTP 服务、JSON 接口、下载任务调度
cmd/webui/        内嵌的单页界面
  index.html
  style.css
  app.js
cmd/root.go       注册 gui 命令；不带参数启动时直接进入图形界面
```

其余文件均来自上游 ipatool，未做改动。

## 已知限制

- 只能获取**免费**应用 —— 这是 ipatool 继承下来的 App Store 接口限制。
- 即使是免费应用，苹果也可能要求账号绑定有效的支付方式。
- 目前主要在 Windows 上验证。界面本身跨平台，macOS / Linux 理论上可用但未充分测试。

## 免责声明

本项目只是在 ipatool 之上做了一层界面，不涉及任何破解，也无法下载你 Apple ID 无权获取的内容。请仅用于备份你合法拥有的应用，且只作个人用途。你需要自行承担遵守 Apple 服务条款及当地法律法规的责任。

## 致谢

- [majd/ipatool](https://github.com/majd/ipatool) —— 原始命令行工具与整套服务层，本项目的图形界面完全建立在它之上
- 图形界面与中文本地化 —— [AndyDEV233](https://github.com/AndyDEV233)

## 许可

MIT，见 [LICENSE](LICENSE)。原始代码版权归 ipatool 作者所有。
