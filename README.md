# Clean Orphans

A smart and safe cleanup script for development environments. It identifies and kills orphaned background processes (processes where PPID=1) to reclaim memory without affecting your active development sessions (like IDEs, Claude Code, etc.).

Designed primarily for **macOS**, but works safely on **Linux** as well.

## Features

- **Safe Mode**: Only targets detached, orphaned tools (`PPID=1`), ensuring no active work is disrupted.
- **AI / MCP Support**: Automatically cleans up orphaned Model Context Protocol (MCP) servers and AI dev tools.
- **Mobile Dev**: Kills stuck Flutter testers, Dart tooling, and stale ADB logcat streams.
- **Deep Clean Mode**: Safely shuts down heavy background daemons that aren't technically orphaned but can consume GBs of RAM when idle (e.g., Gradle Daemons, Kotlin LSP, iOS Simulators).

## Installation

```bash
git clone <your-repo-url> clean-orphans
cd clean-orphans
./install.sh
```

Ensure `~/.local/bin` is in your `PATH` (e.g., by adding `export PATH="$HOME/.local/bin:$PATH"` to your `~/.zshrc` or `~/.bashrc`).

## Usage

Run the script safely to clean up orphaned background processes:
```bash
clean-orphans
```

Run with `--deep` to aggressively shut down heavy memory consumers like Kotlin LSP, Gradle Daemons, and all iOS Simulators (macOS only):
```bash
clean-orphans --deep
```
