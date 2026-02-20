# ARIM — AI Nutrition Tracker

> Telegram-based nutrition tracking assistant powered by n8n, Google Gemini, and Google Sheets.

---

## Overview

ARIM (AI Registration & Intake Manager) is an n8n workflow that enables users to log meals, track daily nutrition goals, and receive personalized reports entirely through Telegram. It handles new user registration via a conversational state machine, processes text, voice, and image inputs, and stores all data in Google Sheets.

---

## Architecture

```
Telegram Trigger
    └── Typing indicator
    └── Check Cache (static data)
        ├── [Cached = registered] → Input Message Router
        └── [Not cached] → Registered? (Google Sheets lookup)
            ├── [User_ID found] → Input Message Router
            └── [User_ID empty] → Registration Flow

Input Message Router
    ├── Text    → get_message (text) → Validate Input
    ├── Voice   → Download → Fix MIME → Analyze (Gemini) → Validate Input
    ├── Image   → Download → Fix MIME → Analyze (Gemini) → Validate Input
    └── Other   → get_error_message → Validate Input

Validate Input → Has Content?
    ├── [Yes] → ARIM Agent (Gemini) → MarkdownV2 formatter → Send message
    └── [No]  → Fallback Message → Send Fallback

Registration Flow
    └── get_message (register)
    └── Register Agent (Code — state machine)
    └── Registration Done?
        ├── [Yes] → Write Registration (Sheets) → Save to Cache → MarkdownV → Send message
        └── [No]  → Get Reg Reply → MarkdownV → Send message (ask next question)

Report Subworkflow (triggered by ARIM tool)
    └── Get Meals Info + Get User Info (parallel)
    └── Unify data → Merge → Get chart message → Send back message
```

---

## Features

| Feature | Details |
|---|---|
| **Multi-step registration** | Conversational state machine — collects name, calorie target, protein target. Calculates targets from biometrics if user doesn't know them (Mifflin-St Jeor + 1.4 activity multiplier). |
| **Meal logging** | Accepts text descriptions, voice messages (transcribed by Gemini), and food photos (analyzed by Gemini Vision). |
| **Daily reports** | Summarizes calories, protein, carbs, and fats with progress bars against user targets. |
| **Profile updates** | Users can update name, calorie target, or protein target at any time. |
| **In-memory cache** | Workflow static data caches registered users to skip the Google Sheets lookup on every message. |
| **MarkdownV2 safe output** | All Telegram messages are escaped and chunked to respect the 4096-character limit. |

---

## Stack

| Component | Tool |
|---|---|
| Workflow engine | [n8n](https://n8n.io) |
| AI / Vision / Voice | Google Gemini 2.5 Pro |
| Chat model (ARIM agent) | Google Gemini 2.0 Flash Lite |
| Database | Google Sheets |
| Messaging | Telegram Bot API |

---

## Google Sheets Schema

### `Profile` sheet

| Column | Type | Description |
|---|---|---|
| `User_ID` | string | Telegram `chat.id` |
| `Name` | string | User's display name |
| `Calories_target` | number | Daily calorie goal (kcal) |
| `Protein_target` | number | Daily protein goal (g) |

### `Meals` sheet

| Column | Type | Description |
|---|---|---|
| `User_ID` | string | Telegram `chat.id` |
| `Date` | string | Format: `YYYY-MM-DD` |
| `Meal_description` | string | Free-text meal description |
| `Calories` | number | kcal |
| `Proteins` | number | g |
| `Carbs` | number | g |
| `Fats` | number | g |

A template spreadsheet with the correct headers is available here:
[Google Sheets Template](https://docs.google.com/spreadsheets/d/11kI8q0oB2vPzbVJOItdna5o0y7szuoprJSA8v0Bt-Ec/edit?usp=sharing)

---

## Setup

### 1. Prerequisites

- n8n instance (self-hosted or cloud)
- Telegram Bot Token (via [@BotFather](https://t.me/botfather))
- Google Cloud project with Gemini API enabled
- Google Sheets OAuth2 credential configured in n8n

### 2. Credentials required

| Credential | Used by |
|---|---|
| `telegramApi` | Telegram Trigger, all send/download nodes |
| `googlePalmApi` | Gemini image and voice analysis nodes |
| `googleSheetsOAuth2Api` | All Google Sheets read/write nodes |

### 3. Import

1. Import `Nutrition_Tracker_Fixed_v5_1.json` into n8n via **Workflows → Import from file**.
2. Open each credential placeholder and connect your own credentials.
3. Update the `documentId` in all Google Sheets nodes to point to your spreadsheet.
4. Activate the workflow.

> **Do not click "Listen for test event" while the workflow is active.** The Telegram Trigger can only run in one mode at a time. To test, deactivate the workflow first, capture a test event, then reactivate — or use pinned execution data from a previous run.

---

## Registration Flow (state machine)

The `Register Agent` node is a pure Code node. It uses `$getWorkflowStaticData('global')` to persist conversation state between messages without requiring a database or session service.

```
step: name
  → ask for name

step: know_targets
  → yes → step: get_calories
  → no  → step: get_weight

step: get_calories → step: get_protein → step: confirm
step: get_weight → step: get_height → step: get_age → step: get_goal → step: confirm

step: confirm
  → yes → write to Sheets, cache user, send confirmation
  → no  → reset to step: name
```

Calorie and protein targets are calculated using:
- **BMR**: Mifflin-St Jeor formula
- **TDEE**: BMR × 1.4 (moderate activity)
- **Protein**: 1.8–2.2 g/kg depending on goal

---

## ARIM Agent Tools

The main ARIM AI agent (registered users path) has access to four tools:

| Tool | Purpose |
|---|---|
| `appendMealData` | Appends a structured meal entry to the Meals sheet |
| `getProfileData` | Reads the user's current profile targets |
| `updateProfileData` | Updates name, calorie target, or protein target |
| `getReport` | Triggers the report subworkflow for a given date |

---

## Known Limitations

- **Telegram Trigger test mode conflict** — n8n cannot listen for test events while the workflow is active. Use pinned data or deactivate before testing.
- **Static data persistence** — The in-memory cache (`$getWorkflowStaticData`) does not survive an n8n server restart. After a restart, the first message from each user will re-trigger the Google Sheets lookup before routing correctly. This is non-breaking.
- **Single gender assumption** — The BMR calculator defaults to male parameters. Add a gender collection step to `Register Agent` if accuracy for all users is required.
- **No message deduplication** — If Telegram delivers the same webhook twice (rare but possible), the workflow will process it twice.

---

## License

MIT
