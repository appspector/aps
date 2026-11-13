---
name: find-errors
description: Find errors and warnings in the app's logs around a failure with AppSpector, and tie them to requests and user actions. Use when the app shows an error, crashes or misbehaves.
---

# Find errors in the logs

If the AppSpector tools (`status`, `http_requests`, …) aren't available, or the `appspector` MCP server failed to start, follow the `setup` skill first.

1. **Start narrow.** Call `logs` with `level: "error"` (or `"warning"`) and `since` around when the problem happened. Many apps log everything at `verbose` or `debug`, so if levels find nothing, search by text instead: `text` matches the message or the tag (an exception name, `Error`, `failed`, an HTTP status, a screen name); `regex` for patterns such as `(?i)timeout|offline`.
2. **Widen around the moment.** Once you have a line, list everything a few seconds either side of it (`since`/`until` with ISO times, no level filter) to see what led up to it. `tag` and `source` (`user` for the app's logger, `system` for stdout and `NSLog`) narrow it down.
3. **Connect it.** Check `http_requests` for the same time window (`failed: true`): many errors follow a failed request. Check `events` with `summary: true` for other monitors (custom events, analytics, notifications) that mark user actions.
4. **Report** the error lines with their times, what happened just before, and the likely cause. Say when you're inferring.

If the session is live and the error hasn't happened yet, see the `reproduce-bug` skill. Times come from the device's clock.
