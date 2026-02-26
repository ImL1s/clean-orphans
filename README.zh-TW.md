# 🧹 Clean Orphans

[![macOS](https://img.shields.io/badge/os-macOS-black?logo=apple)](#) [![Linux](https://img.shields.io/badge/os-Linux-blue?logo=linux)](#) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> **[English README](README.md)**

一個聰明、安全、極速的開發環境清理腳本。

自動辨識並終結孤兒背景進程（`PPID=1` 的進程）來回收記憶體，**不影響你正在使用的開發工具**（IDE、終端、Claude Code 等）。

專為 **macOS** 行動開發者（Flutter/iOS/Android）和使用現代 AI 工具（MCP 伺服器）的開發者優化。

## 為什麼你需要這個

AI 程式碼工具和行動開發工具鏈會產生大量背景進程，但**經常無法在退出時正確清理**。這些孤兒進程默默累積，在你察覺電腦變慢之前就已吃掉 **10-20+ GB 記憶體**。

與其在活動監視器裡手動搜尋或直接重開機，不如用 `clean-orphans` 在毫秒內安全清理。

### 根本原因

#### 1. MCP 伺服器：退出時不清理

Claude Code、Cursor、OpenCode、Antigravity 等工具會以子進程方式啟動 [MCP（Model Context Protocol）](https://modelcontextprotocol.io/) 伺服器。當父進程（IDE 或終端）退出時——尤其是崩潰、強制關閉或關閉分頁——這些子進程**不會被終止**，變成孤兒進程（`PPID=1`）並無限期運行。

**為什麼會發生：**
- macOS 缺少 `prctl(PR_SET_PDEATHSIG)` —— [沒有原生機制](https://github.com/nodejs/help/issues/1389)在父進程死亡時自動清理子進程。
- Node.js 的 `child.kill()` 送出 SIGTERM 但[不等待清理完成](https://github.com/nodejs/node/issues/34830)，而巢狀的 `npm exec` 包裝層會阻擋信號傳遞。
- [MCP 協議定義了優雅關閉流程](https://github.com/anthropics/claude-code/issues/1935)，但多數宿主實作在退出時並未觸發。

**實際影響：**

| 工具 | 問題 | 影響 |
|------|------|------|
| Claude Code | [退出時 MCP 伺服器未終止](https://github.com/anthropics/claude-code/issues/1935) | 進程跨工作階段累積 |
| Claude Code | [Chrome MCP 每分鐘產生約 4 個進程且不清理](https://github.com/anthropics/claude-code/issues/15861) | 約 10 小時後佔 **27 GB** |
| Claude Code | [父終端被殺後子代理進程洩漏](https://github.com/anthropics/claude-code/issues/20369) | 每個孤兒進程約 45 MB |
| Claude Code | [VS Code 擴充套件洩漏 worker 進程](https://github.com/anthropics/claude-code/issues/15906) | Linux 上觸發 OOM killer |
| Cursor | [MCP 子進程未被清理](https://forum.cursor.com/t/mcp-server-process-leak/151615) | 數天內洩漏 3-5 GB |
| Cursor | [MCP 進程隨時間累積](https://forum.cursor.com/t/mcp-server-processes-are-not-terminated-and-accumulate-over-time-causing-memory-leaks/143181) | 數十個孤兒 `node`/`npm` 進程 |
| OpenCode | [工作階段結束時 MCP 進程未終止](https://github.com/anomalyco/opencode/issues/6633) | [殭屍進程累積](https://github.com/anomalyco/opencode/issues/11225) |
| Antigravity | [Language server 高記憶體消耗](https://discuss.ai.google.dev/t/solved-antigravity-hangs-due-to-language-server-windows-x64-high-memory-consumption/116025) | [退出後殭屍進程殘留](https://antigravity.codes/blog/antigravity-server-crashed-fix)；背景端口未釋放 |

#### 2. 前端 Dev Server：Ctrl+C 不一定有用

webpack-dev-server [以留下孤兒 node 進程聞名](https://github.com/webpack/webpack-dev-server/issues/1201)，Ctrl+C 後 node 進程不會死。用 `child_process` 產生的 [子進程無法被 kill](https://github.com/webpack/webpack-dev-server/issues/5026)，父進程退出後 dev server 繼續跑。Vite、Next.js dev server 等前端工具也有類似問題。

| 工具 | 問題 | 影響 |
|------|------|------|
| webpack-dev-server | [孤兒 node 進程](https://github.com/webpack/webpack-dev-server/issues/1201) | 佔用端口，node 進程殘留 |
| webpack-dev-server | [子進程無法被 kill](https://github.com/webpack/webpack-dev-server/issues/5026) | 父進程退出不會終止 server |
| Vue/webpack | [Ctrl+C 後 dev server 繼續跑](https://github.com/vuejs-templates/webpack/issues/802) | 需要手動 kill |

#### 3. Flutter / Dart：SIGTERM 無法到達 VM（行動開發）

`flutter` 命令實際上是一個 shell 腳本包裝器。IDE 關閉時發送的 SIGTERM 只到達 shell 進程，[無法傳遞到底層的 Dart VM](https://github.com/Dart-Code/Dart-Code/issues/5155)。shell 正常退出，但 VM 進程變成孤兒。

Flutter daemon 也會[產生 `xcdevice observe` 等子進程](https://github.com/flutter/flutter/issues/73859)，且從不清理。

| 工具 | 問題 | 影響 |
|------|------|------|
| Flutter / Dart | [IDE 關閉後 daemon 變成孤兒](https://github.com/Dart-Code/Dart-Code/issues/5216) | SIGTERM [無法穿透](https://github.com/Dart-Code/Dart-Code/issues/5155) shell 包裝層 |
| Flutter | [daemon 洩漏 `xcdevice observe`](https://github.com/flutter/flutter/issues/73859) | 孤兒子進程持續累積 |

#### 4. Gradle：Daemon 不斷繁殖

Gradle daemon 設計上會常駐以加速建置。但只要 JVM 參數、Java 版本或 Gradle 版本有差異，就會[產生新的 daemon](https://docs.gradle.org/current/userguide/gradle_daemon.html)。多專案 Kotlin 環境可能會[產生 3 個以上 Kotlin daemon](https://github.com/gradle/gradle/issues/34755)，每個佔 1 GB+ heap。內建的 3 小時閒置超時對開發機來說太長了。

| 工具 | 問題 | 影響 |
|------|------|------|
| Gradle Daemon | [多實例耗盡記憶體](https://discuss.gradle.org/t/tons-of-gradle-daemons-exhausting-memory/20579) | 設定不匹配就產生副本 |
| Kotlin Daemon | [過度記憶體使用](https://github.com/gradle/gradle/issues/34755) | 3+ daemon × 每個 1 GB+ |

#### 5. iOS 模擬器：沉默的記憶體黑洞

前一次 Xcode 工作階段的 CoreSimulator 進程會[殘留在背景](https://www.repeato.app/managing-xcodes-coresimulator-devices-folder-a-practical-guide/)，因為 Xcode [無法判斷你還需要什麼](https://developer.apple.com/forums/thread/758703)，不會主動清理。合計可佔用 **10-20+ GB**。

---

### 我們怎麼解決

| 策略 | 做法 | 對正在使用的工作安全嗎？ |
|------|------|--------------------------|
| **孤兒偵測** (`PPID=1`) | 只殺父進程已死的進程——洩漏進程的定義特徵 | 安全 — 活躍的 IDE/終端子進程永遠有存活的父進程 |
| **正則比對** | 用 `ORPHAN_PATTERNS` 陣列比對已知的問題工具，不是盲目殺進程 | 安全 — 只比對特定工具簽名 |
| **`pgrep` 取代 `ps\|grep`** | 使用 `pgrep -f` 避免自身匹配和誤判 | 安全 |
| **前端 dev server** | 捕捉孤兒 webpack-dev-server、vite、next.js、esbuild、turbopack 進程 | 安全 — 只清 PPID=1 孤兒 |
| **優雅終結** | SIGTERM → 等 2 秒 → 只對無回應的進程使用 SIGKILL | 安全 — 給進程儲存狀態的時間 |
| **深度模式分離** | 重型 daemon（Gradle、Kotlin LSP）需要明確的 `--deep` 旗標；xcodebuild 更進一步限制只殺孤兒 | 安全 — 需主動選擇，不會意外觸發 |
| **預覽模式** | `--dry-run` 預覽所有動作但不執行 | 不適用 — 只讀 |

---

## 安裝

```bash
git clone https://github.com/ImL1s/clean-orphans.git
cd clean-orphans
./install.sh
```

> **注意：** 確保 `~/.local/bin` 在你的 `PATH` 中。將以下行加入 `~/.zshrc` 或 `~/.bashrc`：
> ```bash
> export PATH="$HOME/.local/bin:$PATH"
> ```

---

## 使用方式

### 安全模式（預設）
安全模式只清理已脫離的孤兒工具（`PPID=1`）。**設計上避免影響活躍的工作階段** — 有存活父進程的進程不會被比對到。

```bash
clean-orphans
```

### 深度清理模式（`--deep`）
關閉不算孤兒但閒置時佔用大量記憶體的重型背景 daemon。*（Gradle 等工具會在下次建置時自動重啟。）*

> **警告：** 深度模式會殺掉非孤兒的 Gradle、Kotlin LSP、Flutter daemon。如果正在進行建置或編譯，可能會被中斷。請先用 `--dry-run` 預覽會被影響的進程。

```bash
clean-orphans --deep
```

### 預覽模式（`--dry-run`）
預覽腳本*會殺掉*什麼，但不實際執行。適合用來審查你能回收多少記憶體。

```bash
clean-orphans --dry-run
clean-orphans --deep --dry-run
```

### 說明（`-h`、`--help`）
```bash
clean-orphans --help
```

---

## 自訂

在 `clean-orphans` 腳本中編輯 `ORPHAN_PATTERNS` 陣列即可加入你的工具：

```bash
ORPHAN_PATTERNS=(
  # AI & MCP Servers (Common)
  "mcp-server|playwright-mcp"

  # ...在這裡加入你的！
  "my-custom-heavy-daemon"
)
```
然後再跑一次 `./install.sh` 即可全域套用。

---

## 貢獻

歡迎 Pull Request！如果你知道其他經常留下孤兒背景進程的開發工具，歡迎開 PR 把它們加入預設的正則比對清單。

## 授權
本專案採用 MIT 授權。
