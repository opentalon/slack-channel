# OpenTalon Slack Channel

YAML-driven Slack channel for [OpenTalon](https://github.com/opentalon/opentalon). No compiled binary — runs in-process using the core's generic YAML channel runtime.

## Prerequisites

1. A Slack app with **Socket Mode** enabled
2. An **App-Level Token** (`xapp-...`) with `connections:write` scope
3. A **Bot Token** (`xoxb-...`) with scopes: `chat:write`, `reactions:read`, `reactions:write`, `app_mentions:read`, `channels:history`, `im:history`, `users:read`

### Creating the Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and create a new app
2. Under **Socket Mode**, enable it and generate an App-Level Token with `connections:write`
3. Under **OAuth & Permissions**, add the bot scopes listed above
4. Under **Event Subscriptions**, subscribe to bot events: `app_mention`, `message.im`
5. Install the app to your workspace

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

The channel provides 5 tools that the LLM can call:

| Tool | Description |
|------|-------------|
| `slack.post_message` | Post a message to a channel or thread |
| `slack.add_reaction` | Add an emoji reaction to a message |
| `slack.read_thread` | Read all replies in a thread |
| `slack.update_message` | Edit a previously sent message |
| `slack.get_user_info` | Get a user's profile |

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
| `LICENSE` | MIT License |
