# Data Collection Reference

Patterns for gathering system data in menubar apps. All collection runs on background threads.

## Shell Command Pattern

The foundation for most data collection — run a CLI tool, capture output, parse it:

```swift
func runCommand(_ path: String, arguments: [String] = []) -> String? {
    let pipe = Pipe()
    let proc = Process()
    proc.executableURL = URL(fileURLWithPath: path)
    proc.arguments = arguments
    proc.standardOutput = pipe
    proc.standardError = FileHandle.nullDevice
    do { try proc.run() } catch { return nil }
    let data = pipe.fileHandleForReading.readDataToEndOfFile()
    proc.waitUntilExit()
    guard proc.terminationStatus == 0 else { return nil }
    return String(data: data, encoding: .utf8)
}
```

**With error capture** (for surfacing errors in the UI):

```swift
func runCommandWithError(_ path: String, arguments: [String]) -> (output: String?, error: String?) {
    let outPipe = Pipe()
    let errPipe = Pipe()
    let proc = Process()
    proc.executableURL = URL(fileURLWithPath: path)
    proc.arguments = arguments
    proc.standardOutput = outPipe
    proc.standardError = errPipe
    do { try proc.run() } catch {
        return (nil, "Failed to start: \(error.localizedDescription)")
    }
    let outData = outPipe.fileHandleForReading.readDataToEndOfFile()
    let errData = errPipe.fileHandleForReading.readDataToEndOfFile()
    proc.waitUntilExit()
    let stderr = String(data: errData, encoding: .utf8)?
        .trimmingCharacters(in: .whitespacesAndNewlines) ?? ""
    guard proc.terminationStatus == 0 else {
        return (nil, stderr.isEmpty ? "Exit status \(proc.terminationStatus)" : stderr)
    }
    return (String(data: outData, encoding: .utf8), nil)
}
```

## Common macOS Data Sources

| Data | Command | Key Flags |
|------|---------|-----------|
| Network bandwidth | `/usr/bin/nettop` | `-d -P -L 2 -s 1 -x -n -J bytes_in,bytes_out` |
| TCP connections | `/usr/sbin/lsof` | `-n -P -iTCP` |
| GPU stats (Apple Silicon) | `/usr/sbin/ioreg` | `-r -c AGXAccelerator -d 2` |
| Process list | `/bin/ps` | `-eo pid,rss,pcpu,comm -r` |
| Memory pressure | `/usr/bin/memory_pressure` | (no args) |
| Disk usage | Use `FileManager.mountedVolumeURLs` + `URL.resourceValues` | (no shell needed) |
| Battery info | `/usr/bin/pmset` | `-g batt` |
| System info | `/usr/sbin/system_profiler` | `SPHardwareDataType` |
| WiFi info | Use `CWWiFiClient.shared()` from CoreWLAN | (no shell needed) |
| CPU stats | Use `host_processor_info` Mach API | (no shell needed) |
| Network devices (ARP) | `/usr/sbin/arp` | `-a` |
| Load averages | Use `getloadavg()` | (no shell needed) |
| Network byte counters | `/usr/sbin/netstat` | `-I en0 -b` |
| DNS servers | `/usr/sbin/scutil` | `--dns` |
| Uptime | `/usr/bin/uptime` | (no args) |

When a native Swift/Mach API exists (marked "no shell needed"), prefer it over shelling out — it's faster and more reliable.

## String Parsing Patterns

### Extract value after a known prefix
```swift
if let range = output.range(of: "free percentage: ") {
    let after = output[range.upperBound...]
    if let end = after.firstIndex(of: "%"),
       let value = Int(after[after.startIndex..<end]) {
        // use value
    }
}
```

### Split process name from PID ("Safari.12345")
```swift
var name = field
if let dot = field.range(of: ".", options: .backwards),
   Int(field[dot.upperBound...]) != nil {
    name = String(field[..<dot.lowerBound])
}
```

### Classify IP addresses as local/internet
```swift
func isLocalAddress(_ host: String) -> Bool {
    host == "localhost" || host == "::1" ||
    host.hasPrefix("10.") || host.hasPrefix("127.") ||
    host.hasPrefix("192.168.") || host.hasPrefix("169.254.") ||
    host.hasPrefix("fe80:") || host.hasPrefix("fc") || host.hasPrefix("fd") ||
    (host.hasPrefix("172.") && {
        let parts = host.split(separator: ".")
        return parts.count >= 2 && (16...31).contains(Int(parts[1]) ?? 0)
    }())
}
```

## Process Aggregation

Group multiple PIDs of the same app into one row:

```swift
var aggregated: [String: (totalMB: Double, totalCPU: Double, count: Int)] = [:]
for (name, mb, cpu, _) in processes {
    var entry = aggregated[name] ?? (0, 0, 0)
    entry.totalMB += mb
    entry.totalCPU += cpu
    entry.count += 1
    aggregated[name] = entry
}
// Display as "node (10)" when count > 1
```
