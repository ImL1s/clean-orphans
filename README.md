# 🧹 Clean Orphans

[![macOS](https://img.shields.io/badge/os-macOS-black?logo=apple)](#) [![Linux](https://img.shields.io/badge/os-Linux-blue?logo=linux)](#) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A smart, safe, and lightning-fast cleanup script for development environments. 

It identifies and kills orphaned background processes (processes where `PPID=1`) to reclaim memory **without affecting your active development sessions** (like your IDE, terminal, or Claude Code/AI agents). 

Specially optimized for **macOS** mobile developers (Flutter/iOS/Android) and developers using modern AI tools (MCP servers).

## ✨ Why do you need this?

When IDEs crash, terminals close unexpectedly, or AI development agents stop, they often leave behind "zombie" or "orphaned" background processes. These can quickly eat up gigabytes of RAM.

Instead of manually hunting them down in Activity Monitor or restarting your computer, `clean-orphans` safely sweeps them away in milliseconds.

### 🎯 Targets Include:
- 🤖 **AI & MCP Servers:** `playwright-mcp`, `mcp-server`, custom tools (`context7`, `mobile-mcp`).
- 📱 **Mobile Dev Tooling:** Stuck `flutter_tester`, Dart tooling daemons, and stale `adb logcat` streams.
- 📦 **Node Wrappers:** Orphaned `npm exec` processes.
- 💥 **Deep Clean Targets:** Gigabyte-heavy daemons like Gradle (`org.gradle.launcher.daemon`), Kotlin LSP, xcodebuild, and iOS Simulators.

### 🛡️ Graceful Termination
The script doesn't just blindly destroy processes. It sends a polite `SIGTERM` first, waits up to 2 seconds for processes to shut down gracefully, and only uses a forceful `SIGKILL` (`-9`) for stubborn, unresponsive processes.

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
Safe mode ONLY targets detached, orphaned tools (`PPID=1`). **It is 100% safe to run anytime.** It will never kill processes attached to an active terminal or IDE.

```bash
clean-orphans
```

### 2. Deep Clean Mode (`--deep`)
When you feel your system lagging, use the `--deep` flag. This will safely shut down heavy background daemons that aren't technically orphaned but can consume GBs of RAM when idle. *(Tools like Gradle will automatically restart on your next build).*

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
