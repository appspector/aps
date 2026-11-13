---
name: check-performance
description: Check the app's performance with AppSpector — FPS, CPU, memory, network, battery and the worst moments. Use when the app is slow, janky, hot or drains the battery.
---

# Check performance

If the AppSpector tools (`status`, `http_requests`, …) aren't available, or the `appspector` MCP server failed to start, follow the `setup` skill first.

1. **Summarize.** Call `performance` for the session, with `since`/`until` around the problem if the user knows when it happened. It returns min, average, p95, max and latest for FPS, CPU %, memory and network speed, the battery, and the moments of highest CPU and memory and lowest FPS.
2. **Look at the worst moments.** For each, list what happened around it: `http_requests` and `logs` with `since`/`until` a few seconds either side. Large responses, bursts of requests, many log lines or an image or database load often line up with CPU spikes and dropped frames.
3. **Live:** if the session is live, ask the user to repeat the slow action, then call `performance` again with `since: "1m"` and compare. A `screenshot` shows which screen they're on.
4. **Report** the numbers that are out of line (FPS well under 60, CPU near 100 %, memory that keeps growing), when they happened, and what the app was doing then.

Times come from the device's clock. If `performance` has no samples, the performance monitor may be off for this app.
