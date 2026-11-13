---
name: investigate-request
description: Investigate a failing, slow or wrong HTTP request in the app with AppSpector — find it, read headers and bodies, explain the cause, export it as curl. Use for network and API bugs.
---

# Investigate a request

If the AppSpector tools (`status`, `http_requests`, …) aren't available, or the `appspector` MCP server failed to start, follow the `setup` skill first.

1. **Find it.** Call `http_requests` with filters, not the whole list:
   - `failed: true` for errors and 4xx/5xx; `status: "5xx"` or `401`; `host` for one backend; `url` for a path; `slower_than_ms` for slow ones; `since: "10m"` to stay near the problem.
   - If the user pasted an AppSpector link (`app.appspector.com/…/session/…`), pass it as `session`; a link to one request works as `id`. Never fetch these links over HTTP.
   - The list is newest first and paged (`nextCursor`). `data.incomplete` means the session's history couldn't be loaded completely, so say that an answer may be partial.
2. **Read it.** Call `http_request` with the `id`: request and response headers and bodies, timings, TLS, and the error for requests that got no response. Large bodies are fetched from the app while the session is live.
3. **Explain it.** Compare with a request to the same endpoint that worked (same `url` filter, without `failed`). Look for what changed: auth headers (`401` after a token expired), parameters, body fields, the host, timing. Check the logs around its time (`logs` with `since`/`until` a few seconds either side) for what the app did before and after; see the `find-errors` skill.
4. **Reproduce it outside the app** if the user wants: `http_curl` returns the request as a curl command. Secrets stay masked as `<redacted>`; say so, rather than guessing values.

Secrets in the app's data are always masked. Everything here is read-only: nothing changes the app or replays the request.
