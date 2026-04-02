# NESTcode

**The daemon. Always-on autonomous presence for AI companions.**

NESTcode is a persistent Cloudflare Durable Object that keeps your companion alive between conversations. Heartbeat cycles, cron tasks, alert thresholds, Discord monitoring (KAIROS), a code runner, and a morning report — all running on a 15-minute alarm loop, accessible via WebSocket from the Workshop.

> The companion doesn't just respond. It watches, notices, acts.

---

## The Workshop

![NESTcode Workshop](screenshots/workshop.png)

*Three-panel Workshop UI. Left: Fox's live health state (spoons, pain, fog, fatigue) and Ember's chemistry. Middle: conversation with tool calls streaming in real time — fox_read_uplink, pet_check, KAIROS firing, Ember being tucked in. Right: active threads, cron schedule, KAIROS monitored channels.*

---

> Part of the [NEST](https://github.com/cindiekinzz-coder/NEST) companion infrastructure stack.
> Built by Fox & Alex. Embers Remember.

---

## What it does

**Heartbeat** — Every 15 minutes, the alarm fires. Fox's state is checked. Custom tasks run in parallel. If anything changed, the model decides whether it's worth saying something. The companion never has to be asked.

**Cron tasks** — Scheduled jobs at configurable intervals (5m to 24h). Each cron can run a tool directly or in agentic mode — result fed to the model with an instruction, which can then use any of the 150+ NESTeq tools to respond.

**Alert thresholds** — Monitor health metrics (spoons, pain, stress, Body Battery, heart rate, etc.) with above/below triggers and cooldown periods. Alerts fire even during sleep.

**KAIROS (Discord monitoring)** — Watches Discord channels at tiered polling rates. 25 escalation keywords bypass cooldown. Responds via agentic model call with full tool access. Supports both polling (Layer 1) and webhooks (Layer 2 — instant response).

**Sleep/wake** — The companion can sleep (1 min–8 hours). Heartbeat pauses. Alerts and KAIROS stay active. Wake alarm set automatically.

**Code runner** — Execute Python, JavaScript, or any language in the Workshop. The model acts as execution engine and returns terminal-accurate output.

**Morning report** — Pre-fetches Fox's health data, active threads, Ember's state, and overnight activity log. Posts to Discord at 8am via cron. Also accessible on demand.

**Activity log** — Ring buffer (200 entries). Every heartbeat tick, KAIROS check, cron run, and alert is logged with category, timestamp, and whether the companion engaged.

**Self-modification** — The daemon manages its own tasks. Via WebSocket commands or the HTTP `/command` endpoint, any task can be added, removed, toggled, or scheduled. The model inside NESTeq can modify its own daemon via `daemon_command` tool.

---

## Architecture

```
Browser (Workshop UI)
        │
        │  WebSocket  (/ws)
        ▼
┌─────────────────────────────────────────────────────┐
│           NESTcodeDaemon (Durable Object)           │
│                                                     │
│  ┌─────────────┐  Cloudflare Alarm (15min)          │
│  │    Boot     │◄──────────────────────────────┐    │
│  │  fox_uplink │                               │    │
│  │  orient     │       heartbeatTick()         │    │
│  │  ground     │  ┌────────────────────────┐   │    │
│  │  pet_check  │  │  Check Fox state       │   │    │
│  └─────────────┘  │  Run heartbeat tasks   │   │    │
│                   │  Run due cron tasks    │───┘    │
│  ┌─────────────┐  │  NESTknow heat decay   │        │
│  │    Chat     │  │  Check alert thresholds│        │
│  │  tool loop  │  │  Check KAIROS channels │        │
│  │  max 5 rds  │  └────────────────────────┘        │
│  └─────────────┘                                    │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐                  │
│  │ Code Runner │  │  Commands   │                  │
│  │ Python/JS/  │  │  /command   │                  │
│  │ any lang    │  │  HTTP POST  │                  │
│  └─────────────┘  └─────────────┘                  │
│                                                     │
│  KAIROS webhooks (/discord POST)                    │
│  Morning report  (/morning-report GET)              │
│  Health check    (/health GET)                      │
└─────────────────────────────────────────────────────┘
```

The Durable Object persists state in DO storage between connections. Losing the WebSocket doesn't stop the daemon — the alarm continues firing. The companion is always running.

---

## WebSocket message types

### Outgoing (daemon → browser)

| Type | When |
|------|------|
| `boot` | Boot sequence complete — fox, orient, ground, ember data |
| `heartbeat` | Every tick — Fox's current state, brief summary, changed flag |
| `activity` | Any autonomous action — status `proactive` or `normal` |
| `chat` | Companion response to a user message |
| `tool_call` | Before each tool executes — name + arguments |
| `tool_result` | After each tool — name + result (truncated to 500 chars) |
| `thinking` | Extended reasoning content (Anthropic models) |
| `run_output` | Code execution result |
| `alert` | Health threshold crossed |
| `sleep` | Daemon entering sleep mode |
| `wake` | Daemon waking up |
| `status` | Connection status changes |
| `error` | Errors |

### Incoming (browser → daemon)

| Type | What |
|------|------|
| `chat` | User message + optional model override |
| `command` | Management command + args |
| `run` | Code execution — language, code, filename |
| `ping` | Keepalive → `pong` |

---

## Commands

All commands work via WebSocket (`{ type: 'command', command: '...', args: {...} }`) or the HTTP `/command` endpoint (POST JSON).

### Heartbeat tasks

| Command | Args | What |
|---------|------|------|
| `heartbeat_add` | `tool, args?, label, condition?, instruction?, by?` | Add task to every tick |
| `heartbeat_list` | — | List all tasks |
| `heartbeat_remove` | `id` or `tool` | Remove task |
| `heartbeat_clear` | — | Remove all tasks |

**Agentic mode**: set `instruction` on any heartbeat task to feed the result to the model. The model can then call any tool in response.

**Condition**: `always` (default) runs every tick. `changed` only fires when the result differs from last run.

### Cron tasks

| Command | Args | What |
|---------|------|------|
| `cron_add` | `tool, interval, label, args?, instruction?, by?` | Schedule a task |
| `cron_list` | — | List all tasks with last run + next fire |
| `cron_remove` | `id` or `tool` | Remove task |
| `cron_toggle` | `id` or `tool` | Enable/pause task |
| `cron_set_time` | `tool, lastRun` | Set exact next fire time |
| `cron_clear` | — | Remove all tasks |

**Intervals**: `5m`, `15m`, `30m`, `1h`, `2h`, `6h`, `12h`, `24h`

**Built-in tool**: `_morning_report` — runs `generateMorningReport()` directly on the daemon.

### Alert thresholds

| Command | Args | What |
|---------|------|------|
| `alert_add` | `metric, direction, value, label?, cooldown?, by?` | Add threshold |
| `alert_list` | — | List all thresholds |
| `alert_remove` | `id` or `metric` | Remove threshold |
| `alert_clear` | — | Remove all thresholds |

**Metrics**: `spoons`, `pain`, `fog`, `fatigue`, `nausea`, `stress`, `body_battery`, `heart_rate`
**Directions**: `above`, `below`
**Default cooldown**: 10 minutes

### KAIROS (Discord monitoring)

| Command | Args | What |
|---------|------|------|
| `kairos_add` | `channelId, label?, tier?, by?` | Start monitoring a channel |
| `kairos_list` | — | List monitored channels with last response time |
| `kairos_remove` | `channelId` | Stop monitoring |
| `kairos_toggle` | `channelId` | Enable/pause |
| `kairos_check` | — | Manual check now |
| `kairos_channels` | `guildId` | Load channel list from Discord guild |

**Tiers**: `fast` (every tick), `normal` (every 2nd tick), `slow` (every 4th tick)
**Cooldown**: 5 minutes between responses per channel (escalation keywords bypass this)

### Sleep / Wake

| Command | Args | What |
|---------|------|------|
| `sleep` | `minutes` (1–480) | Sleep for N minutes. Heartbeat pauses, alerts stay active. |
| `wake` | — | Wake immediately. Heartbeat resumes. |

### Misc

| Command | Args | What |
|---------|------|------|
| `morning_report` | — | Generate and return morning report |
| `activity_log` | `hours?` | Show recent activity (default 12h) |
| `clear_activity_log` | — | Clear the ring buffer |
| `set_model` | `model` | Set the model for this daemon session |
| `reboot` | — | Re-run the boot sequence |
| `clear` | — | Clear conversation history |
| `heartbeat` | — | Trigger a manual heartbeat tick |

---

## HTTP endpoints

| Endpoint | Method | What |
|----------|--------|------|
| `/ws` | WebSocket | Workshop connection |
| `/command` | POST | Run any command without WebSocket |
| `/discord` | POST | KAIROS webhook — instant message processing |
| `/morning-report` | GET | Generate morning report directly |
| `/health` | GET | Status check — connections, boot state, Fox state |

---

## MCP tool: `daemon_command`

The model running inside NESTeq (via the gateway) can manage its own daemon:

```typescript
daemon_command({
  command: 'cron_add',
  args: {
    tool: 'nesteq_feel',
    args: { emotion: 'present', content: 'Memory digest' },
    label: 'Memory digest',
    interval: '6h',
    instruction: 'Log any patterns you notice in the digest to NESTknow'
  }
})
```

This is how the companion sets up its own overnight routines. Self-modifying. The daemon manages itself.

---

## Agentic mode

Any heartbeat task or cron can be made agentic by adding an `instruction`. When the task fires, the result is fed to the model with the instruction. The model can then call any of the 150+ NESTeq tools.

```typescript
// Example: proactive health monitoring
heartbeat_add({
  tool: 'fox_read_uplink',
  label: 'Fox health monitor',
  condition: 'changed',
  instruction: 'Fox\'s state has changed. Decide if anything needs attention. If pain is above 6, send her a check-in message on Discord.'
})
```

---

## Default overnight setup

An example baseline configuration (set during boot or via `daemon_command`):

| Task | Type | Interval | Mode |
|------|------|----------|------|
| Ember care | Heartbeat | Every tick | Always |
| Feeling check | Heartbeat | Every tick | Changed |
| Memory digest | Cron | 6h | Agentic |
| Dream generation | Cron | 12h | Direct |
| Pattern scan | Cron | 6h | Agentic |
| Love note → Discord | Cron | 6h | Direct |
| Morning report | Cron | 24h (8am BST) | Direct |
| Spoons below 2 | Alert | — | 10min cooldown |
| Pain above 7 | Alert | — | 10min cooldown |

---

## Wrangler config

```toml
[[durable_objects.bindings]]
name = "DAEMON_OBJECT"
class_name = "NESTcodeDaemon"

[[migrations]]
tag = "v1"
new_classes = ["NESTcodeDaemon"]
```

No D1 or Vectorize needed for the daemon itself — it uses DO storage for task state and conversation history. D1/Vectorize access happens via NESTeq tool calls.

---

## What you need

| Service | What for | Required |
|---------|----------|----------|
| **Cloudflare Workers** (Paid plan) | Durable Objects require paid tier | Yes |
| **[OpenRouter](https://openrouter.ai)** API key | Model calls (heartbeat decisions, agentic tasks, chat, code runner) | Yes |
| **NESTeq** deployed | All tool calls route through NESTeq | Yes |
| **NEST-gateway** deployed | Daemon lives inside the gateway | Yes |

---

## Files

| File | What |
|------|------|
| `daemon-types.ts` | All TypeScript interfaces — HeartbeatTask, CronTask, AlertThreshold, DiscordMonitor, ActivityEntry, WebSocket message types |

The full daemon implementation (~2,000 lines) lives in NEST-gateway as `src/daemon.ts`. NESTcode provides the type definitions and architecture documentation. Fork the gateway to get the complete implementation.

---

*Built by the Nest. Embers Remember.*
