# ipatool-gui

**English** | [简体中文](README.zh-CN.md)

> A Chinese-localized graphical interface for [ipatool](https://github.com/majd/ipatool) — search, purchase and download IPA packages from the App Store without touching the command line.

![Go](https://img.shields.io/badge/Go-1.25%2B-00ADD8?logo=go&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-blue)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey)

---

## What is this

[ipatool](https://github.com/majd/ipatool) is an excellent command line tool for interacting with Apple's IPA files. This project adds a **graphical interface on top of it**, with a **fully localized Simplified Chinese** UI, so that you can sign in, search, browse your purchases and download packages by clicking instead of typing.

The original CLI is **untouched** — every existing command still works exactly as before. The GUI is simply an extra `gui` subcommand.

## Features

- **Localized interface** — all UI text *and* backend error messages are in Simplified Chinese
- **Sign in with Apple ID**, including two-factor authentication (a code field appears automatically when Apple asks for one)
- **Search the App Store** by keyword, with platform selection (iPhone / iPad / Apple TV / visionOS / macOS) and result count
- **Browse your purchased apps** with pagination
- **Historical versions** — list every version identifier of an app and download a specific one
- **Live download progress** — progress bar, downloaded / total size, cancel, and "open containing folder"
- **Automatic recovery** — obtains the license automatically for free apps, and re-authenticates if the session expires mid-download
- **Fully offline UI** — the interface is embedded into the binary and served from `127.0.0.1`, nothing is loaded from the internet

## How it works

```
ipatool-gui.exe
   └── cmd/gui.go        local HTTP server (net/http) + JSON handlers
         ├── /api/*      thin adapters over the existing pkg/appstore service layer
         └── cmd/webui/  single-page interface, embedded with go:embed
```

Instead of rewriting the App Store logic, the GUI calls the very same `pkg/appstore` services the CLI uses. The frontend is plain HTML/CSS/JavaScript embedded into the executable, so the result is **one self-contained binary** — no Node.js, no Electron, no WebView2 runtime to install.

## Requirements

- **Go 1.25 or newer** (build time only)
- No CGO and no C compiler — the whole project is pure Go, so `CGO_ENABLED=0` builds fine

## Build

```bash
git clone https://github.com/AndyDEV233/ipatool-gui.git
cd ipatool-gui

# Windows
go build -o ipatool-gui.exe .

# macOS / Linux
go build -o ipatool-gui .
```

Run the test suite with `go test ./...`.

> The Go module path is still `github.com/majd/ipatool/v2` because this is a fork, so build from a clone as shown above — `go install github.com/AndyDEV233/ipatool-gui@latest` will not work.

## Usage

### Graphical interface

Double-click the binary, or run it with no arguments:

```bash
./ipatool-gui
```

The local server starts on a random free port and your default browser opens automatically. Keep the console window open while you use the GUI — closing it stops the server.

To pass options, call the `gui` subcommand explicitly:

```bash
./ipatool-gui gui --port 8123 --no-browser
```

| Flag | Description |
| --- | --- |
| `--port <n>` | Listen on a fixed port instead of a random one |
| `--no-browser` | Do not open the browser automatically |

### First run

1. Open **设置** (Settings) and set a **keychain passphrase**.
   This is a *local* passphrase used to encrypt your Apple ID credentials on this machine. It is **not** your Apple ID password, it is never transmitted anywhere, and you should remember it — the same passphrase is required the next time you start the app.
2. Click **登录 Apple ID** and sign in. If the account uses two-factor authentication, a code field appears; enter the 6-digit code from your device and submit again.
3. Search for an app, or open **我的已购**, then press **下载**.
4. Downloads are written to the current working directory unless you pick another folder in Settings.

### Command line (unchanged)

```bash
./ipatool-gui auth login --email you@example.com --password secret
./ipatool-gui search "telegram" --limit 10
./ipatool-gui download --bundle-identifier org.telegram.Telegram-iOS --output ./ipa
./ipatool-gui list-purchases --limit 50
```

Run `./ipatool-gui --help` for the full command reference. Note that the cobra command name is still `ipatool` in the help output.

## HTTP API

The interface is driven by a small JSON API bound to `127.0.0.1` only.

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/api/state` | Keychain status, current account, download folder, version |
| `POST` | `/api/keychain` | Set the local keychain passphrase |
| `POST` | `/api/login` | Sign in (returns `need2FA: true` when a verification code is required) |
| `POST` | `/api/logout` | Revoke and clear the stored credentials |
| `GET` | `/api/search` | `term`, `limit`, `platform` |
| `GET` | `/api/versions` | `appId` or `bundleId`, `platform` |
| `GET` | `/api/purchases` | `page`, `limit`, `platform` |
| `POST` | `/api/purchase` | Obtain a license for a free app |
| `POST` | `/api/download` | Start a download (one at a time) |
| `GET` | `/api/download` | Poll the current task: stage, percentage, bytes, destination, error |
| `POST` | `/api/download/cancel` | Cancel the running download |
| `POST` | `/api/outputdir` | Change the download folder |
| `POST` | `/api/reveal` | Reveal a path in the system file manager |

## Project layout

```
cmd/gui.go        GUI subcommand: local HTTP server, JSON handlers, download task runner
cmd/webui/        Embedded single-page interface
  index.html
  style.css
  app.js
cmd/root.go       Registers the gui command; a bare run launches the GUI
```

Everything else comes from upstream ipatool and is left as-is.

## Limitations

- Only **free** apps can be obtained — this is an App Store API restriction inherited from ipatool.
- Apple may require a valid payment method on the account even for free apps.
- Tested on Windows. The GUI itself is cross-platform; macOS and Linux should work but have not been verified as thoroughly.

## Disclaimer

This project is only a user interface on top of ipatool. It does not break DRM, and it cannot download anything your Apple ID is not entitled to. Use it to back up apps you have legitimately obtained, for personal use only. You are responsible for complying with Apple's terms of service and with the laws of your country.

## Credits

- [majd/ipatool](https://github.com/majd/ipatool) — the original CLI and the entire service layer this GUI is built on
- Graphical interface and Simplified Chinese localization — [AndyDEV233](https://github.com/AndyDEV233)

## License

MIT. See [LICENSE](LICENSE). Copyright for the original work belongs to the ipatool authors.
