# HabitCircle — Data Architecture & Technical Solution

---

## The Core Principle

Every user action writes to a raw event table. Every night at 11:59 PM a batch job reads all events for that day and produces **one clean row** in `daily_habit_scores`. One row per user per habit per day.

Every algorithm, every insight, every premium analytics card reads **only from `daily_habit_scores`**. Never from raw event tables directly.

```
User actions throughout the day
            ↓
    Raw event tables
    (habit_logs, nudges, chats, notifications)
            ↓
  Nightly batch job (11:59 PM)
            ↓
  daily_habit_scores — 1 row per user per habit per day
            ↓
         Analytics layer
     ↙         ↓          ↘
EWMA        Prewhitened   Nudge
baselines   Pearson       effect
(habit      (circle       analysis
 rhythm)    correlations)
            ↓
      premium_insights
      (day 21 paywall)
            ↓
     weekly_habit_summaries
```

**Why this matters:**
- Analytics is fast — reads one clean table per habit, not joining raw log tables
- Traceable — if an insight looks wrong, trace it to one column in one row
- Recomputable — raw event tables are source of truth; `daily_habit_scores` can be rebuilt entirely
- Testable — every algorithm has a single, predictable input
- Habit-scoped — one row per habit per day means insights are per-habit, not averaged across the user's whole life

---

## Layer 1 — Raw Event Tables

These capture what users actually did. Written to in real time throughout the day. Never read by analytics directly.

---

### `users`
| Column | Type | What it stores |
|--------|------|----------------|
| `user_id` | UUID | Primary key |
| `name` | TEXT | Display name |
| `phone` | TEXT | Primary identifier — OTP auth (India default) |
| `email` | TEXT | Optional — for web recovery |
| `avatar_url` | TEXT | Profile photo |
| `premium_tier` | ENUM | `free` / `me_vs_me` / `circle` / `circle_pro` |
| `premium_since` | TIMESTAMP | When they upgraded — needed for cohort analysis |
| `onboarded` | BOOL | Completed 2-screen onboarding |
| `created_at` | TIMESTAMP | Signup date |

---

### `habits`
| Column | Type | What it stores |
|--------|------|----------------|
| `habit_id` | UUID | Primary key |
| `created_by` | UUID | FK → users — who created this habit |
| `title` | TEXT | "Drink 2L Water" |
| `description` | TEXT | Optional context |
| `frequency` | ENUM | `daily` / `weekly` / `monthly` / `custom` |
| `target_count` | INT | For weekly: "3 times per week" — target_count = 3 |
| `custom_days` | JSONB | `["monday", "wednesday", "friday"]` — for custom frequency |
| `reminder_time` | TIME | User's preferred reminder for this habit |
| `icon` | TEXT | Emoji or icon slug |
| `color` | TEXT | Hex code — for habit card theming |
| `is_shared` | BOOL | True → both Tracker and Witnesses log to same record |
| `created_at` | TIMESTAMP | — |
| `archived_at` | TIMESTAMP | Null if active. Set when habit is deleted/archived. |

---

### `habit_members`
The join table between users and habits. Every relationship — Tracker, Witness, pending invite — lives here.

| Column | Type | What it stores |
|--------|------|----------------|
| `member_id` | UUID | Primary key |
| `habit_id` | UUID | FK → habits |
| `user_id` | UUID | FK → users |
| `role` | ENUM | `tracker` / `witness` |
| `can_create_habits` | BOOL | Witness permission — can they create habits for this Tracker? |
| `visibility_level` | ENUM | `full_log` / `streak_only` / `today_only` — Tracker controls per-Witness |
| `invited_by` | UUID | FK → users — who sent the invite |
| `invite_method` | ENUM | `phone` / `link` |
| `invite_token` | TEXT | Short-lived token for link-based invites |
| `invited_at` | TIMESTAMP | — |
| `accepted_at` | TIMESTAMP | Null until accepted — pending invites visible in Tracker's habit settings |
| `status` | ENUM | `pending` / `active` / `removed` |
| `removed_at` | TIMESTAMP | When Tracker removed a Witness, or Witness left |

**Note:** A Tracker has exactly one `tracker` row per habit. A Witness can be a tracker on their own habits and a witness on others — their role is per-habit, not per-account.

---

### `habit_logs`
Every check-in, by any member, on any habit.

| Column | Type | What it stores |
|--------|------|----------------|
| `log_id` | UUID | Primary key |
| `habit_id` | UUID | FK → habits |
| `user_id` | UUID | FK → users — who logged |
| `date` | DATE | Which day this log counts for |
| `logged_at` | TIMESTAMP | Exact moment they tapped the check-in |
| `note` | TEXT | Optional short note — "got it in at midnight" |
| `media_url` | TEXT | Optional photo proof — v1.1 only |
| `entry_method` | ENUM | `manual` / `widget` / `notification_tap` — how they logged |

**Why `date` and `logged_at` are separate:**
A habit completed at 11:55 PM was done "today." A habit logged at 12:05 AM was done "yesterday." `date` is the day it counts for (user-controlled, defaults to today). `logged_at` is when the tap happened. The gap tells us latency and whether users log in real time or catch up.

---

### `nudges`
One-tap accountability signals from Witness to Tracker.

| Column | Type | What it stores |
|--------|------|----------------|
| `nudge_id` | UUID | Primary key |
| `habit_id` | UUID | FK → habits |
| `from_user_id` | UUID | FK → users — who sent the nudge |
| `to_user_id` | UUID | FK → users — who received it |
| `sent_at` | TIMESTAMP | — |
| `seen_at` | TIMESTAMP | When Tracker opened the notification |
| `logged_after` | BOOL | Did the Tracker log this habit within 2 hours of the nudge? |
| `log_latency_min` | INT | Minutes between nudge and log — null if no log followed |

**Why `logged_after` and `log_latency_min` are stored here:**
These two columns are the raw material for the core premium insight: "On days your Witness nudged you, you completed this habit X% of the time." Computing this insight cleanly requires knowing the causal sequence — nudge first, then log.

---

### `habit_chats`
Per-habit message thread. Isolated from general messaging — every message belongs to one habit.

| Column | Type | What it stores |
|--------|------|----------------|
| `chat_id` | UUID | Primary key |
| `habit_id` | UUID | FK → habits |
| `user_id` | UUID | FK → users — sender |
| `message` | TEXT | Message content |
| `media_url` | TEXT | v1.1 — image/video |
| `sent_at` | TIMESTAMP | — |
| `read_by` | JSONB | `{"user_id_1": "2026-01-15T10:00:00"}` — per-reader read receipts |

---

### `notification_events`
Feeds Thompson Sampling — tracks which hour slot produces a response within the response window.

| Column | Type | What it stores |
|--------|------|----------------|
| `event_id` | UUID | Primary key |
| `user_id` | UUID | FK → users |
| `habit_id` | UUID | FK → habits — nullable (some notifications are global) |
| `notification_type` | ENUM | `habit_reminder` / `nudge_received` / `weekly_summary` / `premium_reveal` |
| `sent_at` | TIMESTAMP | — |
| `hour_slot` | INT | 0–23 — which hour it was sent |
| `delivered` | BOOL | FCM/APNs delivery confirmed |
| `responded` | BOOL | User opened or acted within response window |
| `response_window_min` | INT | 30 for reminders, 10 for nudges |
| `alpha` | INT | Running count of responses for this slot — updated after each event |
| `beta` | INT | Running count of non-responses for this slot |

**Note on `delivered`:** This column is why HabitShare's notification bug cannot happen in HabitCircle. Every notification has a delivery receipt. If `delivered` is false 5 minutes after `sent_at`, the job retries. The retry is logged as a new row with the same `habit_id` and `notification_type`.

---

### `user_notification_slots`
Per-user, per-habit Thompson Sampling state. 24 rows per user per habit at signup, seeded uniformly.

| Column | Type | What it stores |
|--------|------|----------------|
| `slot_id` | UUID | Primary key |
| `user_id` | UUID | FK → users |
| `habit_id` | UUID | FK → habits — null for global slots |
| `hour_slot` | INT | 0–23 |
| `alpha` | INT | Starts at 1. Increments on response. |
| `beta` | INT | Starts at 1. Increments on non-response. |
| `last_updated` | TIMESTAMP | When last notification was sent in this slot |

---

## Layer 2 — The Nightly Batch Job

Runs at **11:59 PM per user timezone**. For each user, for each of their active habits, reads the day's raw events and produces one row in `daily_habit_scores`.

```
For each user:
  For each active habit:

  1. Read habit_logs for today
       → logged (bool), log_time, entry_method

  2. Read nudges received today
       → was_nudged (bool), nudge_count, log_latency_min

  3. If shared habit:
       → Read habit_logs from all co-trackers
       → partner_logged (bool), shared_streak_increment

  4. Compute streak_day:
       If logged today → streak_day = yesterday's streak_day + 1
       If not logged  → streak_day = 0

  5. Compute shared_streak_day (shared habits only):
       If both logged → shared_streak_day = yesterday + 1
       If either missed → shared_streak_day = 0

  6. Write one row → daily_habit_scores

  7. Update EWMA baseline in habit_baselines:
       new_baseline = (0.1 × today_logged) + (0.9 × yesterday_baseline)

  8. If premium user AND days_with_data ≥ 28:
       → Run Ljung-Box autocorrelation test on this habit's series
       → If autocorrelation detected: AR(1) prewhiten, use residuals
       → Run Pearson on prewhitened series
       → Apply Bonferroni correction
       → If all 4 guards pass: write to circle_correlations

  9. If days_with_data == 21:
       → Generate premium_insights preview row
       → Set blurred = true
       → Queue paywall push notification for next morning
```

---

## Layer 3 — `daily_habit_scores` (The Central Table)

**One row per user per habit per day. Every algorithm reads from here.**

| Column | Type | Source | What it represents |
|--------|------|--------|-------------------|
| `user_id` | UUID | — | — |
| `habit_id` | UUID | — | — |
| `date` | DATE | — | — |
| `logged` | BOOL | habit_logs | Did the user check in today |
| `log_time` | TIMESTAMP | habit_logs | When they logged — null if not logged |
| `log_latency_min` | INT | computed | Minutes between reminder sent and log. Null if not logged or no reminder. |
| `entry_method` | ENUM | habit_logs | `manual` / `widget` / `notification_tap` |
| `was_nudged` | BOOL | nudges | Did a Witness send a nudge today |
| `nudge_count` | INT | nudges | How many nudges received today |
| `nudge_log_latency_min` | INT | nudges | Minutes from first nudge to log. Null if not nudged or no log followed. |
| `nudge_converted` | BOOL | computed | was_nudged AND logged AND nudge_log_latency_min ≤ 120 |
| `streak_day` | INT | computed | Consecutive days logged including today |
| `is_shared` | BOOL | habits | True if this is a shared habit |
| `partner_logged` | BOOL | habit_logs | True if co-tracker also logged today. Null for non-shared. |
| `shared_streak_day` | INT | computed | Consecutive days both logged. Null for non-shared. |
| `witness_active_today` | BOOL | habit_members + nudges | Did any Witness open the app or send a nudge today |
| `day_of_week` | INT | computed | 0=Monday … 6=Sunday — for weekly pattern detection |

---

## Layer 4 — Analytics Tables

Computed from `daily_habit_scores`. Never written to in real time — updated nightly or when thresholds are met.

---

### `habit_baselines`
One row per user per habit. Updated every night.

| Column | Type | What it stores |
|--------|------|----------------|
| `user_id` | UUID | — |
| `habit_id` | UUID | — |
| `ewma_completion` | FLOAT | Rolling baseline for daily completion rate (0–1) |
| `ewma_log_latency` | FLOAT | Rolling baseline for how quickly they usually log |
| `days_with_data` | INT | Total days this habit has been active — gates analytics |
| `updated_at` | DATE | — |

**Formula:** `S_t = (0.1 × today_logged) + (0.9 × yesterday_baseline)`

`today_logged` is 1 if logged, 0 if not. The EWMA smooths over time — a missed day moves the baseline down slightly, not dramatically. After 19 effective days the baseline reflects the user's real rhythm for that habit.

---

### `circle_correlations`
Prewhitened Pearson results. Written when all 4 guards pass. One row per variable pair per habit per user.

| Column | Type | What it stores |
|--------|------|----------------|
| `correlation_id` | UUID | — |
| `user_id` | UUID | — |
| `habit_id` | UUID | — |
| `feature_x` | TEXT | e.g. `was_nudged`, `witness_active_today`, `partner_logged` |
| `feature_y` | TEXT | e.g. `logged`, `log_latency_min` |
| `n_points` | INT | Days of joint data used |
| `ljung_box_stat` | FLOAT | Q-statistic from autocorrelation test |
| `autocorr_detected` | BOOL | True if Ljung-Box p < 0.05 |
| `ar1_applied` | BOOL | True if prewhitening was used |
| `r_raw` | FLOAT | Pearson on raw series — stored for audit |
| `r_prewhitened` | FLOAT | Pearson on AR(1) residuals — the number used for insights |
| `p_value` | FLOAT | Significance after prewhitening |
| `bonferroni_threshold` | FLOAT | Adjusted threshold used for this user (0.05 / pairs_tested) |
| `passes_all_guards` | BOOL | N≥28, \|r\|>0.4, p<bonferroni_threshold, Ljung-Box test run |
| `computed_at` | DATE | — |

**Pairs tested per habit:**
- `was_nudged` → `logged` (did nudges improve completion?)
- `witness_active_today` → `logged` (does Witness activity affect Tracker?)
- `partner_logged[day-1]` → `logged[day]` (does seeing partner log yesterday motivate today?)
- `log_latency_min` → `streak_continuation` (do faster logs predict longer streaks?)

Maximum 4 pairs. Bonferroni threshold = 0.05 / 4 = **0.0125**.

---

### `premium_insights`
Pre-computed insight cards served at day 21. Stored blurred until user upgrades.

| Column | Type | What it stores |
|--------|------|----------------|
| `insight_id` | UUID | — |
| `user_id` | UUID | — |
| `habit_id` | UUID | — |
| `insight_type` | ENUM | `me_vs_me` / `nudge_effect` / `circle_correlation` / `shared_sync` |
| `headline` | TEXT | The unblurred first line shown in the paywall preview |
| `full_text` | TEXT | The complete insight — shown after upgrade |
| `supporting_stat` | FLOAT | The r_value, percentage, or delta used in the headline |
| `n_days` | INT | Sample size shown to user ("Based on 28 days of data") |
| `confidence` | ENUM | `low` / `moderate` / `high` — shown as a tag on the card |
| `blurred` | BOOL | True until user upgrades. The headline is always visible. |
| `generated_at` | TIMESTAMP | When the batch job created this |
| `shown_at` | TIMESTAMP | When the paywall card was first displayed |
| `converted_at` | TIMESTAMP | When user upgraded to see the full insight. Null if not converted. |

---

## Layer 5 — Report Tables

### `weekly_habit_summaries`
Generated every Monday (or user's chosen day) if the user has ≥ 4 logged days in the past week.

| Column | Type | What it stores |
|--------|------|----------------|
| `summary_id` | UUID | — |
| `user_id` | UUID | — |
| `week_start` | DATE | Monday of the reported week |
| `week_end` | DATE | Sunday |
| `days_logged_this_week` | INT | Across all habits — total check-ins |
| `habits_active` | INT | Active habits this week |
| `completion_rate_this_week` | FLOAT | (logged days / possible days) across all habits |
| `completion_rate_ewma_baseline` | FLOAT | EWMA baseline at week start — the "rhythm" comparison |
| `delta_pct` | FLOAT | completion_rate_this_week − ewma_baseline |
| `direction` | ENUM | `improving` / `stable` / `declining` (>10% = directional) |
| `best_habit_name` | TEXT | Habit with highest completion rate this week |
| `best_streak_habit` | TEXT | Habit with longest current streak |
| `witness_nudge_count` | INT | Total nudges received this week |
| `nudge_conversion_rate` | FLOAT | nudge_converted / total_nudges — did nudges lead to logs? |
| `shared_streak_best` | INT | Best current shared streak across all shared habits |
| `witness_active_days` | INT | Days at least one Witness was active in the user's circle |
| `premium_preview_triggered` | BOOL | Was the day-21 paywall shown this week |

---

## Data Flow Summary

| What user does | Where it lands | When it's computed |
|----------------|----------------|-------------------|
| Creates a habit | `habits` | Immediately |
| Invites a Witness | `habit_members` (status: pending) | Immediately |
| Witness accepts invite | `habit_members` (status: active) | Immediately |
| Logs a habit | `habit_logs` | Immediately |
| Sends a nudge | `nudges` | Immediately |
| Sends a chat message | `habit_chats` | Immediately |
| Taps notification | `notification_events` (responded=true) | Immediately |
| — | `daily_habit_scores` | 11:59 PM nightly batch |
| — | `habit_baselines` (EWMA) | Nightly, after daily_habit_scores |
| — | `circle_correlations` | Nightly, if N ≥ 28 and premium |
| — | `premium_insights` (blurred) | Day 21 batch — paywall trigger |
| — | `user_notification_slots` (α/β update) | After each notification event |
| — | `weekly_habit_summaries` | Monday morning, if ≥ 4 logged days |

---

## Analytics Deep Dive — EWMA (Habit Rhythm Baseline)

**What it is:**
The same exponentially weighted moving average used in Me vs Me, applied per habit per user. Every night after logging, the completion baseline for each habit is updated.

**The formula:**
```
S_t = (0.1 × today_logged) + (0.9 × yesterday_baseline)

where today_logged = 1 if logged, 0 if not
```

**What it produces:**
A number between 0 and 1 representing the user's rolling completion rate for this specific habit. A user who logs their water habit 6 out of 7 days consistently will have a baseline around 0.85. A user who logs 3 out of 7 will settle around 0.40.

**Why this matters for Me vs Me analytics:**
The EWMA baseline IS "you at your rhythm." The Me vs Me premium insight compares this week's completion rate against it:

```
This week's completion rate: 0.86
Your rhythm (EWMA baseline): 0.61
Delta: +0.25 → "You are 25 points above your own baseline this week."
```

No comparison to other users. No comparison to an idealised "best month" (the CTO memo's correct criticism of the original Me vs Me concept). Just this week vs your real rolling average.

**λ = 0.1 gives 19 days of effective memory.** Recent enough to reflect genuine change in behaviour. Stable enough that one missed day does not collapse the baseline. This is the same choice as Me vs Me — the reasoning is identical.

**Concrete example:**
Your water habit baseline is 0.72 (your rhythm). You have a perfect week — logged all 7 days. Each night:

```
Day 1: (0.1 × 1) + (0.9 × 0.72) = 0.748
Day 2: (0.1 × 1) + (0.9 × 0.748) = 0.773
...
Day 7: baseline reaches ~0.80
```

One great week moves the needle from 0.72 to 0.80. Not dramatically. Sustained improvement over 3 weeks will move it to 0.90. That is the correct behaviour.

---

## Analytics Deep Dive — Prewhitened Pearson (Circle Correlations)

**What problem it solves:**
Standard Pearson correlation on daily time series is statistically invalid. Human behaviour on consecutive days is autocorrelated — if you logged your habit today, you are more likely to log it tomorrow (habit momentum). Standard Pearson assumes independent observations. Autocorrelation inflates apparent significance, producing correlations that look strong but are partly an artifact of the serial structure of the data.

The CTO memo flagged this as the core statistical defect in the original Me vs Me design. HabitCircle fixes it before any insight is shown.

---

**Step 1 — Ljung-Box test for autocorrelation:**

Before running Pearson on any variable pair, the batch job tests both series for serial autocorrelation using the Ljung-Box Q-statistic at lag 7 (one week).

```
H₀: No autocorrelation up to lag 7
H₁: Autocorrelation present

If p < 0.05 → autocorrelation detected → prewhitening required
If p ≥ 0.05 → series is approximately independent → Pearson on raw data is valid
```

`autocorr_detected` and `ar1_applied` are stored in `circle_correlations` for every computed pair. If autocorrelation is detected but was not corrected, the insight is blocked entirely.

---

**Step 2 — AR(1) prewhitening:**

If autocorrelation is detected in either series, the batch job fits a first-order autoregressive model AR(1) to each series and uses the **residuals** for Pearson — not the raw values.

```
For series X (e.g., was_nudged over 30 days):
  1. Fit AR(1): X_t = φ × X_{t-1} + ε_t
  2. Estimate φ (autoregressive coefficient) via OLS
  3. Compute residuals: ε_t = X_t − (φ × X_{t-1})
  4. Use residuals ε_t for Pearson instead of raw X_t

Repeat for series Y (e.g., logged over 30 days).
Then run Pearson on (residuals_X, residuals_Y).
```

The residuals represent the "unexplained" part of each day — what cannot be predicted from yesterday's value alone. Correlating residuals isolates the genuine day-to-day relationship between the two variables, stripped of the momentum structure both series share.

---

**Step 3 — Bonferroni correction:**

HabitCircle tests at most 4 variable pairs per habit. Testing 4 pairs at p < 0.05 means a ~18% chance of at least one false positive by random chance (1 − 0.95⁴). Bonferroni correction divides the threshold by the number of tests:

```
Adjusted threshold = 0.05 / 4 = 0.0125

An insight is only surfaced if p < 0.0125 — not p < 0.05.
```

This is stored as `bonferroni_threshold` in `circle_correlations`. If the number of pairs tested ever changes, the threshold adjusts automatically.

---

**The 4 guards — all must pass:**

| Guard | Threshold | Why |
|-------|-----------|-----|
| Minimum data | N ≥ 28 days joint data | 21 days is autocorrelated thin. 28 provides enough residual degrees of freedom after AR(1) correction. |
| Effect size | \|r\| > 0.4 (prewhitened) | Weak correlations are not shown. 0.4 is reduced from 0.5 to account for the slight power reduction from prewhitening. |
| Significance | p < 0.0125 (Bonferroni) | Adjusted for 4 pairs per habit. Far stricter than raw p < 0.05. |
| Autocorrelation test | Ljung-Box run on both series | The test must have been run. If it was skipped (data error), the insight is blocked. |

All four must pass. If any fails, nothing is shown. No exceptions. The insight card never displays a statistic the methodology cannot support.

---

**What the insight looks like — variable pairs and their sentences:**

`was_nudged` → `logged`:
> *"On days your mom sent you a nudge, you completed Water at 73%. On other days: 38%. Based on 34 days of data. (Moderate confidence)"*

`witness_active_today` → `logged`:
> *"On weeks your sister was also active in the app, your habit completion rate averaged 81%. Weeks she wasn't active: 54%."*

`partner_logged[day-1]` → `logged[day]`:
> *"When your partner logged Exercise yesterday, you logged it today 78% of the time. When they missed: 41%."*

**Language rules — applied to every insight string:**
- Never: "causes," "leads to," "proves," "because"
- Always: "tends to be higher when," "in your data, X and Y move together," "on days when"
- Every card shows: sample size ("Based on 34 days"), confidence level, and a one-line disclaimer: *"This is a pattern in your data, not a causal proof."*

---

## Analytics Deep Dive — Thompson Sampling (Notification Timing)

**What problem it solves:**
A notification sent when the user is asleep, in a meeting, or already distracted gets ignored. Ignored notifications train users to dismiss them. HabitShare's broken notifications were the top complaint in every low-rating review — not because the notifications didn't send, but because they sent at the wrong time and were then ignored, breaking the habit loop.

Thompson Sampling learns each user's receptive window from their own response behaviour. No survey. No "pick your reminder time" screen. The app figures it out.

---

**The mechanism:**

For every user, for every habit, for every hour of the day (24 slots), the app maintains two counters in `user_notification_slots`:

```
α (alpha) = number of times a notification in this slot produced a response
β (beta)  = number of times it was ignored
```

When it is time to send a habit reminder, the job:
1. Draws a random sample from Beta(α + 1, β + 1) for each hour slot
2. Sends at whichever slot drew the highest sample
3. Observes: did the user log the habit within the response window?
4. Updates: if yes → α += 1. If no → β += 1.

```
If user logged within 30 min of reminder → α += 1
If user did not log within 30 min       → β += 1
```

---

**Concrete walkthrough:**

| Hour slot | α | β | Typical sample range |
|-----------|---|---|----------------------|
| 7 AM | 1 | 9 | 0.05 – 0.25 — rarely drawn, this user doesn't respond mornings |
| 9 AM | 14 | 3 | 0.70 – 0.88 — drawn most days |
| 1 PM | 2 | 2 | 0.20 – 0.70 — uncertain, occasionally tried |
| 9 PM | 11 | 4 | 0.58 – 0.82 — second best slot |

The algorithm has learned this user responds best at 9 AM and 9 PM. It sends there most of the time, but occasionally tests 1 PM in case something has changed (the user started working from home, shifted their schedule).

---

**Separate slots per habit:**
A user might want their water habit reminder at 8 AM but their exercise reminder at 6 AM. Thompson Sampling runs independently per habit — `user_notification_slots` has 24 rows per user per habit. Habit-level personalisation, not just user-level.

---

**What counts as a response:**

| Notification type | Positive signal | Response window |
|-------------------|-----------------|-----------------|
| Habit reminder | User logs the habit | 30 minutes |
| Nudge received | User opens the notification | 10 minutes |
| Weekly summary | User opens the summary screen | 2 hours |

A notification opened 3 hours later is treated as ignored — the goal is to find the moment of peak receptivity, not track eventual opens.

---

**MVP 1 note:**
Thompson Sampling requires existing open/ignore data to learn from. On day 1 there is none. MVP 1 uses a fixed reminder time (user's chosen time from onboarding, or 9 PM default). `user_notification_slots` is seeded uniformly (α=1, β=1 for all 24 slots) from day 1. Thompson Sampling activates after 14 days of data once the α/β distributions have meaningful signal. The transition is invisible to the user.

---

## Analytics Deep Dive — Nudge Effect Analysis

**What it is:**
A simplified, non-correlation analysis that surfaces the core circle insight without requiring 28 days of joint data. Available as a preview even before Pearson clears all guards.

**The calculation:**
```
nudge_days    = all days in window where was_nudged = true
no_nudge_days = all days in window where was_nudged = false

completion_on_nudge_days    = COUNT(logged=true, nudge_days)    / COUNT(nudge_days)
completion_on_no_nudge_days = COUNT(logged=true, no_nudge_days) / COUNT(no_nudge_days)

nudge_lift = completion_on_nudge_days − completion_on_no_nudge_days
```

**Guards for the nudge effect (simpler than full Pearson):**
- Minimum 7 nudge days AND 7 non-nudge days (cannot compare two groups of one)
- nudge_lift > 0.15 (Witness nudges must produce at least 15 percentage point lift — below this is noise)
- Total days ≥ 14

**Why this is statistically defensible (simpler than Pearson):**
This is a proportion test between two groups, not a time-series correlation. Autocorrelation is less of a concern because we are comparing group means, not sequential observations. A chi-squared test of independence is sufficient. No prewhitening required.

**The insight this produces:**

Day 18 (before full Pearson is ready):
> *"When your mom nudges you, you complete this habit 2× more often."*

Day 28+ (Pearson clears all guards):
> *"On days your mom sent you a nudge, you completed Water at 73%. On other days: 38%. Based on 34 days of data. (r = 0.51, p = 0.008)"*

The nudge effect analysis serves as the **day-21 paywall hook**. It is real arithmetic, not an estimate. The user sees a number that describes their actual circle behaviour. The full Pearson insight, which requires more data and passes stricter guards, is what they unlock with premium.

---

## The Full Analytics Stack — Summary

| Algorithm | Type | When it runs | Requires premium | What it produces |
|-----------|------|-------------|-----------------|-----------------|
| EWMA | Statistical | Every night from day 1 | No | Habit rhythm baseline per user per habit |
| Nudge Effect | Proportion test | From day 14 | No (preview) | Nudge lift — "your circle helps this much" |
| Prewhitened Pearson | Statistical (corrected) | Nightly from day 28 | Yes | Circle correlation insights — validated variable pairs |
| Thompson Sampling | Bayesian ML | Every notification send | No | Personalised reminder time per user per habit |
| Me vs Me Delta | Arithmetic | Weekly | Yes (Me vs Me tier) | This week vs EWMA rhythm — above / on / below |

All algorithms read from `daily_habit_scores`. No algorithm touches raw event tables. If `daily_habit_scores` is rebuilt, every insight can be regenerated from scratch.

---

## Notification Reliability Architecture

HabitShare's top critical reviews all cite the same bug: notifications stop working. Both Android and iOS. No developer response. This is HabitCircle's most important technical commitment.

**The problem with client-side scheduling:**
Most habit apps schedule notifications on the device. If the user clears app data, reinstalls, or the OS kills the background process, scheduled notifications vanish silently. The app has no way to know they were never delivered.

**HabitCircle's approach — server-side scheduling:**

```
1. User sets reminder time for a habit
       ↓
2. Server stores: user_id, habit_id, reminder_time, timezone
       ↓
3. Every day at reminder_time, server sends a push via FCM (Android) / APNs (iOS)
       ↓
4. FCM/APNs returns a delivery receipt
       ↓
5. Batch job checks: any reminders sent > 5 min ago with no delivery receipt?
       ↓
6. If yes → retry once immediately, then flag in notification_events (delivered=false)
```

The scheduled reminders live on the server, not the device. App reinstall, data clear, OS upgrade — none of these affect delivery. The only failure mode is the user turning off all notifications at OS level, which no app can override.

**One-tap log from notification shade (Android):**
The nudge notification and habit reminder both include an action button that logs the habit without opening the app. This is implemented via a notification action intent that calls the log endpoint directly. The `entry_method` column in `habit_logs` stores `notification_tap` for these logs — used to measure how much of the logging happens without any app interaction.
