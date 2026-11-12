# Pacevera Fitness Decision Engine

Pacevera is a deterministic fitness decision engine exposed through the Model Context Protocol (MCP). It receives normalized training and health evidence supplied by the calling AI host, evaluates recovery and training constraints, and returns a structured decision.

The primary operation is to evaluate an existing scheduled session. A decision describes the change from the scheduled session to the resulting session:

```text
scheduled session (from) -> resulting session (to)
```

Each decision includes the relevant evidence, applied rules, confidence, signal coverage, and limitations.

## Scope

Pacevera provides the following capabilities:

- Deterministic calculation of recovery state, readiness, muscle-group fatigue, and training load.
- Session decisions with the types `keep`, `adjust`, `substitute`, `defer`, and `advance`.
- Constraint handling for injuries, equipment, available time, and exercise substitutions.
- Training-plan generation and caller-owned plan adjustment through preview and commit operations.
- Structured decision provenance through rule identifiers, measured values, source information, and version metadata.
- Normalization support for Apple Health, Google Health Takeout, Garmin, Strava, Oura, and WHOOP data shapes.

Apart from two optional connections you turn on yourself — Google Health and the Garmin Personal API sync, both described below — Pacevera does not connect to these providers on behalf of the user. Otherwise the calling host obtains provider exports or API responses, or the user places exports in the selected folder.

## Architecture

```text
AI host
  └─ Collects the user's question and evidence
       └─ MCP server / desktop extension
            ├─ Evidence normalization
            ├─ Fitness-state calculation
            ├─ Training-load calculation
            ├─ Deterministic rule evaluation
            ├─ Decision and plan operations
            └─ Structured result: Decision / Action / Reason
```

The AI host is responsible for interpreting the user's request, collecting evidence, selecting tools, and presenting the result. Pacevera performs the calculations and rule evaluation; it does not call a model to generate a decision. Given the same code, rule set, and input, the result is deterministic.

## Decision and recommendation

| Type | Description |
|---|---|
| Recommendation | A training suggestion generated without a required prior session or plan. |
| Decision | A traceable change applied to an existing scheduled session or plan. |

`decide_session` requires `scheduledSession` to establish the prior state. If no scheduled session is provided, the tool returns `no_scheduled_session` rather than creating a session. Use `generate_plan` when a plan is required.

Decision output follows five layers:

```text
Evidence -> Fitness State -> Decision -> Action -> Reason
```

Decision intent and resulting action are separate fields. The same intent may produce different actions depending on the user's equipment, available time, injuries, and scheduled session.

## MCP tools

### Decision and planning tools

| Tool | Description |
|---|---|
| `assess_fitness_state` | Returns recovery, readiness, muscle-group fatigue, and training-load state. |
| `decide_session` | Evaluates a scheduled session and returns the resulting session. |
| `decide_exercise_substitution` | Selects a replacement movement subject to injury and equipment constraints. |
| `generate_plan` | Generates a periodized training plan. |
| `preview_adjust_plan` | Returns a deterministic patch and diff for a caller-owned plan. |
| `commit_adjust_plan` | Validates the plan version and applies a caller-owned patch. |

### Supporting and read tools

| Tool | Description |
|---|---|
| `get_evidence_coverage` | Reports available and missing evidence signals. |
| `explain_decision` | Returns the process-local rule and source trace for a previous decision. |
| `submit_outcome` | Accepts an observed outcome for a prior case; storage is bounded and local to the configured environment. |
| `search_exercises` / `get_exercise` | Queries the exercise catalog and graph relationships. |
| `search_workouts` / `get_workout` | Queries structured workout content. |
| `get_user_profile` | Returns caller-supplied user constraints and training settings. |
| `get_training_history` / `get_training_context` | Returns training history and related context. |

## Evidence and output

Evidence is supplied by the calling host. It may include recovery measurements, vendor-computed assessments, completed workouts, scheduled sessions, goals, and constraints.

Important input rules:

- Only measured or explicitly provided values should be sent. Missing signals are reported in `signalCoverage` and may lower `confidence`.
- A workout without a training-load value is not treated as zero load and is excluded from muscle-group fatigue calculations.
- RPE may be retained as evidence but is not a term in training-load or muscle-fatigue calculations.
- Vendor-computed values such as readiness, recovery, and Body Battery are used as reported and are not recomputed by Pacevera.

A typical `decide_session` response has this shape:

```json
{
  "decision": { "type": "adjust", "intent": "reduce_today_intensity" },
  "action": {
    "from": { "focus": "Tempo Run", "durationMinutes": 50, "intensity": "high" },
    "to": { "focus": "Moderate run", "durationMinutes": 50, "intensity": "moderate" },
    "changed": ["focus", "intensity"]
  },
  "confidence": "high",
  "signalCoverage": {
    "recovery": { "usable": ["readiness"], "missing": ["sleep"] },
    "training": { "usable": ["trainingLoad"], "missing": [] }
  },
  "decisionBasis": {
    "governingRule": {
      "ruleId": "EVD-R-002",
      "measured": { "quantity": "readiness_score", "value": 48 }
    }
  }
}
```

Pacevera does not fill missing values with defaults. `signalCoverage.recovery` describes current recovery-signal availability; `signalCoverage.training` describes whether recent workouts include the training-load data required for fatigue calculations.

## Deployment status

| Deployment mode | Status | Boundary |
|---|---|---|
| Local desktop extension | Available | Runs through stdio on the user's computer. It makes no model calls, and fetches provider data only through the optional Google Health and Garmin connections the user turns on. |
| User-controlled private deployment | Planned | Intended to run in a device, private network, or VPC controlled by the user or organization. |
| Hosted remote deployment | Not available | The repository contains resource-server readiness code, but production deployment, authorization infrastructure, and hosted privacy controls are not complete. |

The desktop extension has no runtime dependencies beyond the Node.js standard library. It does not use analytics, telemetry, crash reporting, provider accounts, or provider OAuth tokens.

## Privacy

The desktop extension processes the minimum health-related Evidence supplied by the caller for the requested fitness decision. It does not retain, sell, use for model training, or use Evidence for unrelated purposes.

Unless you turn on the optional Google Health connection or Garmin Personal API sync, the extension makes no outbound network requests, and it sends Evidence to no Pacevera server in any case. On the user's own computer it does keep durable records: a bounded continuity record, a local SQLite store holding the decisions it produced, confirmed plan changes, the outcomes the user reports back, and the derived state behind them, and — when you ask your AI host to open the dashboard — a dashboard page and snapshot in `~/.pacevera`. All of them are the user's to export or delete. The AI host, operating system, imported files, and any host conversation history are outside the extension's control and are governed by their respective policies.

See [PRIVACY.md](PRIVACY.md) for the complete policy and deployment scope, and [TERMS.md](TERMS.md) for the terms of use. The canonical versions are published at <https://pacevera.com/privacy> and <https://pacevera.com/terms>.

Pacevera is not a medical device and does not provide medical advice. It is intended for general fitness and training purposes only.

## Installation

Pacevera is distributed as a Claude Desktop extension (`.mcpb`).

1. Download `pacevera.mcpb` from [Releases](../../releases).
2. Optionally verify the SHA-256 checksum published in the release notes:

   ```bash
   shasum -a 256 pacevera.mcpb
   ```

3. In Claude Desktop, open **Settings → Desktop app**, select **Manage** next to Extensions, then **Advanced settings → Install Extension…** and choose the file. In the older settings layout the same page is **Settings → Extensions**.
4. Restart Claude Desktop after installing or upgrading the extension.

The extension requires Node.js 20 or newer. Local persistence of decision traces and outcomes needs Node.js 22.5 or newer, which is where `node:sqlite` exists; on Node 20 or 21 the extension installs and runs without it, and reading your selected export folder for a decision is unaffected. The local plan decision is offered only once this machine holds a plan and its context, so a fresh installation does not list it — that depends on what is stored, not on the Node version.

Claude Desktop and other local MCP hosts are supported through the desktop extension, and Claude's own Dispatch can drive that installed extension from a phone with no remote deployment anywhere in the path — it needs the desktop app running, a Pro or Max plan, and the feature to be available to you. A Pacevera-operated hosted remote deployment is still not available for public use.

## Release status

The current public release is `v0.6.0`, using Decision Engine `1.13.0` and `base_rules@1.7.0`. It is free to download and use. It includes ten public tools, bounded local continuity, personalized single-workout generation, decision traces, package validation／dry-run／rollback, and an optional local Google Health connection. The Oura and WHOOP readers were implemented from their published API specifications and have not yet been validated against real responses; Apple Health, Garmin, Google Health, and Strava readers were developed from real export files.

## Support

- Bug reports and technical questions: [GitHub Issues](../../issues)
- Privacy requests: **support@pacevera.com**

## License

Pacevera is proprietary software. See [LICENSE](LICENSE) for the applicable terms.

## Pacevera v0.6.0

The Claude Desktop extension includes Today’s Brief, evidence-first local export reading, and plain-language tool titles. During installation, choose **Your exported health data folder** and select the parent folder containing any of these optional subfolders:

```text
Your exported health data folder/
├── export_apple_health/        # Apple Health export.xml
├── export_garmin/              # Garmin Export Your Data / DI_CONNECT
├── export_garmin_personal/     # normalized/ for what the Garmin sync wrote
├── export_strava/              # Strava activities.csv
└── export_google_health/       # raw/ for JSON you pulled yourself,
                                # normalized/ for what the connection wrote
```

Pacevera reads the selected folder locally; missing sources are reported as unavailable rather than guessed. The MCPB checksum is published in the release notes.

Pacevera includes an optional Google Health connection that runs on your computer. You
approve it in your own browser, or by scanning a QR code with a phone; the
browser route reaches Google with no Pacevera server anywhere in its path, and
the QR route passes through a Pacevera relay that handles only a short-lived
authorization handoff — never your health data, your tokens, or the PKCE secret
that completes the exchange on your machine. Whichever route you finish cancels
the other.

The connection normalizes each Google Health response in memory and
writes only the resulting evidence; the provider response itself is never
written to disk. You can also disconnect from inside the app: it revokes the
grant at Google, deletes the stored credential, deletes the evidence earlier
syncs wrote, and clears the derived records from the local store. Files you
placed in the folder yourself are reported back for you to delete, never
removed for you.

**This connection is capped at 100 accounts, and Google will warn you that it
has not verified the app.** Pacevera's Google app moved out of `Testing` into
`Production` on 2026-08-20, so an invitation is no longer required — but
Google's verification review for the health scopes is not complete. Until it is,
Google shows its own "Google hasn't verified this app" screen before you can
consent, and at most 100 accounts can authorize the connection, counted over the
app's entire lifetime. Reading your own exported folders, supplying evidence
through your AI host, and every decision tool are unaffected.

v0.6.0 keeps less about you and closes one gap. Pacevera no longer holds a name, height or weight anywhere: no decision ever read them, so they are gone from the local store — an existing store is tidied the next time it opens — and a Strava export's date of birth and weight stay in your file, unread. The folder your records are saved in, and who they belong to, are now set only by your own installation; nothing an assistant sends along with a question can change either. The extension's documentation and support links open this repository. Every decision is computed exactly as in v0.5.9.

v0.5.9 adds an optional Garmin sync through Garmin's own Personal API. Create a Personal API key for your own account on Garmin Connect Web and save it in the extension settings, where Claude Desktop keeps it on your computer — never paste it into the conversation. Pacevera then calls only `personal.connectapi.garmin.com`, reads your own activities, daily summaries, sleep and HRV, normalizes them in memory and writes only the resulting evidence to `export_garmin_personal/normalized/`. Names, profile images, GPS coordinates and place names are not read. The Garmin delete tool removes only that folder; exports you placed there yourself are untouched. Without a key, Garmin readings still come from an export you place in the folder, as before.

v0.5.9 also changes what a decision will do. Raising a session now needs a recovery reading from the last 24 hours, and when HRV is what carries it, the HRV reading itself must be that recent; a stale reading holds the session rather than raising it. One workout that reaches the folder through two sources — a Garmin activity also synced into Apple Health — is counted once. A measured HRV reading takes precedence over a vendor readiness score for the same day, and Body Battery is kept as evidence but no longer scored. Readings that carry only a date, such as Garmin's daily resting heart rate, are placed at local noon, so east of UTC+12 today's reading is read today. Sleep you only describe in conversation is treated as reference: it can no longer raise a session, and the answer asks you to add the night from your device. The local plan decision now reads your evidence folder instead of failing, asking to open the dashboard writes a local page with today's decision and a weekly-trends chart, and today's evidence preview also returns 7, 28 and 90-day personal reference values.

v0.6.0 is free to download and use, as v0.5.9 was.

v0.5.8 corrects three things a decision was saying that it could not support. When a movement you ruled out emptied a session, the replacement Pacevera offered had never been checked against the same restriction — say you cannot squat, and both barbell squats came out while a bodyweight squat went in, with the reason line mentioning only what had been removed. The replacement now passes the same check, and when nothing passes, no movement is named and the limitation says why instead of inventing one. It also tells you the movement is a fallback rather than something taken from your own plan, which it never used to mention at all. When no rule acts, the closing sentence no longer says every reading is in range: recovery below 60 on an already-easy day, or muscle-group fatigue at 40 on a day that is not hard, can cross a line without any rule acting, and the sentence now names the reading, the line, and why nothing stepped in. And on a week with no recorded training at all, it no longer says the target muscles carry no load from the last week — there was nothing to read, which is a different statement, and it is the one you now get.

Today's Brief follows the language of the conversation: ask in Traditional Chinese and the card is in Traditional Chinese, ask in English and it is in English. In v0.5.7 it followed your desktop app's display language instead, regardless of what you were speaking.

v0.5.8 also stopped implying that Garmin had a connection to repair: at that release Garmin readings only arrived as an export placed in the folder. From v0.5.9 the optional Personal API sync above is a second route, and the assistant is told which one this installation has.

v0.5.7 fixes two ways the extension could stop answering, both of them present in v0.5.6. On a day when no fresh recovery reading arrived — the day a decision is most worth asking for — asking for today's decision from a stored plan returned nothing at all, because the local store refused to record the empty readiness the engine deliberately reports. Existing stores are repaired in place, with your rows intact. And a single very large, deeply nested request could end the extension outright, which looks from the outside like the whole connector vanishing; one bad request now fails on its own and the extension keeps running. The one rule that raises a session's intensity no longer raises it while confidence is low: thin evidence may hold a session or lower it, not add load. Coverage is also reported more plainly — the vendor scores that were actually used are named, and the coverage figure is a fraction of five kinds of evidence rather than a number that could run past its own maximum. The folder reading records what it imported from Garmin without handing your folder path or file fingerprints to your AI host. (This release was described as making Today's Brief follow the language you are speaking in. That was inaccurate: in v0.5.7 the card follows your desktop app's display language. v0.5.8 is the release where it follows the conversation.)

v0.5.6 reads your HRV against your own recent nights rather than a fixed reference, and where your device already computed a readiness or HRV score, that score is used instead of scoring the raw reading a second time. The raw reading is kept and reported as superseded, named alongside what replaced it, so a decision can no longer say a reading is missing when you supplied one. Connecting Google Health, syncing it, and asking for today's decision from a stored plan no longer ask you for an account identifier: Pacevera never issued one, and being asked for it again in a later conversation is what made a working connection look expired. Where this machine holds more than one profile, Pacevera asks which one by name rather than guessing.

v0.5.2 added the source chain to every reading — where Pacevera read it, and which app wrote it, so a Garmin figure synced into Apple Health is not reported as Apple Health's own. A reading whose writer the export does not name is reported as unnamed rather than guessed. Decisions also carry a small continuity record that a later conversation can hand back; Pacevera checks whether it still describes your current state and says so when it does not. It is checked, never merged: the decision always comes from the evidence held now.
