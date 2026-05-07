---
name: fitbit-export
description: >
  Extract all Fitbit data before the API shuts down (September 2026).
  Authenticates via OAuth2+PKCE (opens browser), extracts 12 data types
  to raw JSON, handles rate limiting and checkpoint/resume.
  Supports multiple Fitbit accounts (family export).
allowed-tools: Bash, Read, AskUserQuestion
---

# Fitbit Export

Interactive skill to extract all Fitbit data before the September 2026 API shutdown.
Guides the user through setup, authentication, and extraction with checkpoint/resume.

**Input:** None required. The skill manages everything.
**Output:** Raw JSON files + TCX GPS tracks in a user-chosen output directory.

---

## Phase 0: Bootstrap

Ensure `uv` is available. All commands use `uvx` to run `fitbit-export` directly
from GitHub — no local clone or venv needed.

```pseudocode
BOOTSTRAP():
  uv_check = Bash("command -v uv")
  IF uv_check FAILS:
    DISPLAY "uv is required. Install: curl -LsSf https://astral.sh/uv/install.sh | sh"
    HALT

  # Verify fitbit-export is runnable via uvx
  version_check = Bash("uvx --from git+https://github.com/mountainash-io/fitbit-export fitbit-export --help")
  IF version_check FAILS:
    DISPLAY "Failed to run fitbit-export via uvx. Check network connectivity and try again."
    HALT

  DISPLAY "Environment ready."
```

### Helper: RUN

All phases use this helper to invoke CLI commands via uvx:

```pseudocode
RUN(args):
  RETURN Bash("uvx --from git+https://github.com/mountainash-io/fitbit-export fitbit-export " + args)
```

---

## Phase 1: Output Directory

Ask the user where to store the export.

```pseudocode
OUTPUT_DIR():
  output = AskUserQuestion(
    question: "Where should the exported Fitbit data be saved?",
    header: "Output",
    options: [
      { label: "~/fitbit-export-output", description: "Home directory (Recommended)" },
      { label: "./fitbit-export-output", description: "Current directory" },
      { label: "Custom path", description: "You specify the path" }
    ],
    multiSelect: false
  )

  IF output == "Custom path":
    WAIT for user input

  RUN("config --output {output_dir}")
  RETURN output_dir
```

---

## Phase 2: Authentication

Discover existing users or authenticate new ones.

```pseudocode
AUTHENTICATE(output_dir):
  tokens = Bash("ls ~/.fitbit-export/tokens-*.json 2>/dev/null")

  IF tokens IS empty:
    DISPLAY "No Fitbit accounts found. Let's connect your first account."
    DISPLAY ""
    DISPLAY "A browser window will open for you to log into Fitbit."
    DISPLAY "After you authorize, you'll be redirected back automatically."
    DISPLAY ""

    confirm = AskUserQuestion(
      question: "Ready to open the browser for Fitbit authorization?",
      header: "Auth",
      options: [
        { label: "Yes, open browser", description: "Opens fitbit.com login page" },
        { label: "Not yet", description: "I need to do something first" }
      ],
      multiSelect: false
    )

    IF confirm == "Not yet":
      DISPLAY "No problem. Run this skill again when you're ready."
      HALT

    RUN("add-user")
    users = DISCOVER_USERS()

  ELSE:
    users = DISCOVER_USERS()
    DISPLAY "Found {len(users)} authenticated Fitbit account(s):"
    FOR user IN users:
      DISPLAY "  - {user.display_name} ({user.user_id})"

    add_more = AskUserQuestion(
      question: "Want to add another Fitbit account (e.g., family member)?",
      header: "Users",
      options: [
        { label: "No, export these", description: "Proceed with found accounts" },
        { label: "Yes, add another", description: "Opens browser for another Fitbit login" }
      ],
      multiSelect: false
    )

    IF add_more == "Yes, add another":
      DISPLAY "Log out of fitbit.com first if adding a different account."
      RUN("add-user")
      users = DISCOVER_USERS()

  RETURN users


DISCOVER_USERS():
  result = RUN("list-users")
  PARSE user_id and display_name from output
  RETURN users
```

---

## Phase 3: Select Users

Let the user choose which accounts to export.

```pseudocode
SELECT_USERS(users):
  IF len(users) == 1:
    RETURN users

  selection = AskUserQuestion(
    question: "Which accounts do you want to export?",
    header: "Accounts",
    options: [
      { label: "All accounts", description: "Export all {len(users)} accounts" },
      -- dynamically add one option per user --
    ],
    multiSelect: false
  )

  IF selection == "All accounts":
    RETURN users
  ELSE:
    RETURN [selected_user]
```

---

## Phase 4: Extract

Run the extraction with progress monitoring.

```pseudocode
EXTRACT(output_dir, users):
  FOR user IN users:
    DISPLAY "Exporting {user.display_name} ({user.user_id})..."
    DISPLAY ""

    # Check for existing checkpoint
    user_dir = "{output_dir}/{user.user_id}-{user.first_name}"
    checkpoint = Read("{user_dir}/.checkpoint.json")

    IF checkpoint EXISTS:
      completed = checkpoint.completed
      in_progress = checkpoint.in_progress
      DISPLAY "Resuming previous export."
      DISPLAY "  Already completed: {', '.join(completed)}"
      IF in_progress:
        FOR dtype, progress IN in_progress:
          DISPLAY "  In progress: {dtype} (last: {progress.last_completed_date})"
      DISPLAY ""

    # Run the export
    result = RUN(
      "export --all --user {user.user_id} --output {output_dir}",
      timeout: 600000  # 10 minutes max per run
    )

    DISPLAY result

    # Check outcome
    ANALYSE_RESULT(result, user_dir)
```

---

## Phase 5: Analyse and Advise

Parse the export result and advise the user on next steps.

```pseudocode
ANALYSE_RESULT(result, user_dir):
  checkpoint = Read("{user_dir}/.checkpoint.json")

  IF checkpoint IS null:
    DISPLAY "Export may have failed before writing any data. Check the output above."
    RETURN

  completed = checkpoint.completed
  all_types = ["spo2", "breathing_rate", "skin_temperature", "hrv",
               "weight", "sleep", "heart_rate_summary", "nutrition",
               "activities", "activity_tcx", "daily_summary",
               "heart_rate_intraday"]

  remaining = [t for t IN all_types IF t NOT IN completed]

  IF len(remaining) == 0:
    DISPLAY ""
    DISPLAY "Export complete! All 12 data types extracted."
    DISPLAY "Data saved to: {user_dir}/raw/"
    RETURN

  # Check if rate limited
  rate_limited = "429" IN result OR "rate limit" IN result.lower()

  IF rate_limited:
    DISPLAY ""
    DISPLAY "Hit Fitbit's rate limit (150 requests/hour)."
    DISPLAY "Completed so far: {', '.join(completed)}"
    DISPLAY "Remaining: {', '.join(remaining)}"
    DISPLAY ""

    IF "heart_rate_intraday" IN remaining:
      IF "heart_rate_intraday" IN checkpoint.in_progress:
        last_date = checkpoint.in_progress["heart_rate_intraday"]["last_completed_date"]
        DISPLAY "Intraday HR progress: extracted through {last_date}"
        DISPLAY "This is the largest dataset — it takes ~32 hours for 13 years of data."
        DISPLAY ""

    DISPLAY "Run this skill again in about 1 hour when the rate limit resets."
    DISPLAY "Progress is saved — it will pick up exactly where it left off."

  ELSE:
    DISPLAY ""
    DISPLAY "Some data types could not be extracted."
    DISPLAY "Completed: {', '.join(completed)}"
    DISPLAY "Remaining: {', '.join(remaining)}"
    DISPLAY ""
    DISPLAY "Check the error messages above. Common causes:"
    DISPLAY "  - SpO2/skin temp: device may not support these sensors"
    DISPLAY "  - 400 errors: endpoint may not support the requested date range"
    DISPLAY ""
    DISPLAY "Run this skill again to retry the failed types."
```

---

## Orchestrator

Main entry point tying all phases together.

```pseudocode
MAIN():
  DISPLAY "Fitbit Export"
  DISPLAY "Extract all your Fitbit data before the API shuts down (September 2026)."
  DISPLAY ""

  BOOTSTRAP()
  output_dir = OUTPUT_DIR()
  users      = AUTHENTICATE(output_dir)
  selected   = SELECT_USERS(users)
  EXTRACT(output_dir, selected)
```
