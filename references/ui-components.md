# UI Components Reference

Reusable SwiftUI components for menubar app popovers. Copy these into your app and use them as building blocks.

## Table of Contents
1. [SparklineView](#sparklineview) — Real-time line chart
2. [RateCardView](#ratecardview) — Large colored metric card
3. [UsageBarView](#usagebarview) — Segmented horizontal bar
4. [SectionHeader](#sectionheader) — Icon + title header
5. [SortButton](#sortbutton) — Clickable column sort header
6. [BarView](#barview) — Thin proportional bar
7. [ProcessRowView](#processrowview) — Process list row with bar
8. [StatItem](#statitem) — Label + value pair
9. [Layout Patterns](#layout-patterns) — Two-column popover layout

---

## SparklineView

A responsive line chart with a filled area underneath. Adapts to any container size via `GeometryReader`.

```swift
struct SparklineView: View {
    let data: [Double]
    let color: Color
    let maxValue: Double?

    init(data: [Double], color: Color, maxValue: Double? = nil) {
        self.data = data
        self.color = color
        self.maxValue = maxValue
    }

    var body: some View {
        GeometryReader { geo in
            let maxVal = maxValue ?? max((data.max() ?? 1), 0.001)
            let w = geo.size.width
            let h = geo.size.height

            if data.count > 1 {
                // Stroke line
                Path { path in
                    for (i, val) in data.enumerated() {
                        let x = w * CGFloat(i) / CGFloat(data.count - 1)
                        let y = h - (h * CGFloat(val / maxVal))
                        if i == 0 { path.move(to: CGPoint(x: x, y: y)) }
                        else { path.addLine(to: CGPoint(x: x, y: y)) }
                    }
                }
                .stroke(color, lineWidth: 1.5)

                // Fill area under the line
                Path { path in
                    path.move(to: CGPoint(x: 0, y: h))
                    for (i, val) in data.enumerated() {
                        let x = w * CGFloat(i) / CGFloat(data.count - 1)
                        let y = h - (h * CGFloat(val / maxVal))
                        path.addLine(to: CGPoint(x: x, y: y))
                    }
                    path.addLine(to: CGPoint(x: w, y: h))
                    path.closeSubpath()
                }
                .fill(color.opacity(0.15))
            }
        }
    }
}
```

**Usage:**
```swift
SparklineView(data: monitor.history, color: .blue, maxValue: 100.0)
    .frame(height: 60)
```

**Overlaying multiple sparklines** (e.g., download + upload):
```swift
ZStack {
    SparklineView(data: downloadHistory, color: .blue)
    SparklineView(data: uploadHistory, color: .orange)
}
.frame(height: 80)
```

Add a legend overlay with `.ultraThinMaterial`:
```swift
ZStack(alignment: .topTrailing) {
    // ... sparklines ...
    VStack(alignment: .trailing, spacing: 2) {
        HStack(spacing: 4) {
            RoundedRectangle(cornerRadius: 1).fill(.blue).frame(width: 12, height: 2)
            Text("Download").font(.system(size: 9)).foregroundColor(.secondary)
        }
        HStack(spacing: 4) {
            RoundedRectangle(cornerRadius: 1).fill(.orange).frame(width: 12, height: 2)
            Text("Upload").font(.system(size: 9)).foregroundColor(.secondary)
        }
    }
    .padding(4)
    .background(.ultraThinMaterial)
    .cornerRadius(4)
}
```

---

## RateCardView

A prominent card displaying a single metric with icon, value, and optional subtitle. Uses a colored background tint.

```swift
struct RateCardView: View {
    let title: String
    let value: String
    let subtitle: String
    let icon: String
    let color: Color

    init(title: String, value: String, subtitle: String = "", icon: String, color: Color) {
        self.title = title
        self.value = value
        self.subtitle = subtitle
        self.icon = icon
        self.color = color
    }

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            HStack(spacing: 4) {
                Image(systemName: icon)
                    .foregroundColor(color)
                    .font(.system(size: 11))
                Text(title)
                    .font(.system(size: 11, weight: .medium))
                    .foregroundColor(.secondary)
            }
            Text(value)
                .font(.system(size: 20, weight: .bold, design: .monospaced))
                .foregroundColor(color)
            if !subtitle.isEmpty {
                Text(subtitle)
                    .font(.system(size: 10))
                    .foregroundColor(.secondary)
            }
        }
        .frame(maxWidth: .infinity, alignment: .leading)
        .padding(10)
        .background(color.opacity(0.08))
        .cornerRadius(8)
    }
}
```

**Usage:**
```swift
HStack(spacing: 10) {
    RateCardView(title: "DOWNLOAD", value: "12.4 MB/s",
                 icon: "arrow.down.circle.fill", color: .blue)
    RateCardView(title: "UPLOAD", value: "1.2 MB/s",
                 icon: "arrow.up.circle.fill", color: .orange)
}
```

---

## UsageBarView

A segmented horizontal bar showing multiple proportions. Great for memory breakdowns, disk usage, etc.

```swift
struct UsageBarView: View {
    let segments: [(Double, Color)]  // (fraction 0-1, color)
    let height: CGFloat

    init(segments: [(Double, Color)], height: CGFloat = 20) {
        self.segments = segments
        self.height = height
    }

    var body: some View {
        GeometryReader { geo in
            ZStack(alignment: .leading) {
                RoundedRectangle(cornerRadius: 4)
                    .fill(Color.primary.opacity(0.08))
                HStack(spacing: 0) {
                    ForEach(Array(segments.enumerated()), id: \.offset) { _, seg in
                        Rectangle()
                            .fill(seg.1)
                            .frame(width: max(0, geo.size.width * CGFloat(min(seg.0, 1.0))))
                    }
                }
                .clipShape(RoundedRectangle(cornerRadius: 4))
            }
        }
        .frame(height: height)
    }
}
```

**Usage:**
```swift
UsageBarView(segments: [
    (0.4, .green),     // GPU active
    (0.2, .green.opacity(0.3)),  // GPU mapped
    (0.25, .blue.opacity(0.6)),  // Apps & OS
    // Remaining space is the empty background
], height: 36)
```

---

## SectionHeader

Consistent section title with an SF Symbol icon.

```swift
struct SectionHeader: View {
    let title: String
    let icon: String

    var body: some View {
        HStack(spacing: 6) {
            Image(systemName: icon)
                .font(.system(size: 12, weight: .semibold))
            Text(title)
                .font(.system(size: 13, weight: .semibold))
        }
        .foregroundColor(.primary)
    }
}
```

**Usage:**
```swift
SectionHeader(title: "Bandwidth (last 60s)", icon: "chart.xyaxis.line")
SectionHeader(title: "Internet Destinations", icon: "mappin.and.ellipse")
SectionHeader(title: "Memory Footprint", icon: "cpu")
```

---

## SortButton

Clickable column header that toggles sort direction. Shows a chevron indicator when active.

```swift
struct SortButton<T: Hashable>: View {
    let label: String
    let key: T
    @Binding var currentKey: T
    @Binding var ascending: Bool
    let action: () -> Void

    var body: some View {
        Button(action: {
            if currentKey == key {
                ascending.toggle()
            } else {
                currentKey = key
                ascending = false
            }
            action()
        }) {
            HStack(spacing: 2) {
                Text(label)
                    .font(.system(size: 10, weight: currentKey == key ? .bold : .medium))
                if currentKey == key {
                    Image(systemName: ascending ? "chevron.up" : "chevron.down")
                        .font(.system(size: 8))
                }
            }
            .foregroundColor(currentKey == key ? .primary : .secondary)
        }
        .buttonStyle(.plain)
    }
}
```

**Usage:**
```swift
enum SortKey: String, CaseIterable {
    case memory = "Memory"
    case cpu = "CPU"
    case name = "Name"
}

HStack(spacing: 8) {
    Text("Sort:").font(.system(size: 10)).foregroundColor(.secondary)
    ForEach(SortKey.allCases, id: \.self) { key in
        SortButton(label: key.rawValue, key: key,
                   currentKey: $monitor.sortKey,
                   ascending: $monitor.sortAscending,
                   action: { monitor.resort() })
    }
}
```

---

## BarView

A thin proportional bar for inline metric comparison within list rows.

```swift
struct BarView: View {
    let fraction: Double
    let color: Color

    var body: some View {
        GeometryReader { geo in
            ZStack(alignment: .leading) {
                RoundedRectangle(cornerRadius: 2)
                    .fill(color.opacity(0.1))
                RoundedRectangle(cornerRadius: 2)
                    .fill(color.opacity(0.5))
                    .frame(width: max(0, geo.size.width * CGFloat(min(fraction, 1.0))))
            }
        }
        .frame(height: 4)
    }
}
```

---

## ProcessRowView

A list row for process data with name, value, and a proportional bar.

```swift
struct ProcessRowView: View {
    let name: String
    let value: String
    let fraction: Double
    let barColor: Color

    var body: some View {
        VStack(spacing: 3) {
            HStack {
                Text(name)
                    .font(.system(size: 12, weight: .medium, design: .monospaced))
                    .lineLimit(1)
                Spacer()
                Text(value)
                    .font(.system(size: 12, weight: .bold, design: .monospaced))
            }
            BarView(fraction: fraction, color: barColor)
        }
        .padding(.vertical, 3)
    }
}
```

---

## StatItem

A compact label + value pair for stat grids.

```swift
struct StatItem: View {
    let label: String
    let value: String
    let color: Color

    var body: some View {
        VStack(alignment: .leading, spacing: 2) {
            Text(label)
                .font(.system(size: 10, weight: .medium))
                .foregroundColor(.secondary)
            Text(value)
                .font(.system(size: 14, weight: .bold, design: .monospaced))
                .foregroundColor(color)
        }
    }
}
```

**Usage:**
```swift
HStack(spacing: 16) {
    StatItem(label: "Active", value: "4.2 GB", color: .blue)
    StatItem(label: "Wired", value: "2.1 GB", color: .orange)
    StatItem(label: "Compressed", value: "1.3 GB", color: .purple)
}
```

---

## Layout Patterns

### Two-column popover

The standard layout for information-rich menubar apps:

```swift
struct ContentView: View {
    @StateObject private var monitor = MyMonitor()

    var body: some View {
        HStack(alignment: .top, spacing: 0) {
            // Left column — overview, charts, summary
            ScrollView {
                VStack(alignment: .leading, spacing: 14) {
                    // Header
                    Text("My App")
                        .font(.system(size: 20, weight: .bold))

                    // Headline metric
                    // Rate cards
                    // Sparklines
                    // Breakdown sections
                }
                .padding(16)
            }
            .frame(width: 500)  // or minWidth for flexibility

            Divider()

            // Right column — detailed list (processes, connections, etc.)
            VStack(alignment: .leading, spacing: 8) {
                SectionHeader(title: "Details", icon: "list.bullet")
                // Sort controls
                ScrollView {
                    LazyVStack(spacing: 0) {
                        ForEach(monitor.items) { item in
                            // Row view
                            Divider()
                        }
                    }
                }
            }
            .padding(12)
            .frame(width: 340)
        }
        .frame(width: 840, height: 700)
        .background(.background)
    }
}
```

### Single-column popover

For simpler apps with less data:

```swift
struct ContentView: View {
    @StateObject private var monitor = MyMonitor()

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 14) {
                // All content in one column
            }
            .padding(16)
        }
        .frame(width: 400, height: 500)
        .background(.background)
    }
}
```

### Headline metric pattern

For the main "at a glance" number:

```swift
VStack(alignment: .leading, spacing: 6) {
    HStack(alignment: .firstTextBaseline, spacing: 6) {
        Text(String(format: "%.0f", value))
            .font(.system(size: 48, weight: .bold, design: .rounded))
            .foregroundColor(statusColor)
        Text("GB Available")
            .font(.system(size: 18, weight: .semibold))
            .foregroundColor(statusColor.opacity(0.8))
    }
    Text("of \(total) total")
        .font(.system(size: 12))
        .foregroundColor(.secondary)
}
.padding(16)
.frame(maxWidth: .infinity, alignment: .leading)
.background(statusColor.opacity(0.06))
.cornerRadius(12)
```
