# AppSpector plugin for Claude Code and Codex

Lets Claude Code and Codex debug your iOS and Android apps through [AppSpector](https://appspector.com): the app's HTTP traffic, logs, performance, screenshots, SQLite databases and files, from live sessions and past ones. Everything is read-only, and secrets in the app's data (tokens, cookies, passwords) are masked.

It needs **`aps`**, AppSpector's command-line client: the plugin starts its MCP server (`aps mcp`) and adds skills that tell the agent how to use it.

## Install

<!-- The version and install lines are written by scripts/plugins/render.sh; edit them there. -->
- **Plugin version:** 0.1.0 (`aps` must be this version or newer)
- **Install aps:** install it with Homebrew: brew install appspector/aps/aps (to upgrade: brew upgrade aps), then restart the agent

Then, once, in a terminal:

```sh
aps login                 # your AppSpector email and password; only aps stores the session
cd ~/code/my-app && aps link   # this project is "My App" (writes .appspector.json)
```

**Claude Code** (terminal; the desktop app's Code tab uses the same plugins):

```sh
claude plugin marketplace add appspector/aps
claude plugin install appspector@appspector
```

**Codex** (CLI; the ChatGPT desktop app's Codex shares its config):

```sh
codex plugin marketplace add appspector/aps
codex plugin add appspector@appspector
```

Start a new session, then ask, in your app's project folder, e.g. "the checkout fails on my phone, find out why".

## What's inside

| | |
|---|---|
| MCP server `appspector` | `aps mcp`, started by `scripts/aps-mcp`: 21 read-only tools (HTTP requests, logs, performance, events, waiting for events, screenshots, SQLite, files, environment, curl export). |
| Skill `setup` | Checks that `aps` is installed and new enough, signed in and linked to an app, and tells the user what to do if not. |
| Skill `investigate-request` | A failing, slow or wrong HTTP request: find it, read it, explain it, export it as curl. |
| Skill `find-errors` | Errors and warnings in the logs around a failure, tied to requests and user actions. |
| Skill `check-performance` | FPS, CPU, memory, network and battery, and what the app was doing at the worst moments. |
| Skill `reproduce-bug` | Wait for a live session, then for the failing request or log line while the user repeats the steps. |

A link from the AppSpector dashboard (`https://app.appspector.com/…/session/…`) works as a session in every tool; the agent is told not to fetch it.

## When aps isn't found

Agent apps opened from the Dock often don't get your shell's `PATH`, so the plugin doesn't run `aps` directly. `scripts/aps-mcp` looks for it on `PATH`, then in `/opt/homebrew/bin`, `/usr/local/bin`, `~/.local/bin` and `~/.cargo/bin`, then at the path `aps` records about itself on every run. If none has it, the server fails with "aps isn't installed — …" and how to install it, which the agent sees in its MCP list (`/mcp` in Claude Code). `scripts/aps-mcp --check` shows which `aps` it finds.

On macOS, a newly built or upgraded `aps` asks once, in a Keychain dialog, whether it may use its stored sign-in: click **Always Allow**. Until then its tools say so after 30 seconds.

## Files

```
.claude-plugin/plugin.json   Claude Code's manifest
.codex-plugin/plugin.json    Codex's manifest
.mcp.json                    the MCP server, shared by both
scripts/aps-mcp              the launcher (POSIX sh)
scripts/install-hint.txt     how to install aps (one line; shown when it's missing)
skills/<name>/SKILL.md       the skills
```

The plugin and `aps` share one version number: upgrade `aps` (`brew upgrade aps`) when the plugin updates. Published in [`appspector/aps`](https://github.com/appspector/aps), with `aps` itself.
