---
name: reproduce-bug
description: Reproduce a bug live with AppSpector — wait for a session, then for the failing request or log line while the user repeats the steps, with a screenshot. Use when a bug needs reproducing.
---

# Reproduce a bug live

If the AppSpector tools (`status`, `http_requests`, …) aren't available, or the `appspector` MCP server failed to start, follow the `setup` skill first.

1. **A live session.** `status` lists the app's live sessions. If there's none, ask the user to start the app (on a device or simulator, with the AppSpector SDK) and call `wait_for_session` (`new_only: true` after a relaunch). With several devices, pass `device`.
2. **Know what to wait for.** From the user's description, pick one: an HTTP response (`type: "http"` with `failed: true`, `status: "5xx"`, `host`, `url`), a log line (`type: "log"` with `level: "error"`, `text`, `regex`), or another event (`type: "event"` with `kind`, `text`).
3. **Wait, then let the user act.** Call `wait_for_event` with that filter and a `timeout_seconds` long enough for the steps (60 by default, up to 600), and tell the user to reproduce the bug now. Only events that happen after the call count.
4. **Look at the result.** It returns the first match, or says nothing matched in time. Take a `screenshot` to see the screen they're on, then investigate the match: `http_request` for a request (the `investigate-request` skill), `logs` around its time for a log line (`find-errors`).
5. If nothing matched, widen the filter (drop `status`, wait for any failed request or any error log) and try once more, or check `logs` and `http_requests` with `since: "2m"` for what did happen.

Requests to the app (screenshot, SQLite, files) need a live session with that monitor on; `session_info` lists the enabled monitors.
