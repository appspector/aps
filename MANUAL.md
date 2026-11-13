# aps: the manual

How to use `aps`, the AppSpector command-line client, and its MCP server. Installing it and the plugin for Claude Code and Codex: see the [README](README.md).

## Quick start

```sh
brew install appspector/aps/aps     # macOS; upgrade with `brew upgrade aps`
aps login                           # email + password, stored securely
cd ~/code/my-ios-app
aps link --app "My App"             # this folder now means "My App"
aps sessions ls                     # sessions of My App
aps tail --monitor http,logs        # stream HTTP + logs from the latest live session
aps http ls --failed --since 5m     # failed requests in the last 5 minutes
aps logs --level error              # error log lines
aps screenshot -o screen.png        # screenshot from that session
```

### With an AI agent
```sh
aps login                                   # once, in a terminal
claude plugin marketplace add appspector/aps
claude plugin install appspector@appspector # Claude Code; Codex: see below
```
Then ask the agent, in your app's project folder, e.g. "the checkout fails on my phone, find out why". See [Using aps with Claude Code and Codex](#using-aps-with-claude-code-and-codex).

---

## Choosing an app and a session

Most commands work on one app and one session. You rarely need to spell them out.

### The current app
`aps` picks the app from the first of these that's set:

1. `--app <app>` on the command line
2. The `APPSPECTOR_APP` environment variable
3. A `.appspector.json` file in the current folder or any parent folder, written by `aps link`
4. The global default, set with `aps use <app>`

`<app>` can be the numeric app ID, the app name, or the bundle ID / package name. Name matching ignores case. iOS and Android versions of an app are separate AppSpector apps, so a name can match more than one; `aps` then stops and lists them with their platform and ID. Use the ID or the bundle ID to be exact.

Run `aps status` to see which app is selected and where that came from.

### The session
Commands that act on a session take an optional `[session]` argument:

| You write | Meaning |
|---|---|
| *(nothing)* or `latest` | The newest **live** session of the current app. |
| `--device <name>` | The live session from this device. Matches the custom device name or the device name, ignoring case; a partial name works. Use it when several devices are live. |
| `<session-id>` | A session of the current app, live or not. |
| `<app-id>/<session-id>` | A session of any app. This is the `REF` column in `aps sessions ls`, so it can be copied straight from there. |
| `https://app.appspector.com/<app-id>/session/<session-id>` | A session URL: copied from the dashboard (any page of the session works, e.g. one request's), or logged by your app from the SDK's `startCallback`. Contains both IDs, so it works with no current app. The older form with `/apps/` before the app ID works too. |

If a reference matches more than one session (e.g. two live devices and no `--device`), `aps` doesn't guess:
- In an interactive terminal, it shows a picker.
- Otherwise, including with `--json`, it lists the candidates on stderr and exits with code 8.

Two details:
- Several live sessions from the **same** device (e.g. just after the app restarted) count as one: the newest.
- Without a current app, `--device` searches the live sessions of all your apps.

---

## Manual

### Global options
These work with every command.

| Option | Description |
|---|---|
| `--app <app>` | Use this app for this command. See [Choosing an app and a session](#choosing-an-app-and-a-session). |
| `--json` | Machine-readable output. Lists and single objects are printed as JSON; streams (`tail`) are printed as NDJSON, one object per line. |
| `--env <prod\|staging\|dev-v1\|dev-v2>` | Which AppSpector environment to use. Default: `prod`, or the environment saved by `aps link`. See [Environments](#environments). |
| `--api-url <url>` | Override the REST API base URL, e.g. `https://app.staging.appspector.com/api`. The event bus URL is derived from it. |
| `-v`, `--verbose` | More logging on stderr: every request and response (method, URL, status, time, and the server's error message), WebSocket connections, requests to the app, and local errors. Repeat for more detail: `-vv` adds headers, bodies and protocol packets, `-vvv` timestamps. Credentials are always masked. |
| `--no-color` | Plain output, no colors. Colors also turn off automatically when stdout isn't a terminal. |
| `--no-redact` | Show secrets in the app's data instead of masking them. See [Secrets in the app's data](#secrets-in-the-apps-data). |
| `-h`, `--help` | Help for any command. |
| `-V`, `--version` | Print the version, the commit it was built from and the build date: `aps 0.1.0 (3f2c1ab 2026-10-25)`. |

Data goes to stdout; logs, progress and errors go to stderr.

#### Environments
The environment is the first of these that's set:
1. `--api-url`, then `--env`
2. the host of a dashboard URL given as the session (a URL for another environment than `--env` or `--api-url` is an error)
3. `APPSPECTOR_API_URL`
4. `env` in the nearest `.appspector.json`
5. `APPSPECTOR_ENV`
6. `default_env` in `config.toml`
7. `prod`

Tokens are stored per environment, so you can be signed in to several. An `--api-url` counts as a built-in environment only when it's that environment's `https://…/api` URL; any other URL gets its own token.

### Authentication

#### `aps login`
Sign in with your AppSpector email and password. The password is never stored: only the session AppSpector returns (its token and session cookie) is kept, in your OS keychain (macOS Keychain, Windows Credential Manager, or Linux Secret Service). If no keychain is available, it goes into `credentials.toml` in the config folder (see [Configuration](#configuration)), readable only by you. `aps login` checks the new session with the server before keeping it.

```sh
aps login                                         # prompts for email and password
aps login --email me@example.com                  # prompts for the password only
echo "$PASS" | aps login --email me@example.com --password-stdin   # for CI
```

| Option | Description |
|---|---|
| `--email <email>` | Account email. Prompted for if omitted. |
| `--password-stdin` | Read the password from stdin instead of prompting. |

> **Signed up with Google or GitHub?** Your account has no password yet. Use "Forgot password" on the AppSpector login page to set one, then run `aps login`.

#### `aps logout`
Sign out on the server and remove the stored token. A token in `APPSPECTOR_TOKEN` isn't affected.

#### `aps whoami`
Show the signed-in account: name, email and account ID. Exits with code 3 if you're not signed in.

#### Token from the environment
If `APPSPECTOR_TOKEN` is set, `aps` uses it instead of the stored token. This is useful for CI. The value of the `appspector_session` cookie from a signed-in dashboard works here: `aps` sends the value as that cookie and as the token.

### Context

#### `aps link [--app <app>]`
Link the current folder to an app by writing `.appspector.json`. Every `aps` command run in this folder, or any folder below it, then uses that app. Without `--app`, an interactive terminal shows a picker of your apps.

The file holds the app ID, name and platform, and the environment. It contains no secrets, so it can be committed and shared with your team.

```json
{ "appId": 12, "appName": "My App", "platform": "ios", "env": "prod" }
```

| Option | Description |
|---|---|
| `--path <dir>` | Write the file in this folder instead of the current one (e.g. the repository root). |

#### `aps unlink`
Remove the `.appspector.json` that applies to the current folder.

#### `aps use <app>` / `aps use --clear`
Set or clear the global default app, used when no other app is selected.

#### `aps status`
Show the current context:
- the environment and signed-in account
- the current app and where it came from (flag, environment variable, `.appspector.json` path, or global default)
- that app's live sessions

A quick way for people and agents to see what `aps` will act on. If you're not signed in, it prints what it can and exits with code 3.

### Apps

#### `aps apps ls`
List the apps you can access, including your organizations' apps.

Columns: `ID`, `NAME`, `PLATFORM`, `BUNDLE ID`, `OWNER`, `E2E` (whether end-to-end encryption is on). The current app is marked.

### Sessions

A session is one run of an app with the AppSpector SDK connected. Session states:
- `online`: live
- `offline`: the device lost its connection; its data uploads when it reconnects
- `buffered`
- `archived`: finished, read-only

#### `aps sessions ls`
List the current app's sessions, newest first, with live sessions on top.

| Option | Description |
|---|---|
| `--online` | Only live sessions. |
| `--all-apps` | Live sessions across **all** your apps, instead of the current app's. Useful when no app is selected yet. |
| `--limit <n>` | Maximum number of sessions. Default: 20. |
| `--query <text>` | Search sessions (same search as the dashboard). |

Columns: `REF` (`<app-id>/<session-id>`, usable anywhere a session is expected), `APP`, `STATE`, `DEVICE`, `OS`, `APP VERSION`, `SDK`, `STARTED`, `LAST ACTIVE`.

If no app is selected and `--all-apps` isn't given, `aps` lists live sessions across all apps and suggests `aps link`.

#### `aps sessions show [session]`
Show one session in detail: app, device, app version, SDK version, state, enabled monitors, dispatcher, and the session's dashboard URL.

### Streaming events

#### `aps tail [session]`
Print a session's events: its history first, then new events as they arrive while the session is live. For a session that isn't live, the history is printed and the command exits.

```sh
aps tail                                    # latest live session of the current app
aps tail --device pixel --monitor logs      # logs from the live Pixel session
aps tail 12/5f3c… --monitor http --json     # HTTP events as NDJSON, any app
aps tail --no-history --monitor logs        # only new log lines
```

| Option | Description |
|---|---|
| `--device <name>` | Pick the live session by device. |
| `--monitor <ids>` | Only these monitors, comma-separated: `http`, `logs`, `performance`, `sqlite`, `location`, `environment`, `screenshot`, `ios-core-data`, `user-defaults`, `notification-center`, `analytics`, `commands`, `custom-events`, `file-system`, `shared-prefs` (Android). |
| `--no-history` | Skip past events; only print new ones. |
| `--no-follow` | Print the history and exit, even for a live session. |
| `--since <duration>` | Only events newer than this, e.g. `5m` or `1h`. Event times come from the device's clock. |

In `--json` mode, each line is one event: `{"id": "http.http-response", "timestamp": 1760370000000, "payload": {…}}`. Binary fields such as HTTP bodies are printed as base64. Progress (history loaded, disconnects) goes to stderr. Secrets in the app's data are masked, in both modes (see [Secrets in the app's data](#secrets-in-the-apps-data)).

If a live session goes offline while you're following it, `tail` waits for it to come back, fills in what it missed from the history without printing anything twice, and ends when the session is archived. Press Ctrl-C to stop.

### Investigating a session

These commands answer questions about a session's events instead of streaming all of them. They work on live, offline and archived sessions, and accept `[session]` and `--device` as described above. The events come from the session's history (or archive), kept in a local cache (see [The event cache](#the-event-cache)).

Lists are newest first, `--limit` items at a time (at most 200). In a terminal, a page is printed oldest first, like a log, with the newest line last; with `--json` it's a document `{session, data, items, total, nextCursor}`. To see older items, pass the `nextCursor` back with `--cursor`. `data` says where the events came from (`history`, `archive`, or `live`) and, in `incomplete`, why they might be partial.

Times: `--since` and `--until` take a duration before now (`30s`, `5m`, `2h`, `1d`) or an ISO time (`2026-10-16T12:00:00Z`). Event times come from the device's clock.

#### `aps http ls [session]`
List the app's HTTP exchanges (a request and its response): ID, time, method, status, duration, size, URL.

```sh
aps http ls --failed --since 10m          # errors and 4xx/5xx in the last 10 minutes
aps http ls --status 5xx --url /checkout  # server errors on checkout
aps http ls --slower-than 2s              # slow requests
aps http ls --host api.example.com        # one backend only
```

| Option | Description |
|---|---|
| `--failed` | Only failures: no response (an error such as a timeout), or a status of 400 or more. |
| `--status <list>` | Only these statuses, comma-separated: `404`, `4xx`, `5xx`. |
| `--method <method>` | Only this method, e.g. `POST`. |
| `--url <text>` | Only URLs containing this text (ignoring case). |
| `--host <host>` | Only requests to this host or its subdomains: `--host example.com` matches `api.example.com`, but not `myexample.com`. |
| `--slower-than <duration>` | Only exchanges that took at least this long, e.g. `500ms`. |
| `--since`, `--until` | Time window. |
| `--limit <n>`, `--cursor <c>` | Paging. Default limit: 20. |

#### `aps http show [session] <id>`
Show one exchange in full: request and response headers and bodies, timings, TLS, and the error if it failed. `<id>` is the `ID` from `aps http ls`, or the exchange's dashboard URL (`https://app.appspector.com/<app-id>/session/<session-id>/http/<id>/…`), which names the session too. Text bodies are shown as text (JSON as sent); binary bodies are described. Bodies too large for the SDK to send inline are fetched from the app while the session is live.

| Option | Description |
|---|---|
| `--max-body <n>` | Cut bodies after this many characters. Default: 4000. `0` leaves them out. |

#### `aps http curl [session] <id>`
Print a `curl` command that reproduces one exchange's request: method, URL, headers and body. `aps` only prints the command; it never sends the request. `<id>` is the same as for `http show`.

```sh
aps http curl 7F3A…                       # an ID from `aps http ls`
aps http curl https://app.appspector.com/12/session/5f3c/http/7F3A…/info
aps http curl 7F3A… --shell powershell    # for PowerShell 7.3 or later (curl.exe)
```

- Secrets stay masked, as everywhere (see [Secrets in the app's data](#secrets-in-the-apps-data)), and a comment above the command says so. `--no-redact` gives the real values.
- Headers curl sets itself (`Host`, `Content-Length`, `Connection`, `Transfer-Encoding`) are left out, and `Accept-Encoding` becomes `--compressed`.
- Text bodies go inline (`--data-raw`). A binary body is saved unchanged to a file and read with `--data-binary @file`.
- A body too large to send inline is fetched from the app while the session is live, as in `http show`; otherwise the command says it was left out.

| Option | Description |
|---|---|
| `--shell <shell>` | Quote for `posix` (sh, bash, zsh; the default) or `powershell`. |
| `--body-file <file>` | Where to save a binary request body. Default: `aps-body-<id>.bin` in the current folder. |

#### `aps logs [session]`
List the app's log lines.

| Option | Description |
|---|---|
| `--grep <text>` | Only lines whose message or tag contains this text (ignoring case). |
| `--regex <regex>` | Only lines matching this regular expression ([Rust syntax](https://docs.rs/regex/latest/regex/#syntax); `(?i)` ignores case). |
| `--level <level>` | Only this level or more severe: `verbose`, `debug`, `info`, `warn` (or `warning`), `error`, `assert`. |
| `--tag <text>` | Only lines whose tag contains this text. |
| `--source <source>` | Only `user` lines (the app's logger, CocoaLumberjack) or `system` lines (stdout, `print`, `NSLog`). |
| `--since`, `--until` | Time window. |
| `--limit <n>`, `--cursor <c>` | Paging. Default limit: 50. |

#### `aps perf [session]`
Summarize the app's performance: FPS, CPU %, memory, network speed (minimum, average, 95th percentile, maximum, latest), battery, and the moments of highest CPU and memory and lowest FPS. Takes `--since` and `--until`.

#### `aps events [session]`
List events of any monitor, with their payloads as JSON: custom events, analytics, notifications, location, the SQLite query log, Core Data, UserDefaults (and HTTP and logs too). Performance samples are left out unless asked for with `--monitor performance`.

| Option | Description |
|---|---|
| `--monitor <id>` | Only this monitor's events, e.g. `custom-events`. |
| `--kind <id>` | Only this event ID, e.g. `custom-events.user-event`. |
| `--grep <text>` | Only events whose payload contains this text (ignoring case). |
| `--summary` | Count the events of each kind instead of listing them. |
| `--since`, `--until`, `--limit`, `--cursor` | As above. Default limit: 20. |

#### `aps wait http|log|event [session]`
Wait for the next **new** HTTP response, log line or event that matches, in a live session, and print it. Events that happened before the command started don't count. Useful in scripts and for agents: start it, then reproduce the problem.

```sh
aps wait http --status 5xx                 # the next server error
aps wait log --level error --grep payment  # the next error log mentioning payment
aps wait event --kind custom-events.user-event --grep checkout_done
```

`wait http` takes the filters of `http ls` (`--failed`, `--status`, `--method`, `--url`, `--host`, `--slower-than`); `wait log` those of `logs` (`--grep`, `--regex`, `--level`, `--tag`); `wait event` those of `events` (`--monitor`, `--kind`, `--grep`). All take `--timeout <duration>` (default `60s`). If nothing matches in time, `aps` exits with code 5.

#### `aps sessions wait`
Wait until a live session of the current app appears, e.g. after launching the app, and print it.

| Option | Description |
|---|---|
| `--new` | Only a session that starts after the command does; sessions already live don't count. |
| `--device <name>` | Only a session from this device. |
| `--all-apps` | A live session of any of your apps. |
| `--timeout <duration>` | How long to wait. Default: `2m`. Exit code 5 if nothing appears. |

### Querying a live session

These commands send a request to the app on the device, so they only work while the session is `online`. Only **read-only** requests are allowed in this version. Every command here accepts `[session]` and `--device` as described above.

If the app doesn't answer in time (exit code 5), the monitor may be disabled in the app: the SDK doesn't answer requests for monitors it isn't running.

#### `aps screenshot [session]`
Take a screenshot of the app's current screen.

| Option | Description |
|---|---|
| `-o`, `--output <file>` | Where to save the image. Default: `screenshot-<session>-<time>.<ext>` in the current directory, where `<ext>` matches the image the app sent (iOS sends JPEG, so `.jpg`). |

#### `aps sql dbs [session]`
List the app's SQLite databases: ID, name and path.

#### `aps sql query [session] --db <database> "<sql>"`
Run a query against one of the app's SQLite databases and print the result as a table, or as JSON with `--json`. Only `SELECT`, `PRAGMA`, `EXPLAIN` and `WITH … SELECT` statements are accepted; anything that could change data is rejected (exit code 7), and so is anything `aps` can't be sure about, such as several statements or query parameters (`$x`, `:x`, `@x`, `?`). Values in columns whose names look like credentials (`password`, `auth_token`, …) are masked.

| Option | Description |
|---|---|
| `--db <database>` | Database ID or name, from `aps sql dbs`. |
| `--limit <n>` | Maximum number of rows. Default: 100. Use `0` for no limit. `SELECT` and `WITH` queries are limited on the device; `PRAGMA` and `EXPLAIN` results are cut afterwards. |

#### `aps environment [session]`
Print the device and app info (the app's `Info.plist`), its environment variables and its launch arguments, as JSON. Variables whose names look like credentials (`API_TOKEN`, `AWS_SECRET_ACCESS_KEY`, …) are masked.

#### `aps files ls [session] [path]`
List a folder in the app's sandbox: name, size, modification time and path, folders first. Paths start with `/`; without one, `aps` lists the roots (the app's home folder and its bundle). In Git Bash on Windows, which rewrites arguments that start with `/`, write `//Documents` or set `MSYS_NO_PATHCONV=1`.

#### `aps files cat [session] <path>`
Print a text file, with credentials masked. Binary files aren't printed; save them with `-o <file>`, which writes the file unchanged. Files over 5 MB can't be read through a session (the SDK drops larger messages).

#### `aps commands ls [session]`
List the custom commands the app registered with the SDK, with their arguments. Running them is planned for a later milestone.

#### `aps request [session] <message-id> [--data <json>]`
Send a raw monitor request and print the response as JSON. This is a low-level tool for debugging and scripting. Only read-only message IDs are allowed, e.g. `environment.get-env-info`, `file-system.list-items`, `user-defaults.get-contents` or `ios-core-data.get-models`; `sqlite.exec-query` is allowed with a read-only query. Anything else exits with code 7. Secrets in the answer are masked.

```sh
aps request environment.get-env-info --json
aps request 12/5f3c… file-system.list-items --data '{"path": "/"}'
```

A message ID always contains a dot, or is exactly `screenshot`, so `aps` can tell it apart from a session reference.

| Option | Description |
|---|---|
| `--data <json>` | Request payload. Default: `{}`. |
| `--timeout <duration>` | How long to wait for the app to answer. Default: `30s`. |

### The event cache
`aps` keeps the events it reads in a local SQLite file, so questions about a session don't download it again and again: an archived session is downloaded once, and while `aps mcp` is asked about a live session, it follows it and keeps the cache current. CLI commands download a live session's history on every run (only what's new is stored).

The cache holds the app's data as received (HTTP bodies and logs can contain user data), so it's kept private:
- In the OS cache folder: `~/Library/Caches/com.appspector.aps/` (macOS), `~/.cache/aps/` (Linux), `%LOCALAPPDATA%\appspector\aps\cache\` (Windows). On macOS and Linux only you can read it.
- Sessions unused for 7 days are deleted, and the cache is kept under 1 GB by deleting the least recently used sessions.
- `APS_NO_CACHE=1` keeps everything in memory instead; `APS_CACHE_DIR` moves it.

#### `aps cache ls`
Show where the cache is, its size, and the sessions it holds.

#### `aps cache clear`
Delete everything in the cache.

### Secrets in the app's data
What `aps` shows from your app (HTTP traffic, logs, environment, files, query results) often contains secrets: your users' tokens and cookies, API keys, passwords in request bodies. Because this output is often read by an AI agent, `aps` masks them as `<redacted>` by default, in every command (including `tail`) and in the MCP server:
- headers: `Authorization` (the scheme stays: `Bearer <redacted>`), `Cookie` and `Set-Cookie` (cookie names stay), and any header whose name contains `token`, `secret`, `api-key`, `auth`, `session`, `password` or `signature`
- URLs and form bodies: credential parameters (`access_token`, `api_key`, `key`, `code`, `sig`, …) and passwords in `user:password@`
- JSON (bodies, event payloads, answers): fields whose names look like credentials (`token`, `password`, `secret`, `apiKey`, `authorization`, `sessionId`, `privateKey`, …)
- text (log lines, text bodies): JWTs, `Bearer …` values and `password=…` / `"token": "…"` pairs
- environment variables and SQL columns whose names look like credentials

Masking is best-effort and errs on the side of masking. `--no-redact` shows everything as received; files saved with `-o` are never changed.

### Using aps with Claude Code and Codex
The **AppSpector plugin** gives Claude Code and Codex the `aps mcp` tools and skills for debugging with them: setting up, investigating a request, finding errors in the logs, checking performance, and reproducing a bug live. It's published with `aps` in [`appspector/aps`](https://github.com/appspector/aps), which is a plugin marketplace for both agents:

```sh
brew install appspector/aps/aps
aps login                                     # in a terminal
cd ~/code/my-app && aps link                  # the app this project is

# Claude Code (terminal; the desktop app's Code tab shares the plugins)
claude plugin marketplace add appspector/aps
claude plugin install appspector@appspector

# Codex (CLI; the ChatGPT desktop app's Codex shares the configuration)
codex plugin marketplace add appspector/aps
codex plugin add appspector@appspector
```

Start a new agent session in your app's project folder. `/mcp` in Claude Code should show the `appspector` server connected, and `/appspector:setup` checks everything. The plugin finds `aps` even when an agent app doesn't have your shell's `PATH` (it also looks in `/opt/homebrew/bin`, `/usr/local/bin`, `~/.local/bin`, `~/.cargo/bin`, and where `aps` last ran from).

**Updates:** `brew upgrade aps` updates `aps`. For the plugin:
- Claude Code: `claude plugin marketplace update appspector`, then `claude plugin update appspector@appspector`, and start a new session. Claude Code doesn't update third-party plugins by itself unless you turn it on: `/plugin` → Marketplaces → appspector → Enable auto-update.
- Codex updates it when it starts (or run `codex plugin marketplace upgrade`); start a new session.

The plugin expects `aps` of its own version or newer, so upgrade `aps` too when the plugin updates; `/appspector:setup` checks it.

On macOS, a newly installed or upgraded `aps` asks once, in a Keychain dialog, whether it may use its stored sign-in: click **Always Allow**. If nobody answers, `aps` stops waiting after 30 seconds and says so.

### Using aps with other AI agents

#### `aps mcp`
Run the MCP server on stdin and stdout. The plugin above starts it for Claude Code and Codex; other agents start it themselves once it's in their configuration:

```sh
claude mcp add appspector -- aps mcp            # Claude Code without the plugin (don't use both)
```
Codex, in `~/.codex/config.toml`:
```toml
[mcp_servers.appspector]
command = "aps"
args = ["mcp"]
```
Other clients (Cursor, VS Code, Claude Desktop, …): add a stdio server whose command is `aps` with the argument `mcp`, e.g.:
```json
{ "mcpServers": { "appspector": { "command": "aps", "args": ["mcp"] } } }
```

The server uses what `aps login` stored, and picks the app the way every command does: from `.appspector.json` in the folder the agent runs in (`aps link`), `APPSPECTOR_APP`, or `aps use`. Global options work too, e.g. `aps mcp --env staging`. A later `aps login`, `aps link` or `aps use` takes effect without restarting the agent. If you're not signed in, every tool tells the agent to ask you to run `aps login`; the server never asks for or handles your password.

Tools, all read-only except `link_app`:

| Tool | Does what | Like |
|---|---|---|
| `status` | Account, current app, its live sessions. Agents start here. | `aps status` |
| `list_apps`, `link_app` | List apps; link the project folder to one. | `aps apps ls`, `aps link` |
| `list_sessions`, `session_info`, `wait_for_session` | Find sessions; wait for a new live one. | `aps sessions ls/show/wait` |
| `http_requests`, `http_request`, `http_curl` | Filtered HTTP exchanges; one in full; its request as a curl command. | `aps http ls/show/curl` |
| `logs` | Filtered log lines. | `aps logs` |
| `performance` | Performance summary. | `aps perf` |
| `events` | Any monitor's events, or counts per kind. | `aps events` |
| `wait_for_event` | The next new matching HTTP response, log line or event. | `aps wait` |
| `screenshot` | The app's screen, as an image. | `aps screenshot` |
| `environment` | Device and app info, environment variables. | `aps environment` |
| `sql_databases`, `sql_query` | Read-only SQLite. | `aps sql` |
| `list_files`, `read_file` | The app's sandbox; images are shown. | `aps files` |
| `list_commands` | The app's custom commands. | `aps commands ls` |
| `request` | Any read-only monitor request. | `aps request` |

Each tool's description tells the agent when to use it. A dashboard link the user pastes (`https://app.appspector.com/…/session/…`) goes to the tools as the session, or as the exchange for `http_request` and `http_curl`; the server tells agents never to fetch such links themselves, since they need the user's login.

Results are compact JSON, paged and cut to a size an agent can read; secrets are masked (start the server with `aps mcp --no-redact` to turn that off). While a live session is being asked about, the server follows it in the background, so answers include events up to the last moment; it lets go after 15 minutes without questions.

Agents without MCP can use the same commands with `--json`.

### Environment variables

| Variable | Description |
|---|---|
| `APPSPECTOR_TOKEN` | Use this token (e.g. the dashboard's `appspector_session` cookie value) instead of the stored one. |
| `APPSPECTOR_APP` | The current app (overrides `.appspector.json` and the global default). |
| `APPSPECTOR_ENV` | Default environment (`prod`, `staging`, `dev-v1`, `dev-v2`). |
| `APPSPECTOR_API_URL` | Default REST API base URL (same as `--api-url`). |
| `NO_COLOR` | Disable colors. |
| `APS_NO_KEYRING` | Set to keep the token in `credentials.toml` instead of the OS keychain (e.g. on headless machines). |
| `APS_CONFIG_DIR` | Use this folder for `config.toml` and `credentials.toml` instead of the default. |
| `APS_CACHE_DIR` | Use this folder for the event cache instead of the OS cache folder. |
| `APS_NO_CACHE` | Set to keep the event cache in memory only, so nothing is written to disk. |

### Configuration
Global settings (default environment, default app per environment, last-used email) are kept in a config file:
- macOS: `~/Library/Application Support/com.appspector.aps/config.toml`
- Linux: `~/.config/aps/config.toml`
- Windows: `%APPDATA%\appspector\aps\config\config.toml`

When no keychain is available, the token is kept next to it in `credentials.toml`. The event cache lives in the OS cache folder (see [The event cache](#the-event-cache)).

Per-folder settings live in `.appspector.json` (see `aps link`).

### Exit codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | General error, including an error returned by the app, and apps with end-to-end encryption |
| 2 | Invalid usage (bad arguments, or no app selected where one is needed) |
| 3 | Not signed in, or the token has expired: run `aps login` |
| 4 | Not found (app, session or database), including "no live session", something you have no access to, and `aps unlink` with no `.appspector.json` |
| 5 | Connection problem or timeout, including the app not answering a request in time, and `aps wait` or `aps sessions wait` ending without a match |
| 6 | The session isn't live, so it can't answer requests (also when the dispatcher no longer has it) |
| 7 | Refused: the request would change app data, which isn't allowed in this version |
| 8 | Ambiguous: more than one app or session matches; the candidates are listed on stderr |

---
