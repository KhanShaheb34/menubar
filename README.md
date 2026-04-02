# 💻 menubar

[![skills.sh](https://img.shields.io/badge/skills.sh-menubar-blue)](https://skills.sh/khanshaheb34/menubar/menubar)

A Claude Code skill for creating single-file macOS menubar apps with SwiftUI.

Describe what you want to monitor or display, and get a complete, polished menubar app — compiled with a single `swiftc` command, no Xcode required.

## What it does

This skill teaches Claude how to build macOS menubar apps that:

- Live in the system menu bar (no Dock icon)
- Show a popover with live-updating data when clicked
- Use proven patterns: NSStatusItem + NSPopover, not MenuBarExtra
- Compile from a single `.swift` file with `swiftc`
- Include sparkline charts, color-coded metrics, and polished UI components

## Examples

Ask Claude to build you a menubar app for:

- CPU usage and top processes
- Disk space across all volumes
- WiFi signal strength and connected devices
- Docker container status
- API health monitoring
- Battery cycle count and health
- Git repo status
- Network bandwidth per process
- Any live data you want at a glance

## Install

Run the following script to install the skill for your preferred AI agents:

```bash
npx skills add KhanShaheb34/menubar
```

Or manually clone and add to your Claude Code skills:

```bash
git clone https://github.com/KhanShaheb34/menubar.git
```

Then reference the skill in your settings:

```json
// .claude/settings.json
{
  "skills": ["./path/to/menubar"]
}
```

## Requirements

- macOS (the apps use AppKit and SwiftUI)
- Swift compiler (`swiftc`) — comes with Xcode Command Line Tools:
  ```bash
  xcode-select --install
  ```
- Claude Code

## How it works

Once installed, the skill activates automatically when you ask Claude to build a menubar app. It provides:

1. **Architecture** — The exact AppDelegate + NSStatusItem + NSPopover boilerplate
2. **UI Components** — SparklineView, RateCardView, UsageBarView, SectionHeader, and more
3. **Data Collection** — Patterns for running shell commands (`Process`/`Pipe`) and parsing macOS CLI tools
4. **Design Guidelines** — Monospaced numbers, color-coded status, two-column layouts, SF Symbols

The output is always a single `.swift` file that compiles and runs immediately:

```bash
swiftc -parse-as-library -framework SwiftUI -framework AppKit -o MyApp MyApp.swift
./MyApp
```

## Skill structure

```
menubar-app-creator/
├── SKILL.md                  # Main skill instructions
├── references/
│   ├── architecture.md       # AppDelegate, Monitor pattern, build command
│   ├── ui-components.md      # Reusable SwiftUI components
│   └── data-collection.md    # Shell command patterns, macOS data sources
└── evals/
    └── evals.json            # Test cases for skill evaluation
```

## Credits

Inspired by [Simon Willison's vibe-coded SwiftUI apps](https://simonwillison.net/2026/Mar/27/vibe-coding-swiftui/) — [Bandwidther](https://github.com/simonw/bandwidther) and [Gpuer](https://github.com/simonw/gpuer).

## License

MIT
