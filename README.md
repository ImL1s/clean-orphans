# 🧹 Clean Orphans

[![macOS](https://img.shields.io/badge/os-macOS-black?logo=apple)](#) [![Linux](https://img.shields.io/badge/os-Linux-blue?logo=linux)](#) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A smart, safe, and lightning-fast cleanup script for development environments. 

It identifies and kills orphaned background processes (processes where `PPID=1`) to reclaim memory **without affecting your active development sessions** (like your IDE, terminal, or Claude Code/AI agents). 

Specially optimized for **macOS** mobile developers (Flutter/iOS/Android) and developers using modern AI tools (MCP servers).

## Why You Need This / 為什麼你需要這個

AI-powered coding tools and mobile development toolchains spawn background processes that **frequently fail to clean up after themselves**. These orphaned processes silently accumulate, consuming **10-20+ GB of RAM** before you even notice your machine slowing down.

AI 程式碼工具和行動開發工具鏈會產生大量背景進程，但**經常無法在退出時正確清理**。這些孤兒進程默默累積，在你察覺電腦變慢之前就已吃掉 **10-20+ GB 記憶體**。

Instead of manually hunting them down in Activity Monitor or rebooting, `clean-orphans` safely sweeps them away in milliseconds.

### Root Causes / 根本原因

#### 1. MCP Servers: No Cleanup on Exit / MCP 伺服器：退出時不清理

Tools like Claude Code, Cursor, and OpenCode spawn [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) servers as child processes. When the parent IDE or terminal exits — especially via crash, force-quit, or closing a tab — these child processes are **not terminated**. They become orphaned (`PPID=1`) and keep running indefinitely.

Claude Code、Cursor、OpenCode 等工具會以子進程方式啟動 MCP 伺服器。當父進程（IDE 或終端）退出時——尤其是崩潰、強制關閉或關閉分頁——這些子進程**不會被終止**，變成孤兒進程（`PPID=1`）並無限期運行。

**Why it happens / 為什麼會發生：**
- macOS lacks `prctl(PR_SET_PDEATHSIG)` — there's [no native way](https://github.com/nodejs/help/issues/1389) to auto-kill children when the parent dies.（macOS 沒有原生機制在父進程死亡時自動清理子進程）
- Node.js `child.kill()` sends SIGTERM but [doesn't wait for cleanup](https://github.com/nodejs/node/issues/34830), and nested `npm exec` wrappers add additional layers that signals don't propagate through.（Node.js 的 kill 不等待清理完成，且 `npm exec` 包裝層會阻擋信號傳遞）
- The [MCP protocol specifies a graceful shutdown phase](https://github.com/anthropics/claude-code/issues/1935), but most host implementations don't invoke it on exit.（MCP 協議定義了優雅關閉流程，但多數宿主實作在退出時並未觸發）

**Real-world impact / 實際影響：**

| Tool | Issue | Impact |
|------|-------|--------|
| Claude Code | [MCP servers not terminated on exit](https://github.com/anthropics/claude-code/issues/1935) | Processes accumulate across sessions |
| Claude Code | [Chrome MCP spawns ~4/min without cleanup](https://github.com/anthropics/claude-code/issues/15861) | **27 GB** over ~10 hours |
| Claude Code | [Subagents leak when parent terminal killed](https://github.com/anthropics/claude-code/issues/20369) | ~45 MB per orphaned process |
| Claude Code | [VS Code extension leaks worker processes](https://github.com/anthropics/claude-code/issues/15906) | OOM killer triggered on Linux |
| Cursor | [MCP child processes not cleaned up](https://forum.cursor.com/t/mcp-server-process-leak/151615) | ~3-5 GB over days |
| Cursor | [MCP processes accumulate over time](https://forum.cursor.com/t/mcp-server-processes-are-not-terminated-and-accumulate-over-time-causing-memory-leaks/143181) | Dozens of orphaned `node`/`npm` processes |
| OpenCode | [MCP processes not terminated on session end](https://github.com/anomalyco/opencode/issues/6633) | [Zombie process accumulation](https://github.com/anomalyco/opencode/issues/11225) |

#### 2. Flutter / Dart: SIGTERM Doesn't Reach the VM / Flutter / Dart：SIGTERM 無法到達 VM

The `flutter` command is a shell script wrapper. When IDE sends SIGTERM on shutdown, the signal hits the shell process but [doesn't propagate to the underlying Dart VM](https://github.com/Dart-Code/Dart-Code/issues/5155). The VM process becomes orphaned while the shell exits cleanly.

`flutter` 命令實際上是一個 shell 腳本包裝器。IDE 關閉時發送的 SIGTERM 只到達 shell 進程，[無法傳遞到底層的 Dart VM](https://github.com/Dart-Code/Dart-Code/issues/5155)。shell 正常退出，但 VM 進程變成孤兒。

The Flutter daemon also [spawns sub-processes like `xcdevice observe`](https://github.com/flutter/flutter/issues/73859) that are never cleaned up.

| Tool | Issue | Impact |
|------|-------|--------|
| Flutter / Dart | [Daemon orphaned when IDE closes](https://github.com/Dart-Code/Dart-Code/issues/5216) | SIGTERM [doesn't propagate](https://github.com/Dart-Code/Dart-Code/issues/5155) through shell wrapper |
| Flutter | [`xcdevice observe` leaked by daemon](https://github.com/flutter/flutter/issues/73859) | Orphaned sub-processes pile up |

#### 3. Gradle: Daemon Multiplication / Gradle：Daemon 不斷繁殖

Gradle daemons are designed to stay alive for performance. But a [new daemon is spawned whenever JVM args, Java version, or Gradle version differ](https://docs.gradle.org/current/userguide/gradle_daemon.html) between builds. Multi-project setups with Kotlin can spawn [3+ Kotlin daemons](https://github.com/gradle/gradle/issues/34755), each consuming 1 GB+ of heap. The built-in 3-hour idle timeout is far too long for developer machines.

Gradle daemon 設計上會常駐以加速建置。但只要 JVM 參數、Java 版本或 Gradle 版本有差異，就會[產生新的 daemon](https://docs.gradle.org/current/userguide/gradle_daemon.html)。多專案 Kotlin 環境可能會[產生 3 個以上 Kotlin daemon](https://github.com/gradle/gradle/issues/34755)，每個佔 1 GB+ heap。內建的 3 小時閒置超時對開發機來說太長了。

| Tool | Issue | Impact |
|------|-------|--------|
| Gradle Daemon | [Multiple instances exhaust memory](https://discuss.gradle.org/t/tons-of-gradle-daemons-exhausting-memory/20579) | Config mismatches spawn duplicates |
| Kotlin Daemon | [Excessive memory usage](https://github.com/gradle/gradle/issues/34755) | 3+ daemons × 1 GB+ each |

#### 4. iOS Simulators: Silent Memory Hogs / iOS 模擬器：沉默的記憶體黑洞

CoreSimulator processes from previous Xcode sessions [linger in the background](https://www.repeato.app/managing-xcodes-coresimulator-devices-folder-a-practical-guide/) because Xcode [has no idea what you still need](https://developer.apple.com/forums/thread/758703) and won't clean them up for you. They collectively consume **10-20+ GB**.

前一次 Xcode 工作階段的 CoreSimulator 進程會[殘留在背景](https://www.repeato.app/managing-xcodes-coresimulator-devices-folder-a-practical-guide/)，因為 Xcode [無法判斷你還需要什麼](https://developer.apple.com/forums/thread/758703)，不會主動清理。合計可佔用 **10-20+ GB**。

---

### How We Solve It / 我們怎麼解決

| Strategy | How | Safe for active work? |
|----------|-----|----------------------|
| **Orphan detection** (`PPID=1`) | Only kills processes whose parent has died — the defining trait of a leaked process（只殺父進程已死的進程——洩漏進程的定義特徵） | Yes — active IDE/terminal children always have a living parent |
| **Pattern matching** | Targets known offenders via `ORPHAN_PATTERNS` regex array, not blanket process killing（用已知的正則比對，不是盲目殺進程） | Yes — only matches specific tool signatures |
| **`pgrep` over `ps\|grep`** | Uses `pgrep -f` to avoid self-matching and reduce false positives（用 `pgrep -f` 避免自身匹配和誤判） | Yes |
| **Graceful termination** | SIGTERM → 2s wait → SIGKILL only for unresponsive processes（先禮貌請求，2 秒後才對頑固進程強制終結） | Yes — gives processes time to save state |
| **Deep mode separation** | Heavy daemons (Gradle, Kotlin LSP) require explicit `--deep` flag; `xcodebuild` further restricted to orphans only（重型 daemon 需要 `--deep` 旗標；xcodebuild 更進一步限制只殺孤兒） | Yes — opt-in, never surprises |
| **Dry-run** | `--dry-run` previews everything without killing（預覽模式，不做任何動作） | N/A — read-only |

---

## 🚀 Installation

Install it directly into your local binary folder (`~/.local/bin`):

```bash
git clone https://github.com/YOUR_USERNAME/clean-orphans.git
cd clean-orphans
./install.sh
```

> **Note:** Ensure `~/.local/bin` is in your `PATH`. You can add this line to your `~/.zshrc` or `~/.bashrc`:
> ```bash
> export PATH="$HOME/.local/bin:$PATH"
> ```

---

## 🛠️ Usage & Options

### 1. Safe Mode (Default)
Safe mode ONLY targets detached, orphaned tools (`PPID=1`). It is **designed to avoid active sessions** — processes attached to a living parent (your IDE, terminal, shell) are never matched.

```bash
clean-orphans
```

### 2. Deep Clean Mode (`--deep`)
When you feel your system lagging, use the `--deep` flag. This shuts down heavy background daemons that aren't technically orphaned but can consume GBs of RAM when idle. *(Tools like Gradle will automatically restart on your next build.)*

> **Warning:** Deep mode kills non-orphaned Gradle, Kotlin LSP, and Flutter daemons. If a build or compilation is actively running, it may be interrupted. Use `--dry-run` first to preview what would be affected.

```bash
clean-orphans --deep
```

### 3. Dry Run Mode (`--dry-run`)
Want to see what the script *would* kill without actually terminating anything? Use the dry-run flag. This is great for auditing exactly how much memory you could reclaim.

```bash
clean-orphans --dry-run
# or test both
clean-orphans --deep --dry-run
```

### 4. Help (`-h`, `--help`)
Display the built-in help menu.

```bash
clean-orphans --help
```

---

## ⚙️ Customization

Want to add your own custom tools to the cleanup list? Open the `clean-orphans` script and add your regex pattern to the `ORPHAN_PATTERNS` array:

```bash
ORPHAN_PATTERNS=(
  # AI & MCP Servers (Common)
  "mcp-server|playwright-mcp"
  
  # ... add yours here!
  "my-custom-heavy-daemon"
)
```
Then run `./install.sh` again to apply the changes globally.

---

## 🤝 Contributing

Pull requests are welcome! If you know of other development tools that frequently leave orphaned background processes, feel free to open a PR to add them to the default regex patterns.

## 📄 License
This project is licensed under the MIT License.
