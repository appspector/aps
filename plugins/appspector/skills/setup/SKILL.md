---
name: setup
description: Set up AppSpector for this project — check that the aps CLI is installed and current, signed in and linked to an app. Use when AppSpector tools are missing or failing, or before first use.
---

# Set up AppSpector

The AppSpector tools come from `aps`, a command-line program the user installs; the plugin starts its MCP server (`aps mcp`). Go through these steps in order and stop at the first one that needs the user.

<!-- The two lines below are written by scripts/plugins/render.sh; edit them there. -->
- **Plugin version:** 0.1.0 (`aps` must be this version or newer)
- **Install aps:** install it with Homebrew: brew install appspector/aps/aps (to upgrade: brew upgrade aps), then restart the agent

## 1. Is aps installed, and new enough?
- **No AppSpector tools at all, or the `appspector` MCP server failed to start:** almost always `aps` isn't installed, or isn't where the plugin looks. Tell the user how to install it (the **Install aps** line above). If you can run shell commands, confirm first with the checks below.
- If the AppSpector tools (`status`, `http_requests`, …) are available, call `status`: its `aps.version` is the installed version.
- Otherwise, if you can run shell commands, run `aps --version`. It prints `aps <version> (<commit> <date>)`. If `aps` isn't found, also try `/opt/homebrew/bin/aps`, `/usr/local/bin/aps`, `~/.local/bin/aps` and `~/.cargo/bin/aps`: agent apps opened from the Dock often have a shorter `PATH` than the user's terminal.
- **Missing, or older than the plugin version above:** tell the user how to install it (the **Install aps** line above), and that after installing they need to restart the agent (in Claude Code, `/reload-plugins` is enough) so the tools appear. Don't try to install it yourself.
- **Installed, but the tools still don't appear:** the agent may still remember the failed start. Ask the user to reconnect the `appspector` server (`/mcp` in Claude Code, or restart the agent) and check its error there. "aps isn't installed" means the plugin's launcher couldn't find `aps` in any of the places above.

## 2. Is the user signed in?
- `status` returns `signedIn`. If it's `false`, or a tool says "Not signed in or token expired", ask the user to run this in their own terminal:

  ```sh
  aps login
  ```

  It asks for their AppSpector email and password. **Never** ask for the password yourself or run `aps login` for them. Accounts created with Google or GitHub need a password first ("Forgot password" on the AppSpector login page).

## 3. Which app?
- `status` returns `app` and where it was chosen from (`appSource`). If there's no app, call `list_apps` and ask the user which app it is (iOS and Android versions are separate apps). Then save it with `link_app`:
  - **in a project folder:** `link_app` with the app, which writes `.appspector.json` there (no secrets; it can be committed). If `status` says `projectFolder` is unknown (Codex doesn't tell aps the chat's folder), pass `path`: the project's folder. If the folder already has `.appspector.json`, pass its `appId` as `app` to the tools instead.
  - **in a chat without a project:** `link_app` with `default: true`, which makes it the default app everywhere (like `aps use` in a terminal).
- Or pass `app` to each tool for a one-off question about another app.

## 4. Check
Call `status` again: it should show the account, the app and, if the app is running with the AppSpector SDK, its live sessions. If there are none, the user starts the app; `wait_for_session` waits for it.
