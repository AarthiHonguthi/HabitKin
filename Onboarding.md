# Me vs Me — Onboarding, Goals & Night Flow


## Onboarding Goal

One job: understand who this person is and what they want to improve — in their own words.
By the end, before they log a single day, they see one sentence that feels accurate about themselves.

---

## Screen 1 — Who Are You?

**Question:** What best describes you?

| Option | Follow-up |
|--------|-----------|
| Student | What are you studying toward? → Competitive exam (JEE/NEET/UPSC/CA) / Degree / Skill building / School (9–12) / Other |
| Working | What describes your work? → Job (salaried) / Freelance/gig / Running a business / Teaching/coaching / Other |
| Home Maker | What does your day mostly involve? → Managing household / Raising children / Caregiving / Running home + side work / Other |

This single answer changes all language in the app — a medical student sees "revision block", an SDE sees "deep work session", a homemaker sees "task window."

**Data collected:**
| Field | Type | Value |
|-------|------|-------|
| `user_type` | ENUM | `student` / `working` / `homemaker` |
| `user_subtype` | ENUM | e.g. `competitive_exam`, `salaried`, `freelance`, `caregiving` |

---

## Screen 2 — Your Best Period

**The problem with a fully open question:**
If asked "what was different about that time?" a user might type "the day I spent with my dad" or "my relationship was going well." These are real and valid — but they give Haiku nothing to extract for Health or Career, which are the only two domains in MVP1. A fully open question produces unusable signal.

**The fix — guided but not a form:**
The question is framed to anchor reflection around energy and output, which naturally maps to Health and Career without naming them:

> "Think about a time you felt sharp, consistent, and like yourself — even just two weeks. What were your days actually like? How did you wake up, how much did you get done, how did you feel physically?"

The words *sharp*, *consistent*, *wake up*, *get done*, *feel physically* are deliberate. They pull the answer toward sleep, focus, energy, and output — the exact signals we need — without feeling like a questionnaire.

**What if the answer is still off-signal?**
Haiku evaluates the response for health and career signals. If confidence is below threshold (e.g. the user wrote only about a relationship or an event), a single gentle follow-up appears — not a new screen, just one line beneath the text field:

> "Sounds like that time mattered. How were you sleeping and how focused were you during that period?"

This follow-up fires at most once. The user can skip it. Their original text is always stored verbatim regardless.

**What Haiku extracts (scoped to MVP1 domains only):**

| Signal type | What we look for | Maps to |
|-------------|-----------------|---------|
| Sleep quality | "sleeping well", "woke early", "rested" | Health domain |
| Energy level | "felt sharp", "wasn't tired", "had energy" | Health domain |
| Physical activity | "was going to the gym", "walking daily" | Health domain |
| Focus / output | "was getting a lot done", "studying consistently" | Career domain |
| Consistency | "had a routine", "planned my days" | Career domain |
| Screen / phone | "wasn't on my phone much", "less distracted" | Both |

Signals outside these (family, relationships, emotions) are stored in `extracted_conditions` as context but do not affect domain scoring. They are never discarded — they may matter in future domains.

**Data collected:**
| Field | Type | Value |
|-------|------|-------|
| `best_period_text` | TEXT | Raw free text, stored verbatim, never modified |
| `extracted_health_signals` | JSONB | `["early wake", "gym", "felt rested"]` |
| `extracted_career_signals` | JSONB | `["consistent study", "planned days"]` |
| `extracted_other` | JSONB | Off-domain signals stored for future use |
| `extraction_confidence` | FLOAT | 0–1. Below 0.4 triggers the follow-up question |
| `emotional_tone` | TEXT | `calm` / `proud` / `energetic` / `in_control` |

---

## Screen 3 — Calibration

Two passes. Same questions. Different frame.
First pass: **your ideal day** — used only to establish the ceiling (shown in weekly/monthly reports as aspiration, never as daily pressure).
Second pass: **your average day right now** — the honest current state.

Both values together are averaged to seed the EWMA baseline for days 1–6. The EWMA takes over from real logged data from day 7 onward. Neither value is used as the ongoing daily benchmark — that job belongs entirely to the EWMA.

Users cannot accurately self-report their own average. So we don't rely on it. We use it only as a temporary seed until real data exists.

### Dynamic Slider Logic

Total hours in a day = **24**.
Sliders are **interdependent** — as one fills up, the available range for others shrinks.

**Rule:** `sleep_max = 24 − primary_activity_hours`

> Example: If user says they study 18 hours on their best day → sleep slider range becomes 1–6h (not 1–10h). 24 − 18 = 6. The app enforces this silently, no explanation needed.

Sliders always range **1–24** but the max on each adjusts live as the user sets other values. The constraint is additive — all declared hour fields cannot exceed 24.

---

### Part A — Your Ideal Day (Ceiling Only)

#### Students
| Question | Input | Constraint |
|----------|-------|------------|
| On your best study day, how many hours were you truly focused? | Slider 1–24h | Primary anchor |
| On that same day, how many hours did you sleep? | Slider 1–(24 − study_h) | Shrinks as study_h rises |
| How many steps did you get on a good day? | Number input | — |
| Does that feel like your best, could be better, or just right? | My best / Could be better / Just right | — |
| What gets in your way most? | Phone / Low energy / Inconsistency / No motivation / Can't focus | Single tap |

#### Working Professionals
| Question | Input | Constraint |
|----------|-------|------------|
| On your most productive day, how many hours of deep focused work? | Slider 1–24h | Primary anchor |
| Sleep on that same day? | Slider 1–(24 − work_h) | Shrinks as work_h rises |
| How many steps did you get on a good day? | Number input | — |
| Does that feel like your best, could be better, or just right? | My best / Could be better / Just right | — |
| Biggest blocker? | Phone / Low energy / Inconsistency / No motivation / Can't focus | Single tap |

#### Homemakers
| Question | Input | Constraint |
|----------|-------|------------|
| On your best day, how many hours did you spend on planned tasks (house, kids, work)? | Slider 1–24h | Primary anchor |
| Sleep on that day? | Slider 1–(24 − task_h) | Shrinks as task_h rises |
| How many steps did you get on a good day? | Number input | — |
| Does that feel like your best, could be better, or just right? | My best / Could be better / Just right | — |
| What gets in your way most? | Phone / Low energy / Inconsistency / No motivation / No time for self | Single tap |

**Why steps as a number, not Low/Moderate/High:**
"Moderate" means different things to different people. A number gives us a real anchor to track against. The follow-up question ("My best / Could be better / Just right") gives us the subjective layer — so we know if 4,000 steps is their ceiling or just a lazy day. Both together are more useful than either alone.

---

### Part B — Your Average Day Right Now

Same questions, same slider logic. Different frame:
> "Now, honestly — what does your average day actually look like right now?"

All slider constraints apply identically. The two values are stored separately — never merged. Together they compute the EWMA seed:

`ewma_seed = (ideal_value + current_value) / 2`

This seed is used for days 1–6 only, clearly labelled as an estimate in the UI. From day 7, real logged data starts replacing it. By day 21, the seed has no weight left.

**Data collected (both passes):**
| Field | Type | Purpose |
|-------|------|---------|
| `best_focus_h` | FLOAT | Ceiling — shown in weekly report as aspiration, never daily pressure |
| `current_focus_h` | FLOAT | Half of EWMA seed for focus |
| `best_sleep_h` | FLOAT | Ceiling for health domain |
| `current_sleep_h` | FLOAT | Half of EWMA seed for sleep |
| `best_activity_level` | ENUM | `low` / `moderate` / `high` — ceiling |
| `current_activity_level` | ENUM | Half of EWMA seed for activity |
| `ewma_seed_focus` | FLOAT | `(best_focus_h + current_focus_h) / 2` — days 1–6 baseline |
| `ewma_seed_sleep` | FLOAT | `(best_sleep_h + current_sleep_h) / 2` — days 1–6 baseline |
| `primary_blocker` | ENUM | Routes first week of notifications |

The blocker field directly personalises week 1 nudges — phone → screen nudges, low energy → sleep nudges.

---

## Screen 4 — Domain Weights

Two domains in V1: **Health** and **Career/Learning**

User sees two sliders that must always sum to 100.

**Defaults by user type:**
| User type | Health default | Career default |
|-----------|---------------|----------------|
| Student | 35% | 65% |
| Working | 40% | 60% |
| Home Maker | 50% | 50% |

The Me Score is literally built from these weights. Their definition of success — not a formula imposed on them.

### What "Career/Learning" means for Homemakers

Homemakers may have no traditional career or study goal. For them, this domain is relabelled **Personal Growth** and includes structured options they can choose from:

| Level | Examples |
|-------|---------|
| Pick up a hobby | Painting, gardening, cooking new cuisines, music |
| Learn a skill | Online course, language, coding basics, finance |
| Teach something | Teaching kids, coaching a neighbour, sharing a skill |
| Build something | Start a small side project, document family history, start a blog |

These are shown as suggestions, not requirements. A homemaker can pick one, pick none, or write their own. The goal is to give them a growth anchor — something to track in the Career/Personal Growth domain — because "managing the house" is already their baseline, not their growth edge.

Goals under this domain follow the same weekly/monthly goal system as any other user.

**Data collected:**
| Field | Type | Value |
|-------|------|-------|
| `health_weight` | INT | 0–100 |
| `career_weight` | INT | 0–100 (= 100 − health_weight) |
| `career_domain_label` | TEXT | `career` / `learning` / `personal_growth` — set by user_type |

---

## End of Onboarding — The Hook

Immediately after Screen 4, the app shows a synthetic Me Score built from the EWMA seed (average of ideal and current sliders):

> "Based on what you told us — you're focusing around 3–4 hours a day and sleeping around 6 hours. Your rhythm estimate puts you at about **61%**. Log a few days and we'll replace this with your real number."

- Framed around rhythm, not best self — so it doesn't feel like a failing grade on day 1
- Clearly labelled as an estimate
- Replaced by real EWMA data within 7 days of logging
- The number is the hook — they want to find out if it's accurate

---

## Benchmark System — How Daily Comparison Actually Works

This is the most important architectural decision in the product. The benchmark is not the personal best. The benchmark is the personal rhythm — built from real logged data.

### The problem with personal best as benchmark
Comparing every day to your best day is demotivating. Your best day may have been during a holiday, a particularly motivated week, or a period that's not reproducible. Using it as a daily yardstick guarantees that most days feel like underperformance.

### The problem with self-reported average
Users cannot accurately self-report their own average. They consistently over or underestimate. So we don't ask them to. We build it from real data.

### The solution: EWMA as personal rhythm

`S_t = (0.1 × today_score) + (0.9 × yesterday_baseline)`

The EWMA is updated every night from real logged data. It has a 19-day effective memory window — recent enough to reflect genuine change, stable enough not to panic on one bad day. This IS the user's personal average, computed honestly from behaviour, not self-report.

**Daily comparison is always against the EWMA. Never against personal best.**

> "You're above your rhythm today" / "On rhythm" / "Below your rhythm"

The personal best is shown in the weekly and monthly report only — as the ceiling, as aspiration, as context. Never as the daily benchmark.

### Three-phase baseline evolution

| Phase | Days | What the baseline is |
|-------|------|---------------------|
| Cold start | 1–6 | EWMA seed from onboarding sliders — labelled as estimate |
| Transition | 7–20 | EWMA seed weight fades, real logged data takes over |
| Real baseline | 21+ | Fully data-driven EWMA — seed has zero weight |

### Recovery days are excluded

Days where the user stamps **Recovery** are excluded from EWMA calculation entirely. A rest day that is intentional should not pull the rhythm down. The average should only reflect days the user actually tried to perform.

### Weekly report — how both benchmarks appear together

> "Your rhythm this week: 4.2h focus. Your peak ever: 6.8h. You have 2.6h of ceiling left."

This framing shows where they are (rhythm) and how much room exists (ceiling) — without making the ceiling feel like a daily obligation.

---

## The 7-Day Character Unlock

The cold-start problem is not just technical — it's emotional. Days 1–7 are when most users drop off because they have no data, no insights, and no reason to come back. The character unlock solves this.

### How it works

At the end of onboarding, the user chooses a character — someone they care about:

> "Who do you want to unlock?"
> **Dad / Mom / Best Friend / Partner / Sibling / Mentor**

The character appears immediately — but as a **full silhouette**. Locked. Dark. No face, no details.

A single line beneath it:
> "Log for 7 days to unlock them."

That's the entire mechanic. No explanation needed.

### The 7-day reveal

Each day the user logs (completes their evening check-in), the character becomes a little more visible:

| Day | What unlocks |
|-----|-------------|
| 1 | Silhouette only |
| 2 | Outline appears |
| 3 | Color fills in |
| 4 | Face becomes visible |
| 5 | Expression appears |
| 6 | Almost fully revealed |
| 7 | Full unlock — animation plays |

On day 7, the character is fully revealed with a short message in their voice. Not generic. Written for that character type:

- **Dad:** *"You showed up every day for a week. I don't say this enough — I'm proud of you."*
- **Best Friend:** *"Okay okay I see you. Seven days straight. We're celebrating this."*
- **Partner:** *"I've been watching. You didn't give up. That means everything."*
- **Mom:** *"Seven days. I knew you could. Now don't stop."*

### What the character becomes after day 7

The character is not a gimmick that disappears. After unlock they become the tone layer for the app going forward — they narrate the weekly report, send the nudge notifications, and frame insights in their voice.

This is the V2 AI Companion feature — the 7-day unlock is the bridge that earns it. By the time the companion voice activates, the user has already formed an emotional attachment to the character.

### Why this works

- It creates a concrete 7-day commitment without framing it as a streak
- The emotional pull is attachment, not gamification — there are no points, no badges
- Missing a day doesn't break a streak — it just means the character stays locked one more day. Come back tomorrow, it continues.
- The character is chosen by the user — they pick who matters to them, which makes the unlock personally meaningful

**Data collected:**
| Field | Type | Value |
|-------|------|-------|
| `character_type` | ENUM | `dad` / `mom` / `best_friend` / `partner` / `sibling` / `mentor` |
| `character_unlock_day` | INT | Which day (1–7) the character is currently at |
| `character_unlocked` | BOOL | True after 7 logged days |
| `unlock_completed_at` | TIMESTAMP | When full unlock happened |

---

---

# Goals

Users can have any number of goals. A goal has a type — daily, weekly, monthly, or a custom period the user defines. No cap on how many goals run in parallel.

---

## Goal Types

| Type | What it means |
|------|--------------|
| Daily | A goal with specific time blocks planned for today |
| Weekly | A recurring goal tracked across the week, checked via reminders |
| Monthly | A long-horizon goal tracked across the month |
| Custom | User defines start date and end date — any period they want |

---

## Daily Goal — Creation Flow

**Step 1: Basic info**
- Goal name (e.g. "Learn AWS")
- Time window: start time → end time (e.g. 11:00 AM – 1:00 PM)

**Step 2: Subtopics**
App asks: "What do you want to cover in this session?"
User types freely: "AppSync, GraphQL, DynamoDB"

**Step 3: LLM generates time blocks**
Claude Haiku receives: goal name + total time window + subtopics.
Returns a structured breakdown assigning each subtopic a time block.

Example:
```
11:00 – 11:30  Complete GraphQL
11:30 – 12:00  Complete AppSync
12:00 – 1:00   DynamoDB
```

Progress weight per block = block_duration / total_session_duration × 100
- GraphQL: 30/120 = 25%
- AppSync: 30/120 = 25%
- DynamoDB: 60/120 = 50%

**Step 4: User edits freely**
User can change: block title, start time, end time, order.
Both the time and the subtask content are fully editable.
Any edit by the user sets `user_edited = true` on that block.

After confirming, the session is locked in. This is called a **session**.

**Data collected:**
| Field | Type | Value |
|-------|------|-------|
| `goal_id` | UUID | — |
| `goal_name` | TEXT | "Learn AWS" |
| `goal_type` | ENUM | `daily` |
| `session_start` | TIMESTAMP | 11:00 AM |
| `session_end` | TIMESTAMP | 1:00 PM |
| `subtopics_raw` | TEXT | "AppSync, GraphQL, DynamoDB" — verbatim |
| `subtask_id` | UUID | Per block |
| `subtask_title` | TEXT | "Complete GraphQL" |
| `planned_start` | TIMESTAMP | 11:00 |
| `planned_end` | TIMESTAMP | 11:30 |
| `completion_weight` | FLOAT | 25.0 (%) |
| `ai_generated` | BOOL | true / false |
| `user_edited` | BOOL | true if user changed anything |

---

## Daily Goal — Session Notification Flow

At the end of each time block, a notification fires.

### At block end → notification:
> "11:30 — GraphQL block is up. Done?"
> **[Yes ✓]   [No ✗]**

**If Yes:**
- Block marked complete
- Progress bar advances by that block's weight (e.g. +25%)
- Next block begins passively (no action needed)

**If No:**
- App opens to a reason screen
- User sees options:
  - Got distracted
  - Topic was harder than expected
  - Something came up
  - Running out of time
- Then two options: **Reschedule** or **Cancel Session**

### Reschedule flow:
User picks a new end time for the current block (e.g. extends GraphQL to 11:40).

App automatically cascades all subsequent blocks:
```
Before:
  11:00 – 11:30  GraphQL
  11:30 – 12:00  AppSync
  12:00 – 1:00   DynamoDB

User extends GraphQL to 11:40:
  11:00 – 11:40  GraphQL
  11:40 – 12:10  AppSync
  12:10 – 1:10   DynamoDB  ← warning: overlaps "another goal at 1:00"
```

### Conflict detection:
If the cascade pushes any block into the time window of another goal, a warning appears:

> "**Project Review** is scheduled at 1:00 PM. This session now runs until 1:10 PM."
> **[Auto-shift Project Review after 1:10]   [Cancel this session]**

User picks. If they choose auto-shift, the conflicting goal's start time updates to after the current session ends.

### Early completion:
At any point during a session, user can tap **"Done early"** from the session screen or widget.
- Remaining blocks marked complete
- Session end time updated to actual completion time
- Freed time shown: "You finished 22 minutes early."

---

## Weekly / Monthly / Custom Goals

These are not time-blocked like daily goals. They are commitment-based.

**Creation:**
- Goal name
- Period: weekly / monthly / custom (start + end date)
- Optional: set reminder days (e.g. every Tuesday and Friday)

**Reminders:**
On reminder days, a notification fires:
> "Quick check — have you made progress on [Goal Name] this week?"
> **[Yes, ticked ✓]   [Not yet]**

If Yes → marked as completed for that period.
If Not yet → no penalty, just logged as `not_checked_in`.

**Data collected:**
| Field | Type | Value |
|-------|------|-------|
| `goal_id` | UUID | — |
| `goal_name` | TEXT | — |
| `goal_type` | ENUM | `weekly` / `monthly` / `custom` |
| `period_start` | DATE | — |
| `period_end` | DATE | — |
| `reminder_days` | JSONB | `["tuesday", "friday"]` |
| `completion_log` | JSONB | Per-period completion booleans |

---

## What Might Be Missing — Notes

- **Goal carry-over:** If a daily goal session is skipped entirely, offer to carry it to tomorrow. Tracked as `carried_over = true`.
- **Goal velocity:** For weekly/monthly goals, track progress rate — are they on track to hit the deadline?
- **Session quality:** After a completed daily session, one word quality rating (same as focus sessions: Scattered / Distracted / Solid / Focused / Flow).
- **Goal history:** Every goal — completed, cancelled, carried — stays in history. Never deleted. Patterns surface over time.

---

---

# Night Flow

Every night, before the user sleeps, two things happen: they see their day in data, and they declare sleep.

---

## The Daily Mirror (Evening)

Around 9–10 PM, a notification brings them to this screen.

**What they see:**
- When they woke vs their intention (wake delta)
- Day type they stamped in the morning
- Focus sessions completed, total focused time, quality average
- Goals: how many blocks done, how many skipped
- Any interruptions they logged

**Then two questions:**

> "How productive did you feel today?"
> Wasted / Low / Average / Good / Locked in

> "How content are you with how you spent today?"
> Uneasy / Not great / Okay / Satisfied / Great

Two ratings, not one. A person can complete all goals (productive = 5) and still feel hollow (content = 2). That gap IS the insight — no other app captures it.

**Optional:**
> "What got in the way today?" (free text, skippable)

**Data collected:**
| Field | Type | Value |
|-------|------|-------|
| `productiveness_rating` | INT | 1–5 |
| `contentment_rating` | INT | 1–5 |
| `evening_blocker_text` | TEXT | Free text, stored verbatim |

---

## Declaring Sleep

When ready to sleep, one tap: **"Going to sleep"**

This records `declared_sleep_time`. It also starts passive tracking.

**Data collected:**
| Field | Value |
|-------|-------|
| `declared_sleep_time` | Timestamp of the tap |

---

## Post-Declaration Passive Tracking (Android)

After the user declares sleep, the app watches passively for any phone usage.

**What we track:**
- Which apps were opened
- How long each app was used
- Time of opening (relative to sleep declaration)

**App categories (Haiku assigns silently):**

| Category | Examples |
|----------|---------|
| `reels_shorts` | Instagram Reels, YouTube Shorts, TikTok |
| `social_scroll` | Twitter/X, Reddit, Facebook feed |
| `messaging` | WhatsApp, Telegram — NOT harmful by default |
| `gaming` | Any game app |
| `video_long` | YouTube full videos, Netflix, Prime |
| `reading` | Kindle, browser articles |
| `browsing_general` | Chrome, Safari with no category match |

Messaging is tracked but NOT flagged as harmful — texting a friend at night is not doom scrolling. The distinction matters.

**Data collected:**
| Field | Type | Value |
|-------|------|-------|
| `post_sleep_app_events` | JSONB | `[{app, category, duration_min, opened_at}]` |
| `total_post_sleep_screen_min` | INT | Sum of all app durations after declaration |
| `dominant_category` | TEXT | The category with most time |

This data is **never shown the same night**. It only appears in the weekly report — never as a nightly judgment.

**Weekly report example:**
> "You used your phone for an average of 22 minutes after declaring sleep this week — mostly Reels and YouTube. On those nights your actual wake time was 34 minutes later than your intention. On clean nights it was 8 minutes later."

---

## What's Missing / Open Questions

- **iOS limitation:** Post-declaration passive tracking is not available on iOS the same way. We get screen interaction proxy (was the phone used?) but not per-app breakdown. This should be surfaced honestly in the report: "iOS doesn't give us per-app detail after sleep — we can only tell you your phone was active."
- **Sleep latency:** We don't yet have a clean way to know when they actually fell asleep. Inferred from: last app event + phone idle time after. Not perfect. Label as estimate.
- **Wake time source:** For users who don't tap the morning notification, wake time is null. That null itself is data — logged as `wake_logged = false`.

---

---

# All Data Collected — Full Reference

| Table | Field | Source | Used For |
|-------|-------|--------|----------|
| `users` | `user_type`, `user_subtype` | Screen 1 | Language routing, UX path |
| `onboarding_baseline` | `best_period_text` | Screen 2 | Stored verbatim, never modified |
| `onboarding_baseline` | `extracted_domains`, `extracted_conditions`, `emotional_tone` | Haiku — Screen 2 | Seed for personal best baseline |
| `onboarding_baseline` | `best_focus_h`, `current_focus_h` | Screen 3A + 3B | Ceiling (weekly report) + half of EWMA seed |
| `onboarding_baseline` | `best_sleep_h`, `current_sleep_h` | Screen 3A + 3B | Ceiling (weekly report) + half of EWMA seed |
| `onboarding_baseline` | `best_activity_level`, `current_activity_level` | Screen 3A + 3B | Ceiling + health baseline seed |
| `onboarding_baseline` | `ewma_seed_focus`, `ewma_seed_sleep` | Computed from sliders | Days 1–6 rhythm baseline — fades by day 21 |
| `onboarding_baseline` | `primary_blocker` | Screen 3 | Week 1 notification routing |
| `onboarding_baseline` | `synthetic_me_score` | Computed from EWMA seed | Day 1 hook number — labelled as estimate |
| `user_baselines` | `ewma_value` per metric | Nightly computation | Real rhythm baseline from day 7 onward |
| `user_domains` | `health_weight`, `career_weight` | Screen 4 | Me Score weighting |
| `goals` | `goal_id`, `goal_name`, `goal_type`, `period_start`, `period_end` | Goal creation | Career domain, engagement |
| `goal_sessions` | `session_start`, `session_end`, `status` | Daily goal sessions | Goal velocity, focus tracking |
| `subtasks` | `title`, `planned_start`, `planned_end`, `completion_weight`, `ai_generated`, `user_edited` | LLM + user edit | Progress bar, planning accuracy |
| `subtask_events` | `completed_at`, `outcome`, `reschedule_reason`, `new_end_time` | Notification response | Rescheduling cascade, blocker patterns |
| `goal_conflicts` | `conflict_time`, `resolution` | Cascade detection | UX warning, scheduling intelligence |
| `evening_checkins` | `productiveness_rating`, `contentment_rating`, `evening_blocker_text` | Evening mirror | Most important daily signal |
| `sleep_events` | `declared_sleep_time` | Sleep tap | Sleep latency calculation |
| `post_sleep_app_events` | `app`, `category`, `duration_min`, `opened_at` | Passive tracking (Android) | Weekly sleep + screen report |
