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

## Constants

```pseudocode
UVX_PREFIX = "uvx --from git+https://github.com/mountainash-io/fitbit-export fitbit-export"
TOKEN_DIR  = "~/.fitbit-export"
ALL_TYPES  = ["spo2", "breathing_rate", "skin_temperature", "hrv",
              "weight", "sleep", "heart_rate_summary", "nutrition",
              "activities", "activity_tcx", "daily_summary",
              "heart_rate_intraday"]
```

## State

All phases read and write from this shared state, initialised at startup:

```pseudocode
state = {
  output_dir: null,     # Path — where export data is saved
  users: [],            # list[{user_id: str, display_name: str}] — authenticated accounts
  selected_users: [],   # list[{user_id: str, display_name: str}] — accounts to export
  type_flag: null,      # str — CLI flag: "--all" or "--types spo2,weight,..."
}
```

## Helper: RUN

All phases use this helper to invoke CLI commands via uvx:

```pseudocode
RUN(args):
  RETURN Bash(UVX_PREFIX + " " + args)
```

---

## Phase 0: Bootstrap

Ensure `uv` is available and `fitbit-export` is runnable.

```pseudocode
BOOTSTRAP():
  uv_check = Bash("command -v uv")
  IF uv_check FAILS:
    DISPLAY "uv is required. Install: curl -LsSf https://astral.sh/uv/install.sh | sh"
    HALT

  version_check = RUN("--help")
  IF version_check FAILS:
    DISPLAY "Failed to run fitbit-export via uvx. Check network connectivity and try again."
    HALT

  DISPLAY "Environment ready."
```

---

## Phase 1: Output Directory

Ask the user where to store the export. Sets `state.output_dir`.

```pseudocode
CHOOSE_OUTPUT_DIR():
  selection = AskUserQuestion(
    question: "Where should the exported Fitbit data be saved?",
    header: "Output",
    options: [
      { label: "~/fitbit-export-output", description: "Home directory (Recommended)" },
      { label: "./fitbit-export-output", description: "Current directory" },
      { label: "Custom path", description: "You specify the path" }
    ],
    multiSelect: false
  )

  IF selection == "Custom path":
    WAIT for user to provide path
    state.output_dir = user_provided_path
  ELSE:
    state.output_dir = selection

  RUN("config --output " + state.output_dir)
```

---

## Phase 2: Authentication

Discover existing users or authenticate new ones. Sets `state.users`.

```pseudocode
AUTHENTICATE():
  tokens = Bash("ls " + TOKEN_DIR + "/tokens-*.json 2>/dev/null")

  IF tokens IS empty:
    DISPLAY "No Fitbit accounts found. Let's connect your first account."
    DISPLAY ""
    DISPLAY "Before proceeding:"
    DISPLAY "  1. Log into your Fitbit account at fitbit.com in your browser"
    DISPLAY "  2. Make sure port 8080 is free locally (used for the OAuth callback)"
    DISPLAY ""
    DISPLAY "The tool will open a browser window to authorize API access."
    DISPLAY "After you authorize, you'll be redirected back automatically."
    DISPLAY ""

    confirm = AskUserQuestion(
      question: "Are you logged into fitbit.com and ready to authorize?",
      header: "Auth",
      options: [
        { label: "Yes, I'm logged in", description: "Opens Fitbit authorization page" },
        { label: "Not yet", description: "I need to log in first" }
      ],
      multiSelect: false
    )

    IF confirm == "Not yet":
      DISPLAY "No problem. Run this skill again when you're ready."
      HALT

    RUN("add-user")
    state.users = DISCOVER_USERS()

  ELSE:
    state.users = DISCOVER_USERS()
    DISPLAY "Found {len(state.users)} authenticated Fitbit account(s):"
    FOR user IN state.users:
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
      DISPLAY "To add a different account:"
      DISPLAY "  1. Log out of fitbit.com in your browser"
      DISPLAY "  2. Log in as the new user"
      DISPLAY "  3. Then confirm below"
      DISPLAY ""

      ready = AskUserQuestion(
        question: "Are you logged into fitbit.com as the new user?",
        header: "Auth",
        options: [
          { label: "Yes, ready", description: "Opens authorization for the new account" },
          { label: "Skip", description: "Continue with existing accounts only" }
        ],
        multiSelect: false
      )

      IF ready == "Skip":
        RETURN  # state.users already populated above

      RUN("add-user")
      state.users = DISCOVER_USERS()


DISCOVER_USERS():
  # list-users outputs lines like: "  Alice (ABC123)  4/12"
  output = RUN("list-users")
  users = []
  FOR line IN output.lines:
    MATCH line against pattern: "{name} ({user_id})"
    IF match:
      users.append({ user_id: match.user_id, display_name: match.name })
  RETURN users
```

---

## Phase 3: Select Users

Let the user choose which accounts to export. Sets `state.selected_users`.

```pseudocode
SELECT_USERS():
  IF len(state.users) == 1:
    state.selected_users = state.users
    RETURN

  options = [{ label: "All accounts", description: "Export all {len(state.users)} accounts" }]
  FOR user IN state.users:
    options.append({ label: "{user.display_name} ({user.user_id})", description: "" })

  selection = AskUserQuestion(
    question: "Which accounts do you want to export?",
    header: "Accounts",
    options: options,
    multiSelect: false
  )

  IF selection == "All accounts":
    state.selected_users = state.users
  ELSE:
    matched = [u FOR u IN state.users IF "{u.display_name} ({u.user_id})" == selection]
    state.selected_users = matched
```

---

## Phase 4: Select Data Types

Let the user choose which data types to export. Sets `state.type_flag`.

```pseudocode
SELECT_TYPES():
  selection = AskUserQuestion(
    question: "Which data types do you want to export?",
    header: "Data Types",
    options: [
      { label: "All types",                description: "Export all 12 data types" },
      { label: "Quick (light API usage)",   description: "spo2, breathing_rate, skin_temperature, hrv, weight, sleep" },
      { label: "spo2",                      description: "Blood oxygen levels" },
      { label: "breathing_rate",            description: "Nightly breathing rate" },
      { label: "skin_temperature",          description: "Nightly skin temperature deviation" },
      { label: "hrv",                       description: "Heart rate variability" },
      { label: "weight",                    description: "Weight, BMI, body fat" },
      { label: "sleep",                     description: "Sleep sessions with stages" },
      { label: "heart_rate_summary",        description: "Daily resting HR and zones" },
      { label: "nutrition",                 description: "Food and water logs" },
      { label: "activities",                description: "Logged exercises and workouts" },
      { label: "activity_tcx",              description: "GPS tracks (TCX files)" },
      { label: "daily_summary",             description: "Steps, calories, distance, floors" },
      { label: "heart_rate_intraday",       description: "Minute-by-minute HR (largest, slow)" }
    ],
    multiSelect: true
  )

  IF "All types" IN selection:
    state.type_flag = "--all"
  ELIF "Quick (light API usage)" IN selection:
    state.type_flag = "--types spo2,breathing_rate,skin_temperature,hrv,weight,sleep"
  ELSE:
    state.type_flag = "--types " + ",".join(selection)
```

---

## Phase 5: Extract

Run the extraction with progress monitoring.

```pseudocode
EXTRACT():
  FOR user IN state.selected_users:
    DISPLAY "Exporting {user.display_name} ({user.user_id})..."
    DISPLAY ""

    # Derive user directory name (matches CLI convention: "{user_id}-{first_name_lower}")
    first_name = user.display_name.split()[0].lower()
    user_dir = state.output_dir + "/" + user.user_id + "-" + first_name

    # Check for existing checkpoint
    checkpoint_path = user_dir + "/.checkpoint.json"
    checkpoint_raw = Read(checkpoint_path)

    IF checkpoint_raw IS NOT empty:
      checkpoint = JSON.parse(checkpoint_raw)
      completed = checkpoint.completed OR []
      in_progress = checkpoint.in_progress OR {}
      DISPLAY "Resuming previous export."
      DISPLAY "  Already completed: {', '.join(completed)}"
      FOR dtype, progress IN in_progress:
        last_date = progress.last_completed_date OR "unknown"
        DISPLAY "  In progress: {dtype} (last: {last_date})"
      DISPLAY ""

    # Run the export
    result = RUN(
      "export {state.type_flag} --user {user.user_id} --output {state.output_dir}",
      timeout: 600000  # 10 minutes max per run
    )

    DISPLAY result
    ANALYSE_RESULT(result, user_dir)
```

---

## Phase 6: Analyse and Advise

Parse the export result and advise the user on next steps.

```pseudocode
ANALYSE_RESULT(result, user_dir):
  checkpoint_raw = Read(user_dir + "/.checkpoint.json")

  IF checkpoint_raw IS empty:
    DISPLAY "Export may have failed before writing any data. Check the output above."
    RETURN

  checkpoint = JSON.parse(checkpoint_raw)
  completed = checkpoint.completed OR []
  remaining = [t FOR t IN ALL_TYPES IF t NOT IN completed]

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

    in_progress = checkpoint.in_progress OR {}
    IF "heart_rate_intraday" IN remaining AND "heart_rate_intraday" IN in_progress:
      last_date = in_progress["heart_rate_intraday"].last_completed_date OR "unknown"
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
  CHOOSE_OUTPUT_DIR()

  # Gate: verify output directory is set and writable
  IF state.output_dir IS null:
    DISPLAY "No output directory selected."
    HALT
  dir_check = Bash("mkdir -p " + state.output_dir + " && test -w " + state.output_dir)
  IF dir_check FAILS:
    DISPLAY "Cannot write to {state.output_dir}. Check the path and permissions."
    HALT

  AUTHENTICATE()

  # Gate: verify at least one user was discovered
  IF len(state.users) == 0:
    DISPLAY "No authenticated Fitbit accounts found."
    DISPLAY "The browser authorization may not have completed."
    DISPLAY "Run this skill again to retry."
    HALT

  SELECT_USERS()
  SELECT_TYPES()

  # Gate: verify selections are populated
  IF len(state.selected_users) == 0:
    DISPLAY "No users selected for export."
    HALT
  IF state.type_flag IS null:
    DISPLAY "No data types selected for export."
    HALT

  EXTRACT()
```
