# fitbit-export-skill

LLM skill for extracting all your Fitbit data before the API shuts down in September 2026.

This skill guides an LLM agent through the full export workflow: authentication, user selection, data extraction with progress tracking, and checkpoint/resume for rate-limited sessions.

## How it works

The skill uses [`uvx`](https://docs.astral.sh/uv/guides/tools/) to run the [fitbit-export](https://github.com/mountainash-io/fitbit-export) CLI directly from GitHub — no local clone or virtual environment needed. The only prerequisite is [uv](https://docs.astral.sh/uv/).

## Data types exported (12)

activities, activity_tcx, sleep, heart_rate_summary, heart_rate_intraday,
hrv, spo2, breathing_rate, skin_temperature, weight, daily_summary, nutrition

## Rate limits

Fitbit allows 150 API requests per hour. Most data types complete quickly. Intraday heart rate is 1 request per day of data — a long account history may take multiple sessions. The checkpoint system resumes automatically.

## License

MIT
