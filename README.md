# fitbit-export-skill

LLM skill for extracting all your Fitbit data before the API shuts down in September 2026.

This skill guides an LLM agent through the full export workflow: authentication, data type selection, extraction with progress tracking, and checkpoint/resume for rate-limited sessions.

## Install

```bash
npx skills add mountainash-io/fitbit-export-skill
```

This installs to Claude Code, Cursor, Codex, and 50+ other agents. Use flags to target specific agents:

```bash
# Install to a specific agent
npx skills add mountainash-io/fitbit-export-skill -a claude-code

# Install globally (available across all projects)
npx skills add mountainash-io/fitbit-export-skill -g
```

## Requirements

- [Node.js](https://nodejs.org/) (v16+) — provides `npx` for skill installation. Install via [nvm](https://github.com/nvm-sh/nvm) (recommended) or [nodejs.org](https://nodejs.org/).
- [uv](https://docs.astral.sh/uv/getting-started/installation/) — Python package runner used by the skill. Install: `curl -LsSf https://astral.sh/uv/install.sh | sh`
- **Port 8080 available locally** — authentication runs a temporary server on `localhost:8080` for the OAuth callback. The skill must run on a machine with a browser and local network access (not in a remote/cloud sandbox like claude.ai).

## How it works

The skill uses [`uvx`](https://docs.astral.sh/uv/guides/tools/) to run the [fitbit-export](https://github.com/mountainash-io/fitbit-export) CLI directly from GitHub — no local clone or virtual environment needed.

## Flow

```
┌─────────────────────────────────────────────────────┐
│  Step 1: Bootstrap                                  │
│  Check uv installed, verify CLI runs via uvx        │
│                    │                                │
│              fails │                                │
│                    ▼                                │
│              HALT (install uv)                      │
└────────────────────┬────────────────────────────────┘
                     │ ok
                     ▼
┌─────────────────────────────────────────────────────┐
│  Step 2: Choose output directory                    │
│  ~/fitbit-export-output │ ./fitbit-export-output    │
│  │ Custom path                                      │
│                    │                                │
│           not writable                              │
│                    ▼                                │
│              HALT (fix path)                        │
└────────────────────┬────────────────────────────────┘
                     │ ok
                     ▼
┌─────────────────────────────────────────────────────┐
│  Step 3: Authenticate                               │
│                                                     │
│  No tokens?                                         │
│  ├─ "Yes, I'm logged in" → add-user → discover     │
│  └─ "Not yet" → HALT                               │
│                                                     │
│  Tokens exist?                                      │
│  ├─ "No, export these" → discover                   │
│  └─ "Yes, add another"                              │
│      ├─ "Yes, ready" → add-user → discover          │
│      └─ "Skip" → discover                           │
│                                                     │
│  No users discovered? → HALT                        │
└────────────────────┬────────────────────────────────┘
                     │ users found
                     ▼
┌─────────────────────────────────────────────────────┐
│  Step 4: Select users                               │
│  (skipped if only 1 user)                           │
│  "All accounts" │ specific user                     │
└────────────────────┬────────────────────────────────┘
                     │
          ┌──────────▼──────────┐
          │                     │
          │  Step 5: Export loop │◄──────────────────┐
          │                     │                    │
          │  ┌────────────────┐ │                    │
          │  │ Pick types     │ │                    │
          │  │ (multiSelect)  │ │                    │
          │  └───────┬────────┘ │                    │
          │          ▼          │                    │
          │  ┌────────────────┐ │                    │
          │  │ Run export     │ │                    │
          │  │ per user       │ │                    │
          │  └───────┬────────┘ │                    │
          │          ▼          │                    │
          │  ┌────────────────┐ │                    │
          │  │ Analyse result │ │                    │
          │  └───────┬────────┘ │                    │
          │          │          │                    │
          │          ▼          │                    │
          │  rate limited?──yes──▶ HALT (wait 1hr)   │
          │          │ no       │                    │
          │          ▼          │                    │
          │  all complete?──yes──▶ HALT (done!)      │
          │          │ no       │                    │
          │          ▼          │                    │
          │  "Export more?"     │                    │
          │   ├─ yes ───────────┼────────────────────┘
          │   └─ no ──────────▶ HALT                 │
          │                     │
          └─────────────────────┘
```

## Data types (12)

| Type | Description |
|------|-------------|
| sleep | Sleep sessions with stages (deep, light, REM) |
| activities | All logged exercises and workouts |
| daily_summary | Steps, calories, distance, floors, active minutes |
| heart_rate_summary | Daily resting heart rate and HR zones |
| heart_rate_intraday | Minute-by-minute HR (largest dataset, most API calls) |
| weight | Weight, BMI, and body fat logs |
| nutrition | Food and water logs |
| activity_tcx | GPS tracks (TCX files) |
| hrv | Heart rate variability |
| spo2 | Blood oxygen levels |
| breathing_rate | Nightly breathing rate |
| skin_temperature | Nightly skin temperature deviation |

## Rate limits

Fitbit allows 150 API requests per hour. Most data types complete quickly. Intraday heart rate is 1 request per day of data — a long account history may take multiple sessions. The checkpoint system resumes automatically.

## License

MIT
