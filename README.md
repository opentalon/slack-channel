# OpenTalon Slack Channel

YAML-driven Slack channel for [OpenTalon](https://github.com/opentalon/opentalon). No compiled binary — runs in-process using the core's generic YAML channel runtime.

## Prerequisites

1. A Slack app with **Socket Mode** enabled
2. An **App-Level Token** (`xapp-...`) with `connections:write` scope
3. A **Bot Token** (`xoxb-...`) with the scopes listed below

### Creating the Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From scratch**
2. Give it a name (e.g. `opentalon_bot`) and select your workspace

### Socket Mode

3. Go to **Socket Mode** (left sidebar) → toggle **ON**
4. Create an App-Level Token with the `connections:write` scope — save the `xapp-...` token

### OAuth & Permissions

5. Go to **OAuth & Permissions** → under **Bot Token Scopes**, add:

| Scope | Why |
|-------|-----|
| `app_mentions:read` | Receive @mention events in channels |
| `chat:write` | Send messages |
| `channels:history` | Read messages in public channels the bot is in |
| `channels:join` | Join public channels |
| `channels:read` | View basic channel info |
| `groups:history` | Read messages in private channels the bot is in |
| `groups:read` | View basic private channel info |
| `im:history` | Read direct messages |
| `im:read` | View basic DM info |
| `im:write` | Open DM conversations |
| `reactions:read` | Read emoji reactions |
| `reactions:write` | Add/remove emoji reactions |
| `users:read` | Look up users by name/ID |

### Event Subscriptions

6. Go to **Event Subscriptions** → toggle **ON**
7. Under **Subscribe to bot events**, add:
   - `app_mention` — triggers when someone @mentions the bot in a channel
   - `message.im` — triggers when someone sends a direct message to the bot

### App Home (required for DMs)

8. Go to **App Home** (left sidebar)
9. Scroll to **Show Tabs** and enable:
   - **Messages Tab** — check this to allow DMs with the bot
   - **"Allow users to send Slash commands and messages from the messages tab"** — check this too

> Without the Messages Tab enabled, users will see "Sending messages to this app has been turned off" and DMs won't work.

### Install

10. Go to **OAuth & Permissions** → **Install to Workspace** (or **Reinstall** if updating scopes)
11. Save the **Bot User OAuth Token** (`xoxb-...`)

> After changing scopes or event subscriptions, you must reinstall the app for changes to take effect.

## Setup

### 1. Clone this repo into your OpenTalon channels directory

```bash
cd your-opentalon-project
git clone https://github.com/opentalon/slack-channel channels/slack
```

Or let OpenTalon fetch it automatically via `github`/`ref` in config (see below).

### 2. Set environment variables

```bash
export SLACK_APP_TOKEN="xapp-1-..."
export SLACK_BOT_TOKEN="xoxb-..."
```

Or add them to your `.env` file:

```
SLACK_APP_TOKEN="xapp-1-..."
SLACK_BOT_TOKEN="xoxb-..."
```

### 3. Add to your OpenTalon config.yaml

```yaml
channels:
  slack:
    enabled: true
    plugin: "./channels/slack/channel.yaml"
    config:
      ack_reaction: eyes              # react with this when message received
      done_reaction: white_check_mark # react with this when response sent
```

Or use auto-fetch from GitHub:

```yaml
channels:
  slack:
    enabled: true
    plugin: "./channels/slack/channel.yaml"
    github: "opentalon/slack-channel"
    ref: "master"
    config:
      ack_reaction: eyes
      done_reaction: white_check_mark
```

### 4. Run OpenTalon

```bash
source .env && go run ./cmd/opentalon -config config.yaml
```

You should see:

```
yaml-channel: init auth_test done
yaml-channel: init connect done
yaml-channel: slack started
channel-manager: loaded slack via yaml
yaml-channel: slack connected to WebSocket
```

## Usage

- **DM the bot** — responds directly
- **@mention the bot** in a channel — responds in a thread
- **Reactions** — eyes on receive, checkmark on response (configurable via `config`)

## Config Options

| Key | Description | Default |
|-----|-------------|---------|
| `ack_reaction` | Emoji reaction added when a message is received | _(none)_ |
| `done_reaction` | Emoji reaction added when the response is sent | _(none)_ |

Set either to empty or omit to disable that reaction.

## Tools

The channel provides 7 tools that the LLM can call:

| Tool | Description |
|------|-------------|
| `slack.post_message` | Post a message to a channel or thread |
| `slack.add_reaction` | Add an emoji reaction to a message |
| `slack.read_thread` | Read all replies in a thread |
| `slack.update_message` | Edit a previously sent message |
| `slack.get_user_info` | Get a user's profile by ID |
| `slack.list_users` | List workspace users (find users by name) |
| `slack.open_dm` | Open a DM channel with a user (returns channel ID for `post_message`) |

## How It Works

No Go code, no Slack library. The `channel.yaml` spec defines everything as templated HTTP calls and WebSocket handling:

- **Init**: calls `auth.test` (get bot user ID) and `apps.connections.open` (get WebSocket URL)
- **Connection**: dials the WebSocket URL, auto-reconnects with exponential backoff
- **Inbound**: acks frames, extracts events, filters by type, skips bot messages, deduplicates
- **Outbound**: chunks long messages, calls `chat.postMessage`
- **Hooks**: adds/removes reactions on receive/response

The OpenTalon core's YAML channel runtime executes all of this in-process.

## Files

| File | Purpose |
|------|---------|
| `channel.yaml` | Channel spec — connection, events, message handling |
| `tools.yaml` | Tool definitions for LLM function calling |
| `LICENSE` | Apache 2.0 License |
