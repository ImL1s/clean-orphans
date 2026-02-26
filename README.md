# 🧹 Clean Orphans

[![macOS](https://img.shields.io/badge/os-macOS-black?logo=apple)](#) [![Linux](https://img.shields.io/badge/os-Linux-blue?logo=linux)](#) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> **[繁體中文版 README](README.zh-TW.md)**

A smart, safe, and lightning-fast cleanup script for development environments.

It identifies and kills orphaned background processes (processes where `PPID=1`) to reclaim memory **without affecting your active development sessions** (like your IDE, terminal, or Claude Code/AI agents).

Specially optimized for **macOS** mobile developers (Flutter/iOS/Android) and developers using modern AI tools (MCP servers).

## Why You Need This

AI-powered coding tools and mobile development toolchains spawn background processes that **frequently fail to clean up after themselves**. These orphaned processes silently accumulate, consuming **10-20+ GB of RAM** before you even notice your machine slowing down.

Instead of manually hunting them down in Activity Monitor or rebooting, `clean-orphans` safely sweeps them away in milliseconds.

### Root Causes

#### 1. MCP Servers: No Cleanup on Exit

Tools like Claude Code, Cursor, OpenCode, and Antigravity spawn [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) servers as child processes. When the parent IDE or terminal exits — especially via crash, force-quit, or closing a tab — these child processes are **not terminated**. They become orphaned (`PPID=1`) and keep running indefinitely.

**Why it happens:**
- macOS lacks `prctl(PR_SET_PDEATHSIG)` — there's [no native way](https://github.com/nodejs/help/issues/1389) to auto-kill children when the parent dies.
- Node.js `child.kill()` sends SIGTERM but [doesn't wait for cleanup](https://github.com/nodejs/node/issues/34830), and nested `npm exec` wrappers add additional layers that signals don't propagate through.
- The [MCP protocol specifies a graceful shutdown phase](https://github.com/anthropics/claude-code/issues/1935), but most host implementations don't invoke it on exit.

**Real-world impact:**

| Tool | Issue | Impact |
|------|-------|--------|
| Claude Code | [MCP servers not terminated on exit](https://github.com/anthropics/claude-code/issues/1935) | Processes accumulate across sessions |
| Claude Code | [Chrome MCP spawns ~4/min without cleanup](https://github.com/anthropics/claude-code/issues/15861) | **27 GB** over ~10 hours |
| Claude Code | [Subagents leak when parent terminal killed](https://github.com/anthropics/claude-code/issues/20369) | ~45 MB per orphaned process |
| Claude Code | [VS Code extension leaks worker processes](https://github.com/anthropics/claude-code/issues/15906) | OOM killer triggered on Linux |
| Cursor | [MCP child processes not cleaned up](https://forum.cursor.com/t/mcp-server-process-leak/151615) | ~3-5 GB over days |
| Cursor | [MCP processes accumulate over time](https://forum.cursor.com/t/mcp-server-processes-are-not-terminated-and-accumulate-over-time-causing-memory-leaks/143181) | Dozens of orphaned `node`/`npm` processes |
| OpenCode | [MCP processes not terminated on session end](https://github.com/anomalyco/opencode/issues/6633) | [Zombie process accumulation](https://github.com/anomalyco/opencode/issues/11225) |
| Antigravity | [Language server high memory consumption](https://discuss.ai.google.dev/t/solved-antigravity-hangs-due-to-language-server-windows-x64-high-memory-consumption/116025) | [Zombie processes after quit](https://antigravity.codes/blog/antigravity-server-crashed-fix); backend ports don't clear |

#### 2. Frontend Dev Servers: Ctrl+C Doesn't Always Work

Webpack-dev-server is [notorious for leaving orphaned node processes](https://github.com/webpack/webpack-dev-server/issues/1201) after Ctrl+C. The [child process cannot be killed](https://github.com/webpack/webpack-dev-server/issues/5026) when spawned via `child_process`, and if the parent exits, the dev server continues running. Similar issues affect Vite, Next.js dev server, and other frontend tooling.

| Tool | Issue | Impact |
|------|-------|--------|
| webpack-dev-server | [Orphaned node.exe processes](https://github.com/webpack/webpack-dev-server/issues/1201) | Port occupied, node process lingers |
| webpack-dev-server | [Child process cannot be killed](https://github.com/webpack/webpack-dev-server/issues/5026) | Parent exit doesn't terminate server |
| Vue/webpack | [Dev server continues after Ctrl+C](https://github.com/vuejs-templates/webpack/issues/802) | Needs manual kill |

#### 3. Flutter / Dart: SIGTERM Doesn't Reach the VM (Mobile)

The `flutter` command is a shell script wrapper. When IDE sends SIGTERM on shutdown, the signal hits the shell process but [doesn't propagate to the underlying Dart VM](https://github.com/Dart-Code/Dart-Code/issues/5155). The VM process becomes orphaned while the shell exits cleanly.

The Flutter daemon also [spawns sub-processes like `xcdevice observe`](https://github.com/flutter/flutter/issues/73859) that are never cleaned up.

| Tool | Issue | Impact |
|------|-------|--------|
| Flutter / Dart | [Daemon orphaned when IDE closes](https://github.com/Dart-Code/Dart-Code/issues/5216) | SIGTERM [doesn't propagate](https://github.com/Dart-Code/Dart-Code/issues/5155) through shell wrapper |
| Flutter | [`xcdevice observe` leaked by daemon](https://github.com/flutter/flutter/issues/73859) | Orphaned sub-processes pile up |

#### 4. Gradle: Daemon Multiplication

Gradle daemons are designed to stay alive for performance. But a [new daemon is spawned whenever JVM args, Java version, or Gradle version differ](https://docs.gradle.org/current/userguide/gradle_daemon.html) between builds. Multi-project setups with Kotlin can spawn [3+ Kotlin daemons](https://github.com/gradle/gradle/issues/34755), each consuming 1 GB+ of heap. The built-in 3-hour idle timeout is far too long for developer machines.

| Tool | Issue | Impact |
|------|-------|--------|
| Gradle Daemon | [Multiple instances exhaust memory](https://discuss.gradle.org/t/tons-of-gradle-daemons-exhausting-memory/20579) | Config mismatches spawn duplicates |
| Kotlin Daemon | [Excessive memory usage](https://github.com/gradle/gradle/issues/34755) | 3+ daemons × 1 GB+ each |

#### 5. iOS Simulators: Silent Memory Hogs

CoreSimulator processes from previous Xcode sessions [linger in the background](https://www.repeato.app/managing-xcodes-coresimulator-devices-folder-a-practical-guide/) because Xcode [has no idea what you still need](https://developer.apple.com/forums/thread/758703) and won't clean them up for you. They collectively consume **10-20+ GB**.

---

### How We Solve It

| Strategy | How | Safe for active work? |
|----------|-----|----------------------|
| **Orphan detection** (`PPID=1`) | Only kills processes whose parent has died — the defining trait of a leaked process | Yes — active IDE/terminal children always have a living parent |
| **Pattern matching** | Targets known offenders via `ORPHAN_PATTERNS` regex array, not blanket process killing | Yes — only matches specific tool signatures |
| **`pgrep` over `ps\|grep`** | Uses `pgrep -f` to avoid self-matching and reduce false positives | Yes |
| **Frontend dev servers** | Catches orphaned webpack-dev-server, vite, next.js, esbuild, turbopack processes | Yes — only PPID=1 orphans |
| **Graceful termination** | SIGTERM → 2s wait → SIGKILL only for unresponsive processes | Yes — gives processes time to save state |
| **Deep mode separation** | Heavy daemons (Gradle, Kotlin LSP) require explicit `--deep` flag; `xcodebuild` further restricted to orphans only | Yes — opt-in, never surprises |
| **Dry-run** | `--dry-run` previews everything without killing | N/A — read-only |

---

## Installation

```bash
git clone https://github.com/ImL1s/clean-orphans.git
cd clean-orphans
./install.sh
```

> **Note:** Ensure `~/.local/bin` is in your `PATH`. Add this line to your `~/.zshrc` or `~/.bashrc`:
> ```bash
> export PATH="$HOME/.local/bin:$PATH"
> ```

---

## Usage

### Safe Mode (Default)
Safe mode ONLY targets detached, orphaned tools (`PPID=1`). It is **designed to avoid active sessions** — processes attached to a living parent (your IDE, terminal, shell) are never matched.

```bash
clean-orphans
```

### Deep Clean Mode (`--deep`)
Shuts down heavy background daemons that aren't technically orphaned but can consume GBs of RAM when idle. *(Tools like Gradle will automatically restart on your next build.)*

> **Warning:** Deep mode kills non-orphaned Gradle, Kotlin LSP, and Flutter daemons. If a build or compilation is actively running, it may be interrupted. Use `--dry-run` first to preview what would be affected.

```bash
clean-orphans --deep
```

### Dry Run Mode (`--dry-run`)
Preview what the script *would* kill without actually terminating anything. Great for auditing how much memory you could reclaim.

```bash
clean-orphans --dry-run
clean-orphans --deep --dry-run
```

### Help (`-h`, `--help`)
```bash
clean-orphans --help
```

---

## Customization

Add your own tools to the cleanup list by editing the `ORPHAN_PATTERNS` array in the `clean-orphans` script:

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

## Contributing

Pull requests are welcome! If you know of other development tools that frequently leave orphaned background processes, feel free to open a PR to add them to the default regex patterns.

## License
This project is licensed under the MIT License.
