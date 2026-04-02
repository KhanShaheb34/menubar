---
name: menubar
description: TRIGGER whenever a user asks to build, create, or make anything that lives in the macOS menu bar / status bar / system tray. This includes menubar apps, menubar tools, menubar utilities, status bar indicators, or "a little thing that sits in my menu bar." Also trigger for macOS utilities that monitor or display live data (CPU, RAM, network, docker, files, APIs, temperature) as a menu bar icon or popover — even if the user doesn't say "menubar" explicitly but describes a small always-on macOS monitor. Contains required NSStatusItem + NSPopover boilerplate, SwiftUI component library, and single-file build pattern that cannot be reconstructed from memory. Do NOT trigger for iOS apps, Electron apps, VS Code extensions, terminal scripts, or launch agents.
---

# Menubar App Creator

Build beautiful, single-file macOS menubar apps with SwiftUI. These apps live in the system menu bar, show a popover when clicked, and display live-updating information — all compiled with a single `swiftc` command, no Xcode required.

This pattern was pioneered by Simon Willison's "vibe coding" approach: describe what you want to monitor or display, and produce a complete, self-contained macOS app.

## Architecture Overview

Every menubar app follows the same proven structure in a single `.swift` file:

```
1. Imports (SwiftUI, AppKit, Foundation, plus domain-specific frameworks)
2. Data Models (structs for your domain data)
3. Data Collection (functions that gather info via shell commands or system APIs)
4. Observable Monitor (ObservableObject class with @Published properties + Timers)
5. Formatting Helpers (human-readable number/byte/date formatters)
6. Reusable UI Components (SparklineView, RateCardView, SectionHeader, etc.)
7. ContentView (the main popover layout)
8. AppDelegate (NSStatusItem + NSPopover — the menubar integration)
9. @main App Entry Point (bridges SwiftUI to AppKit)
```

Read `references/architecture.md` for the complete boilerplate code for sections 8 and 9 (the menubar integration), the ObservableObject monitor pattern, and the build command.

Read `references/ui-components.md` for the full library of reusable SwiftUI components (SparklineView, RateCardView, UsageBarView, SectionHeader, SortButton, ProcessRow, BarView) with their complete implementations.

Read `references/data-collection.md` for the Process/Pipe pattern for running shell commands.

## Framework Imports

Always import SwiftUI, AppKit, and Foundation. Add domain-specific frameworks as needed:

| Framework | When to add | Build flag |
|-----------|------------|------------|
| `SwiftUI` | Always | `-framework SwiftUI` |
| `AppKit` | Always (menubar, popover) | `-framework AppKit` |
| `Foundation` | Always (Process, Pipe, Timer) | (auto-linked) |
| `Darwin` | System calls (sysctl, mach APIs, proc_pid_rusage) | (auto-linked) |
| `IOKit` | GPU stats, hardware info via ioreg | `-framework IOKit` |
| `CoreWLAN` | WiFi signal, SSID, BSSID, channel | `-framework CoreWLAN` |
| `SystemConfiguration` | Network reachability, DNS config | `-framework SystemConfiguration` |
| `DiskArbitration` | Disk mount/unmount events | `-framework DiskArbitration` |

The build command must include `-framework` flags for every non-auto-linked framework you import.

## macOS Permissions

Some data sources require user permission. The app will still compile and run, but may return empty data without these:

| Data | Permission needed | How it manifests |
|------|------------------|-----------------|
| WiFi SSID/BSSID | Location Services (macOS Ventura+) | Returns nil/empty without permission |
| Accessibility (window titles) | Accessibility permission | API calls fail silently |
| Screen recording | Screen capture permission | Screenshots fail |
| Microphone/Camera | Audio/Video permission | Access denied |
| Full Disk Access | FDA permission | Can't read some paths |

For WiFi apps, print a note to stderr on launch if SSID comes back empty, suggesting the user grant Location Services permission. For most system monitoring (CPU, memory, GPU, disk, network bandwidth, processes), no special permissions are required.

## Workflow

When building a menubar app, follow these steps:

### 1. Understand what data to display

Ask the user what information they want in their menubar app. Common categories:
- **System monitoring**: CPU, memory, GPU, disk, battery, temperature
- **Network**: bandwidth, connections, DNS, latency, WiFi signal
- **Process info**: top processes by memory/CPU, specific app status
- **External data**: API responses, weather, stock prices, build status
- **Custom**: anything that can be fetched via shell command or HTTP

### 2. Design the data model

Create simple structs for your domain data. Keep them flat — no deep nesting.

```swift
struct MyStats {
    let value: Double
    let label: String
    var fraction: Double { value / max(total, 1) }
}
```

### 3. Build the data collection layer

Use the Process/Pipe pattern to run shell commands and parse their output. See `references/data-collection.md` for the complete pattern. The key principle: wrap macOS CLI tools (like `nettop`, `ioreg`, `ps`, `sysctl`, `lsof`, `system_profiler`, `diskutil`, `pmset`) in Swift functions that return your model structs.

### 4. Create the Observable Monitor

This is the reactive heart of the app. It uses `ObservableObject` with `@Published` properties and `Timer` for periodic updates. Always dispatch data collection to a background queue and update UI on the main queue.

```swift
class MyMonitor: ObservableObject {
    @Published var stats = MyStats(...)
    @Published var history: [Double] = []
    private var timer: Timer?
    private let maxHistory = 60

    init() {
        refresh()
        timer = Timer.scheduledTimer(withTimeInterval: 2.0, repeats: true) { [weak self] _ in
            self?.refresh()
        }
    }

    deinit { timer?.invalidate() }

    func refresh() {
        DispatchQueue.global(qos: .utility).async { [weak self] in
            let data = fetchMyData()  // your data collection function
            DispatchQueue.main.async {
                guard let self = self else { return }
                self.stats = data
                self.history.append(data.value)
                if self.history.count > self.maxHistory { self.history.removeFirst() }
            }
        }
    }
}
```

**Dual-speed polling**: Use a fast timer (2s) for lightweight stats and a slower timer (5s) for expensive operations like process enumeration.

### 5. Choose the layout

Pick the layout based on how much data you're displaying:

**Two-column** (840x700) — Use when you have both summary metrics AND a detailed list (e.g., CPU overview + process list, bandwidth overview + per-process breakdown). Put the summary/charts in the left column and the scrollable list in the right.

**Single-column** (400-500 wide, 500-600 tall) — Use when you have a focused set of metrics without a long list (e.g., disk volumes, battery status, weather). Simpler apps don't need the complexity of two columns.

The decision heuristic: if you have a scrollable list of 10+ items alongside summary cards, go two-column. Otherwise, single-column is cleaner.

### 6. Build the UI

Follow these design principles extracted from high-quality menubar apps:

**Typography**:
- `.monospaced` design for all numerical data (rates, bytes, percentages, PIDs)
- `.system(size: 20, weight: .bold, design: .monospaced)` for headline metrics
- `.system(size: 12-13, weight: .semibold)` for section headers
- `.system(size: 10-11)` for secondary/detail text

**Color coding**: Assign semantic colors to different data types. Blue for downloads/input, orange for uploads/output, green for good/available, red for critical, purple for process bars. Use `.opacity(0.04-0.15)` for subtle backgrounds.

**Components to use** (all defined in `references/ui-components.md`):
- `SparklineView` — real-time line chart with filled area
- `RateCardView` — large colored metric card with icon, value, and subtitle
- `UsageBarView` — segmented horizontal bar showing proportions
- `SectionHeader` — consistent icon + title header
- `SortButton` — clickable column header with active state
- `BarView` — thin proportional bar for inline comparisons

**Loading states**: Always show `ProgressView()` with a message during initial data sampling.

**Quit button**: Always include a quit button at the bottom of the popover. Users need a way to exit a menubar-only app:

```swift
HStack {
    Spacer()
    Button("Quit") {
        NSApplication.shared.terminate(nil)
    }
    .buttonStyle(.plain)
    .font(.system(size: 11))
    .foregroundColor(.secondary)
}
.padding(.top, 8)
```

### 7. Consider dynamic menubar text

Beyond just an icon, you can show a live value in the menu bar itself. This is powerful for the most important metric (CPU %, download speed, temperature):

```swift
// In AppDelegate, create with variable length instead of square:
statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)

// Update the button title from your monitor:
func updateMenuBarTitle(_ value: String) {
    DispatchQueue.main.async {
        self.statusItem.button?.title = value  // e.g., "42%"
    }
}
```

Use this when there's a single dominant metric the user wants to glance at without opening the popover. Keep the text short (2-5 characters). If you use dynamic text, you can combine it with an icon: set both `button.image` and `button.title`.

### 8. Wire up the menubar

Use the AppDelegate pattern from `references/architecture.md`. Choose an appropriate SF Symbol for the menu bar icon. The app delegate creates an `NSStatusItem`, configures an `NSPopover` with `.transient` behavior, and hides the app from the Dock with `NSApp.setActivationPolicy(.accessory)`.

### 9. Build, compile, and verify

```bash
swiftc -parse-as-library \
    -framework SwiftUI \
    -framework AppKit \
    -o MyApp MyAppApp.swift
```

Add additional `-framework` flags as needed (see the framework table above).

**Always compile the app after writing it** and fix any errors before finishing. The `-parse-as-library` flag is required because the file uses `@main`. If compilation fails, read the error, fix the Swift file, and recompile. Do not deliver uncompiled code.

After successful compilation, briefly describe how to run it: `./MyApp`

## Design Guidelines

These are the design patterns that make menubar apps feel polished rather than generic:

1. **Monospaced numbers everywhere** — numerical data should always use `.design(.monospaced)` so columns align and numbers don't jump as they update.

2. **Subtle colored backgrounds** — use `color.opacity(0.04-0.08)` for section backgrounds rather than heavy borders. This creates visual hierarchy without noise.

3. **SF Symbols for all icons** — never use emoji or text characters for icons. SF Symbols adapt to the system theme and look native.

4. **Material backgrounds for overlays** — use `.ultraThinMaterial` for legends overlaid on charts.

5. **Conditional information density** — show more detail for active/interesting items, less for idle ones. If a process has zero bandwidth, dim it. If swap is not in use, hide the swap warning.

6. **Color-status mapping** — green (>30%) = good, orange (15-30%) = caution, red (<15%) = critical. Apply this to any metric with a healthy/unhealthy range.

7. **History visualization** — always include sparkline charts showing recent history (60 samples at the polling interval gives a few minutes of context).

## Common Pitfalls

- **Forgetting `-parse-as-library`** — the `swiftc` command will fail without this flag when using `@main`.
- **Missing `-framework` flags** — if you import CoreWLAN, IOKit, etc., you must add the corresponding `-framework` flag to the build command.
- **Blocking the main thread** — always dispatch shell commands to `DispatchQueue.global(qos: .utility)` and update `@Published` properties on `DispatchQueue.main`.
- **Not using `[weak self]`** — timer callbacks and async closures must capture `self` weakly to avoid retain cycles.
- **Forgetting `deinit`** — always invalidate timers in `deinit`.
- **Not capping history arrays** — use `removeFirst()` when history exceeds `maxHistory` to prevent unbounded memory growth.
- **Missing `.accessory` activation policy** — without `NSApp.setActivationPolicy(.accessory)`, the app will show in the Dock.
- **Using MenuBarExtra instead of NSStatusItem** — `MenuBarExtra` (SwiftUI native) is less flexible. The `NSStatusItem + NSPopover` pattern gives you full control over popover size, behavior, and keyboard focus.
- **No quit button** — menubar-only apps have no Dock icon to right-click quit from. Always include a quit button in the popover.
- **Not compiling before delivering** — always run `swiftc` and verify the app compiles cleanly.
