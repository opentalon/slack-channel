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
| `users:read.email` | Read user email addresses — required for inbound enrichment (forwards `X-User-Email` to WhoAmI) |

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

Credentials are passed via the per-instance `config:` block so a single
opentalon process can run multiple Slack bots side-by-side. The values
support `${ENV_VAR}` expansion, so secrets stay in env vars.

```yaml
channels:
  slack:
    enabled: true
    github: "opentalon/slack-channel"
    ref: "master"
    config:
      app_token: ${SLACK_APP_TOKEN}      # connection token (xapp-…)
      bot_token: ${SLACK_BOT_TOKEN}      # bot user token (xoxb-…)
      ack_reaction: eyes                  # optional: react when received
      done_reaction: white_check_mark     # optional: react when answered
```

### Multiple Slack bots in one OpenTalon process

Each entry under `channels:` is a distinct bot. Give each a unique key
(used for session/dedup/actor scoping) and its own credentials:

```yaml
channels:
  slack-admin:
    enabled: true
    github: "opentalon/slack-channel"
    ref: "master"
    config:
      app_token: ${SLACK_APP_TOKEN_ADMIN}
      bot_token: ${SLACK_BOT_TOKEN_ADMIN}
  slack-customer:
    enabled: true
    github: "opentalon/slack-channel"
    ref: "master"
    config:
      app_token: ${SLACK_APP_TOKEN_CUSTOMER}
      bot_token: ${SLACK_BOT_TOKEN_CUSTOMER}
```

Pair with opentalon's WhoAmI `metadata_headers` to give each bot its own
permissions — the channel writes `msg.Metadata["channel_id"]` from its bot
user ID, which opentalon forwards as an HTTP header so your WhoAmI server
can branch on it.

### Inbound enrichment: sender email → WhoAmI

Every inbound message triggers a `users.info` lookup to fetch the sender's
profile email and display name. The results land in `msg.Metadata` as
`user_email` and `user_name`, which opentalon's WhoAmI `metadata_headers`
config forwards as HTTP headers (e.g. `X-User-Email`). This lets a WhoAmI
server identify users by their corporate email regardless of which Slack
workspace or bot installation the message came through.

```yaml
# opentalon config.yaml
profiles:
  who_am_i:
    metadata_headers:
      channel_id: X-Channel-Id     # which bot
      user_email: X-User-Email     # which user (by corp email)
      user_name:  X-User-Name      # for logs/audit
```

Behaviour:
- **Caching**: lookups are cached for 1h per sender (default; tunable via
  `inbound.enrich.user.cache.ttl` in `channel.yaml`). With Redis configured
  in opentalon, the cache is shared across pods and survives restarts;
  without Redis, an in-memory cache is used instead.
- **Fail-closed**: if `users.info` errors out or the response is missing
  the email (e.g. the bot lacks `users:read.email`), the message is
  rejected with a user-visible error `"We couldn't verify your account
  info right now. Please try again in a moment."` This prevents WhoAmI
  from seeing half-known identities. Make sure the scope is granted before
  rolling this out to production.
- **Performance**: cold lookup adds ~100-300ms latency; cache hits are
  sub-millisecond. Slack's `users.info` is tier-2 rate-limited at ~20
  req/min/workspace, which is fine with caching enabled.

### Migrating from a single-bot setup

Pre-multi-instance configs read credentials from `{{env.SLACK_*_TOKEN}}`
directly inside `channel.yaml`. Move them into the `config:` block as
shown above; the channel.yaml in this repo now reads `{{config.app_token}}`
and `{{config.bot_token}}`. The env var names themselves stay the same
because `${ENV_VAR}` expansion runs against the host's environment.

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
