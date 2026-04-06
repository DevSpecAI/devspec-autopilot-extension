# DevSpec Autopilot Extension for Gemini CLI

Automatically pick up and process [DevSpec](https://devspec.ai) action items using Gemini CLI. Run a single command and Gemini will fetch the next queued item, implement the changes, and report completion back to your team.

## Prerequisites

- [Gemini CLI](https://github.com/google-gemini/gemini-cli) installed with extensions support
- A [DevSpec](https://devspec.ai) account with at least one project
- A DevSpec MCP token with read-write scope

## Setup (< 5 minutes)

### 1. Generate an MCP Token

In the DevSpec web UI, go to **Settings > Autopilot** and generate a new MCP token. Copy the token — you'll need it in the next step.

### 2. Set Environment Variables

```bash
export DEVSPEC_MCP_TOKEN="dvs_your_token_here"
export DEVSPEC_API_URL="https://devspec.ai"
```

Add these to your shell profile (`~/.bashrc`, `~/.zshrc`, etc.) to persist across sessions.

### 3. Install the Extension

```bash
gemini extensions install https://github.com/DevSpecAI/devspec-autopilot-extension
```

### 4. Process an Action Item

```bash
gemini /autopilot:process --yolo
```

Gemini will:
1. Fetch the next queued action item from DevSpec
2. Claim it (preventing other runners from picking it up)
3. Implement the required changes
4. Commit, push, and report completion with full details
5. Send a heartbeat to the DevSpec dashboard

## Scheduling for Continuous Processing

The extension processes one item per invocation. For continuous processing, schedule it with your system's task scheduler.

### cron (Linux/macOS)

```bash
# Process one item every 30 minutes
*/30 * * * * DEVSPEC_MCP_TOKEN="dvs_..." DEVSPEC_API_URL="https://devspec.ai" gemini /autopilot:process --yolo >> /var/log/devspec-autopilot.log 2>&1
```

### launchd (macOS)

Create `~/Library/LaunchAgents/com.devspec.autopilot.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.devspec.autopilot</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/gemini</string>
        <string>/autopilot:process</string>
        <string>--yolo</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>DEVSPEC_MCP_TOKEN</key>
        <string>dvs_your_token_here</string>
        <key>DEVSPEC_API_URL</key>
        <string>https://devspec.ai</string>
    </dict>
    <key>StartInterval</key>
    <integer>1800</integer>
    <key>StandardOutPath</key>
    <string>/tmp/devspec-autopilot.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/devspec-autopilot.err</string>
</dict>
</plist>
```

Load it:
```bash
launchctl load ~/Library/LaunchAgents/com.devspec.autopilot.plist
```

### systemd (Linux)

Create `~/.config/systemd/user/devspec-autopilot.service`:

```ini
[Unit]
Description=DevSpec Autopilot Runner (Gemini CLI)

[Service]
Type=oneshot
Environment=DEVSPEC_MCP_TOKEN=dvs_your_token_here
Environment=DEVSPEC_API_URL=https://devspec.ai
ExecStart=/usr/local/bin/gemini /autopilot:process --yolo
```

Create `~/.config/systemd/user/devspec-autopilot.timer`:

```ini
[Unit]
Description=Run DevSpec Autopilot every 30 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=30min

[Install]
WantedBy=timers.target
```

Enable it:
```bash
systemctl --user enable --now devspec-autopilot.timer
```

## Troubleshooting

### "No items queued"

The DevSpec queue is empty. This is normal — the extension exits cleanly.

- Check the DevSpec web UI to verify action items exist
- Ensure items have `agent_status: queued` (items with other statuses are not picked up)
- Verify your token has scope for the correct project

### "Authentication failed" / 401 Unauthorized

Your MCP token is invalid or expired.

1. Go to DevSpec: **Settings > Autopilot**
2. Revoke the old token and generate a new one
3. Update the `DEVSPEC_MCP_TOKEN` environment variable
4. Verify: `echo $DEVSPEC_MCP_TOKEN` should show the new token

### "Item already claimed by another runner"

Another runner (Gemini CLI or Claude Code) claimed the same item simultaneously. This is a normal race condition and is safe.

- Run the command again — `get_next_work_item` will return the next available item
- No action needed — no data was corrupted

### "Error: No MCP servers found"

The extension isn't properly installed or Gemini CLI can't find the MCP configuration.

1. Verify the extension is installed: check that `gemini-extension.json` exists in the extension directory
2. Try reloading: `gemini extensions reload`
3. Reinstall: `gemini extensions install https://github.com/DevSpecAI/devspec-autopilot-extension`

### "Item processing failed"

Gemini encountered an error during implementation (test failures, merge conflicts, missing dependencies, etc.).

- The item has been marked as `failed` in DevSpec with error details
- Check the Gemini CLI output for the specific error
- Review the action item in DevSpec — the failure report includes what was attempted and where it stopped
- Retry manually or fix the underlying issue and re-queue the item

## How It Works

This extension uses [MCP (Model Context Protocol)](https://modelcontextprotocol.io) to connect Gemini CLI to DevSpec's autopilot system. The extension provides:

- **`gemini-extension.json`** — Configures the MCP connection to DevSpec using your environment variables
- **`commands/autopilot/process.toml`** — The prompt template that guides Gemini through the full autopilot workflow
- **`GEMINI.md`** — Reference documentation for all MCP tools, their parameters, and error handling patterns

When you run `/autopilot:process`, Gemini reads the prompt template and GEMINI.md for context, then executes the workflow: fetch → claim → implement → report → heartbeat.

## Runner Identity

This extension identifies itself as an **ephemeral** runner in DevSpec (as opposed to Claude Code's **persistent** runner). This means:

- Each invocation is independent — no long-running polling loop
- The DevSpec dashboard shows "Gemini CLI" with your machine hostname
- Action items show which runner completed them for audit purposes

## License

MIT — see [LICENSE](LICENSE).
