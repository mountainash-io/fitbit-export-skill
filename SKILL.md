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

Extract all Fitbit data before the September 2026 API shutdown.
Execute this skill top-to-bottom as a runbook. Each section is one step.

**Input:** None required.
**Output:** Raw JSON files + TCX GPS tracks in a user-chosen output directory.

## Reference

```
CLI_PREFIX = "uvx --from git+https://github.com/mountainash-io/fitbit-export fitbit-export"
TOKEN_DIR  = "~/.fitbit-export"
ALL_TYPES  = ["sleep", "activities", "daily_summary", "heart_rate_summary",
              "heart_rate_intraday", "weight", "nutrition", "activity_tcx",
              "hrv", "spo2", "breathing_rate", "skin_temperature"]
```

To run any CLI command: `Bash(CLI_PREFIX + " " + args)`

---

## Step 1: Bootstrap

```pseudocode
DISPLAY "Fitbit Export"
DISPLAY "Extract all your Fitbit data before the API shuts down (September 2026)."
DISPLAY ""

uv_check = Bash("command -v uv")
IF uv_check FAILS:
  DISPLAY "uv is required. Install: curl -LsSf https://astral.sh/uv/install.sh | sh"
  HALT

help_check = Bash(CLI_PREFIX + " --help")
IF help_check FAILS:
  DISPLAY "Failed to run fitbit-export via uvx. Check network connectivity and try again."
  HALT

DISPLAY "Environment ready."
```

---

## Step 2: Choose output directory

```pseudocode
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
  output_dir = user_provided_path
ELSE:
  output_dir = selection

# Verify writable
dir_check = Bash("mkdir -p " + output_dir + " && test -w " + output_dir)
IF dir_check FAILS:
  DISPLAY "Cannot write to {output_dir}. Check the path and permissions."
  HALT

Bash(CLI_PREFIX + " config --output " + output_dir)
```

---

## Step 3: Authenticate

```pseudocode
tokens = Bash("ls " + TOKEN_DIR + "/tokens-*.json 2>/dev/null")

IF tokens IS empty:
  # --- No accounts: first-time setup ---
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

  Bash(CLI_PREFIX + " add-user")

ELSE:
  # --- Existing accounts: offer to add more ---
  DISPLAY "Found existing Fitbit account(s)."

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

    IF ready == "Yes, ready":
      Bash(CLI_PREFIX + " add-user")

# Discover all authenticated users
# list-users outputs lines like: "  Alice (ABC123)  4/12"
user_list_output = Bash(CLI_PREFIX + " list-users")
users = []
FOR line IN user_list_output.lines:
  MATCH line against pattern: "{name} ({user_id})"
  IF match:
    users.append({ user_id: match.user_id, display_name: match.name })

IF len(users) == 0:
  DISPLAY "No authenticated Fitbit accounts found."
  DISPLAY "The browser authorization may not have completed."
  DISPLAY "Run this skill again to retry."
  HALT

DISPLAY "Authenticated accounts:"
FOR user IN users:
  DISPLAY "  - {user.display_name} ({user.user_id})"
```

---

## Step 4: Select users

```pseudocode
IF len(users) == 1:
  selected_users = users
ELSE:
  options = [{ label: "All accounts", description: "Export all {len(users)} accounts" }]
  FOR user IN users:
    options.append({ label: "{user.display_name} ({user.user_id})", description: "" })

  selection = AskUserQuestion(
    question: "Which accounts do you want to export?",
    header: "Accounts",
    options: options,
    multiSelect: false
  )

  IF selection == "All accounts":
    selected_users = users
  ELSE:
    selected_users = [u FOR u IN users IF "{u.display_name} ({u.user_id})" == selection]
```

---

## Step 5: Select types, export, and loop

This step repeats until all types are done, the user declines to continue, or
the API rate limit is hit.

```pseudocode
LOOP:
  # --- Pick data types ---
  chosen = AskUserQuestion(
    question: "Which data types do you want to export?",
    header: "Data Types",
    options: [
      { label: "sleep",              description: "Sleep sessions with stages (deep, light, REM)" },
      { label: "activities",          description: "All logged exercises and workouts" },
      { label: "daily_summary",       description: "Steps, calories, distance, floors, active minutes" },
      { label: "heart_rate_summary",  description: "Daily resting heart rate and HR zones" },
      { label: "heart_rate_intraday", description: "Minute-by-minute HR — largest dataset, uses most API calls" },
      { label: "weight",             description: "Weight, BMI, and body fat logs" },
      { label: "nutrition",          description: "Food and water logs" },
      { label: "activity_tcx",       description: "GPS tracks (TCX files) for activities" },
      { label: "hrv",                description: "Heart rate variability" },
      { label: "spo2",               description: "Blood oxygen levels" },
      { label: "breathing_rate",     description: "Nightly breathing rate" },
      { label: "skin_temperature",   description: "Nightly skin temperature deviation" }
    ],
    multiSelect: true
  )

  IF len(chosen) == 0:
    DISPLAY "No data types selected."
    HALT

  type_flag = "--types " + ",".join(chosen)

  # --- Run export for each selected user ---
  FOR user IN selected_users:
    first_name = user.display_name.split()[0].lower()
    user_dir = output_dir + "/" + user.user_id + "-" + first_name

    DISPLAY "Exporting {user.display_name} ({user.user_id})..."

    # Show resume info if checkpoint exists
    checkpoint_raw = Read(user_dir + "/.checkpoint.json")
    IF checkpoint_raw IS NOT empty:
      checkpoint = JSON.parse(checkpoint_raw)
      completed = checkpoint.completed OR []
      in_progress = checkpoint.in_progress OR {}
      DISPLAY "  Resuming — already completed: {', '.join(completed)}"
      FOR dtype, progress IN in_progress:
        DISPLAY "  In progress: {dtype} (through {progress.last_completed_date OR 'unknown'})"

    # Run the CLI
    result = Bash(
      CLI_PREFIX + " export " + type_flag + " --user " + user.user_id + " --output " + output_dir,
      timeout: 600000
    )
    DISPLAY result

  # --- Analyse outcome ---
  # Re-read checkpoints to determine overall status
  all_complete = true
  any_rate_limited = "429" IN result OR "rate limit" IN result.lower()

  FOR user IN selected_users:
    first_name = user.display_name.split()[0].lower()
    user_dir = output_dir + "/" + user.user_id + "-" + first_name
    checkpoint_raw = Read(user_dir + "/.checkpoint.json")

    IF checkpoint_raw IS empty:
      DISPLAY "Export may have failed for {user.display_name}. Check output above."
      all_complete = false
      CONTINUE

    checkpoint = JSON.parse(checkpoint_raw)
    completed = checkpoint.completed OR []
    remaining = [t FOR t IN ALL_TYPES IF t NOT IN completed]

    IF len(remaining) > 0:
      all_complete = false
      DISPLAY "{user.display_name}: {len(completed)}/{len(ALL_TYPES)} types complete, {len(remaining)} remaining"

      in_progress = checkpoint.in_progress OR {}
      IF "heart_rate_intraday" IN remaining AND "heart_rate_intraday" IN in_progress:
        last_date = in_progress["heart_rate_intraday"].last_completed_date OR "unknown"
        DISPLAY "  Intraday HR progress: through {last_date} (largest dataset, ~32hrs for 13yrs)"
    ELSE:
      DISPLAY "{user.display_name}: all {len(ALL_TYPES)} types complete!"

  # --- Decide next action ---
  IF any_rate_limited:
    DISPLAY ""
    DISPLAY "Hit Fitbit's rate limit (150 requests/hour)."
    DISPLAY "Progress is saved — run this skill again in about 1 hour to continue."
    HALT

  IF all_complete:
    DISPLAY ""
    DISPLAY "All data types exported for all selected users!"
    DISPLAY "Data saved to: {output_dir}"
    HALT

  # Not everything done, not rate limited — offer to continue
  again = AskUserQuestion(
    question: "Export completed for selected types. Want to export more?",
    header: "Continue",
    options: [
      { label: "Yes, choose more types", description: "Select additional types to export" },
      { label: "No, I'm done",           description: "Finish" }
    ],
    multiSelect: false
  )

  IF again == "No, I'm done":
    DISPLAY ""
    DISPLAY "Remaining types can be exported by running this skill again."
    HALT

  # Loop back to type selection
```
