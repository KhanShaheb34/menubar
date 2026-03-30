# Architecture Reference

## Table of Contents
1. [Imports](#imports)
2. [App Entry Point](#app-entry-point)
3. [AppDelegate (Menubar Integration)](#appdelegate)
4. [Observable Monitor Pattern](#observable-monitor-pattern)
5. [Formatting Helpers](#formatting-helpers)
6. [Build Command](#build-command)
7. [Complete Skeleton](#complete-skeleton)

---

## Imports

Every menubar app needs at minimum:

```swift
import SwiftUI
import AppKit
import Foundation
```

Add domain-specific imports as needed — see the Framework Imports table in SKILL.md.

---

## App Entry Point

The `@main` entry point bridges SwiftUI to AppKit via `@NSApplicationDelegateAdaptor`. The `Settings` scene with `EmptyView()` is required — SwiftUI needs at least one scene, but the actual UI lives in the popover.

```swift
@main
struct MyApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) var appDelegate

    var body: some Scene {
        Settings {
            EmptyView()
        }
    }
}
```

---

## AppDelegate

The AppDelegate manages the menubar icon and popover lifecycle. This is the exact pattern used in production menubar apps:

```swift
class AppDelegate: NSObject, NSApplicationDelegate {
    var statusItem: NSStatusItem!
    var popover: NSPopover!

    func applicationDidFinishLaunching(_ notification: Notification) {
        // 1. Create status bar item
        statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.squareLength)

        // 2. Configure the button with an SF Symbol
        if let button = statusItem.button {
            button.image = NSImage(
                systemSymbolName: "chart.bar.fill",  // Choose an appropriate SF Symbol
                accessibilityDescription: "My App"
            )
            button.action = #selector(togglePopover)
            button.target = self
        }

        // 3. Create and configure the popover
        let popover = NSPopover()
        popover.contentSize = NSSize(width: 840, height: 700)  // Adjust to your needs
        popover.behavior = .transient  // Auto-dismiss when clicking outside
        popover.contentViewController = NSHostingController(rootView: ContentView())
        self.popover = popover

        // 4. Hide from Dock — app lives only in the menu bar
        NSApp.setActivationPolicy(.accessory)
    }

    @objc func togglePopover() {
        guard let button = statusItem.button else { return }
        if popover.isShown {
            popover.performClose(nil)
        } else {
            popover.show(relativeTo: button.bounds, of: button, preferredEdge: .minY)
            NSApp.activate(ignoringOtherApps: true)
            popover.contentViewController?.view.window?.makeKey()
        }
    }
}
```

### Key details:

- **`NSStatusItem.squareLength`** — creates a standard-width menu bar icon
- **`.transient` behavior** — popover closes when the user clicks outside it
- **`.minY` edge** — popover appears below the menu bar icon
- **`NSApp.activate(ignoringOtherApps: true)`** — brings the popover to front
- **`.makeKey()`** — ensures the popover receives keyboard focus
- **`.accessory` activation policy** — hides the app from the Dock and Cmd-Tab

### Choosing an SF Symbol for the icon

Pick a symbol that represents your app's purpose:
- Network monitoring: `"arrow.up.arrow.down"`, `"network"`, `"wifi"`
- Memory/GPU: `"memorychip"`, `"cpu"`, `"gauge.with.dots.needle.bottom.50percent"`
- Disk: `"internaldrive"`, `"externaldrive"`
- Battery: `"battery.100percent"`, `"bolt.fill"`
- Weather: `"cloud.sun"`, `"thermometer"`
- General monitoring: `"chart.bar.fill"`, `"gauge.with.dots.needle.bottom.50percent"`
- Clock/time: `"clock"`, `"timer"`

---

## Observable Monitor Pattern

The monitor is the reactive core of the app. It collects data on background threads and publishes changes that SwiftUI automatically picks up.

```swift
class MyMonitor: ObservableObject {
    // Published state that drives the UI
    @Published var primaryMetric: Double = 0
    @Published var items: [MyItem] = []
    @Published var history: [Double] = []
    @Published var errorMessage: String?

    // Timers for periodic updates
    private var fastTimer: Timer?
    private var slowTimer: Timer?
    private let maxHistory = 60  // ~2 minutes at 2s intervals

    init() {
        // Fetch immediately on launch
        refreshFast()
        refreshSlow()

        // Fast timer: lightweight stats every 2 seconds
        fastTimer = Timer.scheduledTimer(withTimeInterval: 2.0, repeats: true) { [weak self] _ in
            self?.refreshFast()
        }
        // Slow timer: expensive operations every 5 seconds
        slowTimer = Timer.scheduledTimer(withTimeInterval: 5.0, repeats: true) { [weak self] _ in
            self?.refreshSlow()
        }
    }

    deinit {
        fastTimer?.invalidate()
        slowTimer?.invalidate()
    }

    func refreshFast() {
        DispatchQueue.global(qos: .utility).async { [weak self] in
            let data = collectLightweightData()
            DispatchQueue.main.async {
                guard let self = self else { return }
                self.primaryMetric = data.value
                self.errorMessage = nil

                // Maintain rolling history
                self.history.append(data.value)
                if self.history.count > self.maxHistory {
                    self.history.removeFirst()
                }
            }
        }
    }

    func refreshSlow() {
        DispatchQueue.global(qos: .utility).async { [weak self] in
            let items = collectExpensiveData()
            DispatchQueue.main.async {
                self?.items = items
            }
        }
    }
}
```

### Using the monitor in ContentView:

```swift
struct ContentView: View {
    @StateObject private var monitor = MyMonitor()

    var body: some View {
        // Use monitor.primaryMetric, monitor.items, etc.
    }
}
```

`@StateObject` creates the monitor once and keeps it alive for the popover's lifetime.

---

## Formatting Helpers

Standard formatters for common data types:

```swift
// Bytes (memory, disk, file sizes)
func formatBytes(_ bytes: UInt64) -> String {
    let gb = Double(bytes) / 1_073_741_824
    if gb >= 1.0 { return String(format: "%.1f GB", gb) }
    let mb = Double(bytes) / 1_048_576
    if mb >= 1.0 { return String(format: "%.0f MB", mb) }
    return String(format: "%.0f KB", Double(bytes) / 1024)
}

// Byte rates (bandwidth)
func formatBytesRate(_ bps: Double) -> String {
    if bps >= 1_073_741_824 { return String(format: "%.2f GB/s", bps / 1_073_741_824) }
    if bps >= 1_048_576 { return String(format: "%.1f MB/s", bps / 1_048_576) }
    if bps >= 1024 { return String(format: "%.1f KB/s", bps / 1024) }
    return String(format: "%.0f B/s", bps)
}

// Megabytes (process memory)
func formatMB(_ mb: Double) -> String {
    if mb >= 1024 { return String(format: "%.1f GB", mb / 1024) }
    return String(format: "%.0f MB", mb)
}

// Percentages
func formatPercent(_ fraction: Double) -> String {
    return String(format: "%.0f%%", fraction * 100)
}

// Duration
func formatDuration(_ seconds: Int) -> String {
    if seconds >= 3600 { return String(format: "%dh %dm", seconds / 3600, (seconds % 3600) / 60) }
    if seconds >= 60 { return String(format: "%dm %ds", seconds / 60, seconds % 60) }
    return "\(seconds)s"
}
```

---

## Build Command

Compile the app with:

```bash
swiftc -parse-as-library \
    -framework SwiftUI \
    -framework AppKit \
    -o AppName AppNameApp.swift
```

Add additional `-framework` flags for any domain-specific frameworks (see SKILL.md table).

The `-parse-as-library` flag is required because the file uses `@main` attribute.

**Always compile before delivering.** If it fails, fix the errors and recompile.

To run: `./AppName` — the icon appears in the menu bar.

---

## Complete Skeleton

Here's a minimal but complete menubar app skeleton that compiles and runs:

```swift
import SwiftUI
import AppKit
import Foundation

// MARK: - Data Model

struct AppData {
    let value: Double
    let label: String
}

// MARK: - Data Collection

func fetchData() -> AppData {
    // Replace with your actual data collection
    return AppData(value: Double.random(in: 0...100), label: "Sample")
}

// MARK: - Monitor

class AppMonitor: ObservableObject {
    @Published var data = AppData(value: 0, label: "Loading...")
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
            let result = fetchData()
            DispatchQueue.main.async {
                guard let self = self else { return }
                self.data = result
                self.history.append(result.value)
                if self.history.count > self.maxHistory { self.history.removeFirst() }
            }
        }
    }
}

// MARK: - Views

struct ContentView: View {
    @StateObject private var monitor = AppMonitor()

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            Text("My App")
                .font(.system(size: 20, weight: .bold))

            Text(String(format: "%.1f", monitor.data.value))
                .font(.system(size: 36, weight: .bold, design: .monospaced))

            Text(monitor.data.label)
                .font(.system(size: 12))
                .foregroundColor(.secondary)

            Spacer()

            HStack {
                Spacer()
                Button("Quit") { NSApplication.shared.terminate(nil) }
                    .buttonStyle(.plain)
                    .font(.system(size: 11))
                    .foregroundColor(.secondary)
            }
        }
        .padding(20)
        .frame(width: 300, height: 220)
        .background(.background)
    }
}

// MARK: - App Delegate

class AppDelegate: NSObject, NSApplicationDelegate {
    var statusItem: NSStatusItem!
    var popover: NSPopover!

    func applicationDidFinishLaunching(_ notification: Notification) {
        statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.squareLength)
        if let button = statusItem.button {
            button.image = NSImage(systemSymbolName: "chart.bar.fill",
                                   accessibilityDescription: "My App")
            button.action = #selector(togglePopover)
            button.target = self
        }

        let popover = NSPopover()
        popover.contentSize = NSSize(width: 300, height: 200)
        popover.behavior = .transient
        popover.contentViewController = NSHostingController(rootView: ContentView())
        self.popover = popover

        NSApp.setActivationPolicy(.accessory)
    }

    @objc func togglePopover() {
        guard let button = statusItem.button else { return }
        if popover.isShown {
            popover.performClose(nil)
        } else {
            popover.show(relativeTo: button.bounds, of: button, preferredEdge: .minY)
            NSApp.activate(ignoringOtherApps: true)
            popover.contentViewController?.view.window?.makeKey()
        }
    }
}

// MARK: - Entry Point

@main
struct MyApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    var body: some Scene {
        Settings { EmptyView() }
    }
}
```

Build and run:
```bash
swiftc -parse-as-library -framework SwiftUI -framework AppKit -o MyApp MyApp.swift && ./MyApp
```
