# Me vs Me — Onboarding Screens Specification

---

## Screen 1 — Who Are You?

The very first screen. One question, three options. This single answer routes the user into a completely different language layer for the rest of the app — same data, different words.

A medical student sees "revision block." An SDE sees "deep work session." A homemaker sees "task window." Nothing else changes technically — only the copy. But it makes the app feel built for them specifically.

After the primary tap, a follow-up appears on the same screen asking for more detail about their subtype.

**What the user sees:**
> "What best describes you?"
> [ Student ]  [ Working ]  [ Home Maker ]

**Follow-up (same screen, appears after tap):**

| Primary choice | Follow-up question | Options |
|---------------|-------------------|---------|
| Student | What are you studying toward? | Competitive exam / Degree / Skill building / School (9–12) / Other |
| Working | What describes your work? | Job (salaried) / Freelance/gig / Running a business / Teaching/coaching / Other |
| Home Maker | What does your day mostly involve? | Managing household / Raising children / Caregiving / Home + side work / Other |

**Data collected:**
| Field | Type | Value | Used For |
|-------|------|-------|----------|
| `user_type` | ENUM | `student` / `working` / `homemaker` | Routes all app language and UX copy |
| `user_subtype` | ENUM | e.g. `competitive_exam`, `salaried`, `caregiving` | Finer language personalisation |

---

## Screen 2 — Your Best Period

The most important question in the entire onboarding. Not because of what they type — because of what typing it forces them to do. They have to remember a version of themselves that felt better. That memory becomes the emotional anchor for everything that follows.

The question is deliberately worded to pull answers toward energy, sleep, and output — the signals that map to Health and Career — without naming those domains. If someone writes about a relationship or a personal event, Haiku evaluates confidence and fires one gentle follow-up below the text field.

**What the user sees:**
> "Think about a time you felt sharp, consistent, and like yourself — even just two weeks. What were your days actually like? How did you wake up, how much did you get done, how did you feel physically?"

Free text field. No character limit. No options. Just write.

**If Haiku confidence < 0.4, one follow-up appears below (not a new screen):**
> "Sounds like that time mattered. How were you sleeping and how focused were you during that period?"

User can skip. Their original text is always stored verbatim regardless.

**What Haiku extracts silently (scoped to MVP1 domains):**

| Signal | Examples | Maps to |
|--------|---------|---------|
| Sleep quality | "sleeping well", "woke early", "felt rested" | Health |
| Energy | "felt sharp", "had energy", "wasn't tired" | Health |
| Physical activity | "going to the gym", "walking daily" | Health |
| Focus / output | "getting a lot done", "studying consistently" | Career |
| Consistency | "had a routine", "planned my days" | Career |
| Screen habits | "wasn't on my phone", "less distracted" | Both |

**Data collected:**
| Field | Type | Value | Used For |
|-------|------|-------|----------|
| `best_period_text` | TEXT | Raw free text — verbatim, never modified | Stored as permanent anchor |
| `extracted_health_signals` | JSONB | `["early wake", "gym", "felt rested"]` | Seeds health domain baseline |
| `extracted_career_signals` | JSONB | `["consistent study", "planned days"]` | Seeds career domain baseline |
| `extracted_other` | JSONB | Off-domain signals (family, emotions, etc.) | Stored for future domains, not scored in MVP1 |
| `extraction_confidence` | FLOAT | 0.0 – 1.0 | Below 0.4 triggers follow-up question |
| `emotional_tone` | TEXT | `calm` / `proud` / `energetic` / `in_control` | Tone context for weekly report language |

---

## Screen 3A — Calibration: Your Ideal Day

First of two calibration passes. This one captures the ceiling — what the user knows they are capable of on a great day. Not used as the daily benchmark. Stored as aspiration, shown in weekly/monthly reports only.

Sliders are **interdependent and dynamic**. Total hours in a day = 24. As one slider fills up, the max of other sliders shrinks in real time.

> Rule: `sleep_max = 24 − primary_activity_hours`
> Example: Study hours set to 18 → sleep slider range becomes 1–6 only.

Steps are captured as a **number input**, not Low/Moderate/High. Followed immediately by a subjective tag so we know if that number is their ceiling or just a normal day.

**What the user sees:**
> "On your ideal day — when you were at your best — what did it look like?"

**Students:**
| Question | Input type | Constraint |
|----------|-----------|------------|
| Hours truly focused on your best study day | Slider 1–24h | Primary anchor |
| Hours slept that same day | Slider 1–(24 − study_h) | Dynamic max |
| Steps on a good day | Number input | — |
| Is that your best, could be better, or just right? | My best / Could be better / Just right | — |
| What gets in your way most? | Phone / Low energy / Inconsistency / No motivation / Can't focus | Single tap |

**Working Professionals:**
| Question | Input type | Constraint |
|----------|-----------|------------|
| Hours of deep focused work on your most productive day | Slider 1–24h | Primary anchor |
| Hours slept that same day | Slider 1–(24 − work_h) | Dynamic max |
| Steps on a good day | Number input | — |
| Is that your best, could be better, or just right? | My best / Could be better / Just right | — |
| Biggest blocker? | Phone / Low energy / Inconsistency / No motivation / Can't focus | Single tap |

**Homemakers:**
| Question | Input type | Constraint |
|----------|-----------|------------|
| Hours spent on planned tasks (house, kids, work) on your best day | Slider 1–24h | Primary anchor |
| Hours slept that same day | Slider 1–(24 − task_h) | Dynamic max |
| Steps on a good day | Number input | — |
| Is that your best, could be better, or just right? | My best / Could be better / Just right | — |
| What gets in your way most? | Phone / Low energy / Inconsistency / No motivation / No time for self | Single tap |

**Data collected:**
| Field | Type | Value | Used For |
|-------|------|-------|----------|
| `best_focus_h` | FLOAT | e.g. 7.0 | Ceiling for career domain — weekly report only |
| `best_sleep_h` | FLOAT | e.g. 7.5 | Ceiling for health domain — weekly report only |
| `best_steps` | INT | e.g. 8000 | Ceiling for activity metric |
| `best_steps_assessment` | ENUM | `my_best` / `could_improve` / `just_right` | Calibrates whether the number is their peak or baseline |
| `primary_blocker` | ENUM | `phone` / `low_energy` / `inconsistency` / `no_motivation` / `cant_focus` | Routes week 1 notification topics |

---

## Screen 3B — Calibration: Your Average Day Right Now

Second calibration pass. Same questions, different frame. This captures the honest current state — where the user actually is today, not where they aspire to be.

Both the ideal (3A) and current (3B) values are averaged to compute the EWMA seed used for days 1–6. Neither is the permanent benchmark — the EWMA takes over from real logged data starting day 7.

**What the user sees:**
> "Now honestly — what does your average day actually look like right now?"

Same slider layout as Screen 3A. Same dynamic constraints. Same steps input.

**Data collected:**
| Field | Type | Value | Used For |
|-------|------|-------|----------|
| `current_focus_h` | FLOAT | e.g. 3.0 | Half of EWMA seed for focus metric |
| `current_sleep_h` | FLOAT | e.g. 5.5 | Half of EWMA seed for sleep metric |
| `current_steps` | INT | e.g. 3500 | Half of EWMA seed for activity metric |
| `current_steps_assessment` | ENUM | `my_best` / `could_improve` / `just_right` | Context for current state |
| `ewma_seed_focus` | FLOAT | `(best_focus_h + current_focus_h) / 2` | Days 1–6 rhythm baseline |
| `ewma_seed_sleep` | FLOAT | `(best_sleep_h + current_sleep_h) / 2` | Days 1–6 rhythm baseline |
| `ewma_seed_steps` | FLOAT | `(best_steps + current_steps) / 2` | Days 1–6 activity baseline |

> **Note:** The EWMA seed is replaced by real data from day 7 and fully retired by day 21. It is always labelled "estimated" in the UI.

---

## Screen 4 — Domain Weights

The user decides how much Health and Career matter to them. The Me Score is built directly from this ratio — it is their definition of a good day, not the app's.

Two sliders that must always sum to 100. They can be changed anytime from settings.

**Default values by user type:**
| User type | Health default | Career / Growth default |
|-----------|---------------|------------------------|
| Student | 35% | 65% |
| Working | 40% | 60% |
| Home Maker | 50% | 50% |

**For Homemakers — what "Career/Growth" means:**
Homemakers may have no traditional study or work goal. For them, this domain is labelled **Personal Growth** and comes with structured suggestions:

| Level | Examples |
|-------|---------|
| Pick up a hobby | Painting, gardening, cooking, music |
| Learn a skill | Online course, language, coding basics |
| Teach something | Coaching a neighbour, sharing a skill |
| Build something | Side project, family history doc, blog |

These are shown as suggestions, not requirements. They can pick one, pick none, or write their own. Goals in this domain follow the same weekly/monthly goal system as any other user.

**What the user sees:**
> "What matters more to you right now?"
> [Health ←——————→ Career/Growth]
> Two sliders. Always sum to 100.

**Data collected:**
| Field | Type | Value | Used For |
|-------|------|-------|----------|
| `health_weight` | INT | 0–100 | Me Score formula weight for health domain |
| `career_weight` | INT | 0–100 (= 100 − health) | Me Score formula weight for career domain |
| `career_domain_label` | TEXT | `career` / `learning` / `personal_growth` | Sets domain label throughout the app |
| `homemaker_growth_choice` | TEXT | e.g. `hobby`, `learn_skill`, `custom` | Homemakers only — seeds their growth goal |

---

## Screen 5 — Tone Preference

One question. Three taps. Controls how the app talks to the user — in notifications, weekly reports, and all insight language. The underlying data never changes. Only the words.

This is set once in onboarding and is always changeable in profile settings. It is invisible infrastructure — no badge or label appears in the main UI saying which mode they're in.

**What the user sees:**
> "How do you want me to talk to you?"

| Option | Feel |
|--------|------|
| **Push me** | Direct. Names gaps clearly. Shows declines. Streaks shown as streaks. |
| **Be straight with me** | Default. Observational. Honest but not harsh. Balanced framing. |
| **Be kind** | Leads with wins. Softens gaps. No streak counter. Never names a decline without a win first. |

**What changes based on tone:**
| Element | Push me | Be straight | Be kind |
|---------|---------|-------------|---------|
| Report gap framing | "You're 23% below your March baseline" | "Focus is tracking below your baseline" | "You have 23% more to unlock" |
| Streak display | Streak counter + broken state shown | Presence rate (5 of 7 days) | Hidden entirely |
| Missed blocks | "3 blocks skipped today" | "A few blocks got skipped" | "Some blocks carried over — that's okay" |
| CUSUM drift alert | Shows at 7 days below baseline | Shows at 7 days | Shows at 14 days, softer language |
| Notification copy | "Block is up. Done or not?" | "Block is up. Done?" | "Time's up — all good?" |

**Data collected:**
| Field | Type | Value | Used For |
|-------|------|-------|----------|
| `tone_preference` | ENUM | `push_me` / `straight` / `be_kind` | Applied to all report copy, notifications, insight language |

---

## Screen 6 — Character Unlock

The last onboarding screen. No data entry. No sliders. One emotional choice.

The cold-start period (days 1–7) is when most users drop off — there are no insights yet, no patterns, no reason to come back. The character unlock solves this without gamification. No points, no badges, no streaks to break.

The user picks someone they care about. The character appears immediately — as a full silhouette. Locked.

> "Log for 7 days to unlock them."

That's it.

**What the user sees:**
> "Who do you want to unlock?"
> [ Dad ]  [ Mom ]  [ Best Friend ]  [ Partner ]  [ Sibling ]  [ Mentor ]

Character appears as a dark silhouette below the options. One line: *"Log for 7 days to unlock them."*

**The 7-day reveal — one step per logged day (evening check-in completed):**
| Day | What changes |
|-----|-------------|
| 1 | Silhouette only |
| 2 | Outline appears |
| 3 | Color fills in |
| 4 | Face becomes visible |
| 5 | Expression appears |
| 6 | Almost fully revealed |
| 7 | Full unlock — animation plays + message in their voice |

**Day 7 unlock messages by character:**
| Character | Message |
|-----------|---------|
| Dad | *"You showed up every day for a week. I don't say this enough — I'm proud of you."* |
| Mom | *"Seven days. I knew you could. Now don't stop."* |
| Best Friend | *"Okay okay I see you. Seven days straight. We're celebrating this."* |
| Partner | *"I've been watching. You didn't give up. That means everything."* |
| Sibling | *"You actually did it. Seven days. That's kind of insane. Keep going."* |
| Mentor | *"Consistency is the rarest thing. You just proved you have it."* |

Missing a day does not reset the counter — the character stays at its current reveal state. Come back the next day, it continues.

After day 7 the character becomes the weekly report narrator and notification voice — the bridge into the V2 AI companion feature.

**Data collected:**
| Field | Type | Value | Used For |
|-------|------|-------|----------|
| `character_type` | ENUM | `dad` / `mom` / `best_friend` / `partner` / `sibling` / `mentor` | Sets voice for weekly reports and notifications post-unlock |
| `character_unlock_day` | INT | 0–7 | Current reveal progress |
| `character_unlocked` | BOOL | `true` after 7 logged days | Gates companion voice feature |
| `unlock_completed_at` | TIMESTAMP | When full unlock happened | Analytics — median time to unlock |

---

## Screen 7 — Weekly Report Day + First 7 Days Promise

This is the last setup screen before the hook. Two things happen here: the user picks when they want their weekly report, and the app explains what the next 7 days look like. This screen is where the character unlock mechanic gets its context — the user now understands *why* 7 days matters.

**What the user sees:**

> "When do you want to see your weekly report?"

Seven day chips, single tap:
> [ Mon ]  [ Tue ]  [ Wed ]  [ Thu ]  [ Fri ]  [ Sat ]  [ Sun ]

Then below, a short card explaining what happens next:

---

> **Your first 7 days**
>
> Log each day and two things happen:
> - Your [character] comes to life — a little more each day
> - You get daily insights as the app learns your patterns
>
> After 7 logged days, your first weekly report unlocks and your [character] is fully revealed.

---

The character name shown here is the one they chose in Screen 6 — "your Dad", "your best friend", etc. It's already personal before they've logged a single day.

**How weekly reports work — rules the user never sees but the system follows:**

| Situation | What happens |
|-----------|-------------|
| User chose Sunday, joins on Wednesday | First Sunday: 4 days only → no report, character progress shown instead |
| Second Sunday onward | Report fires if ≥ 4 days logged that week |
| Week with 1–3 logged days | Brief summary card only, no full report |
| Week with 0 logged days | Nothing shown, no guilt |
| First full report | Requires 7 total logged days — fires on the next chosen day after threshold is reached |

After the first report, the weekly report fires on the chosen day every week forever. Always covers the last 7 calendar days. Not 7 logged days — always 7 calendar days so comparisons between weeks stay consistent.

**What they get during the first 7 days instead of a report:**

Each day they log, the app shows a small daily insight card — not deep analytics (not enough data yet), but rule-based personalised nudges based on their `primary_blocker` from Screen 3A:

| Blocker | Day 1–7 nudge style |
|---------|-------------------|
| Phone | Tips on phone-free morning windows |
| Low energy | Sleep consistency prompts |
| Inconsistency | Morning intention framing |
| No motivation | Why the character is worth unlocking |
| Can't focus | Session quality tips |

Plus the character reveal progress bar — always visible on the home screen. *"Day 3 of 7 — keep going."*

**Data collected:**

| Field | Type | Value | Used For |
|-------|------|-------|----------|
| `report_day` | ENUM | `mon` / `tue` / `wed` / `thu` / `fri` / `sat` / `sun` | Weekly report notification day |
| `days_logged` | INT | 0 → increments each evening check-in | Character unlock + first report eligibility |
| `first_report_eligible` | BOOL | True when `days_logged >= 7` | Gates first weekly report generation |
| `week_start` | DATE | Computed — 7 days before report_day | Weekly report window start |
| `week_end` | DATE | The chosen report_day | Weekly report window end |
| `days_logged_this_week` | INT | Count of check-ins in current 7-day window | Minimum threshold check (≥ 4 to generate report) |
| `report_generated` | BOOL | False if `days_logged_this_week < 4` | Prevents thin reports from firing |

---

## End of Onboarding — The Hook

Immediately after Screen 6, before the user sees the main app, one screen appears. No input required. Just a number.

The synthetic Me Score — computed from the EWMA seed (average of ideal and current slider values), weighted by their domain choices.

**What the user sees:**
> "Based on what you told us — you're focusing around 3–4 hours a day and sleeping around 6 hours."
>
> **Your rhythm estimate: 61%**
>
> *"Log a few days and we'll replace this with your real number."*

- Framed around **rhythm**, not best self — so it doesn't feel like a failing grade on day 1
- Clearly labelled as an estimate
- Replaced by real EWMA data starting day 7
- The number is the hook — they want to find out if it's accurate

---

## Full Data Reference — All Onboarding Fields

| Table | Field | Source | Used For |
|-------|-------|--------|----------|
| `users` | `user_type` | Screen 1 | Language routing across entire app |
| `users` | `user_subtype` | Screen 1 | Fine-grained copy personalisation |
| `onboarding_baseline` | `best_period_text` | Screen 2 | Stored verbatim — never modified |
| `onboarding_baseline` | `extracted_health_signals` | Screen 2 (Haiku) | Seeds health domain context |
| `onboarding_baseline` | `extracted_career_signals` | Screen 2 (Haiku) | Seeds career domain context |
| `onboarding_baseline` | `extracted_other` | Screen 2 (Haiku) | Off-domain signals for future use |
| `onboarding_baseline` | `extraction_confidence` | Screen 2 (Haiku) | Triggers follow-up if < 0.4 |
| `onboarding_baseline` | `emotional_tone` | Screen 2 (Haiku) | Weekly report tone context |
| `onboarding_baseline` | `best_focus_h` | Screen 3A | Ceiling — weekly/monthly report only |
| `onboarding_baseline` | `best_sleep_h` | Screen 3A | Ceiling — weekly/monthly report only |
| `onboarding_baseline` | `best_steps` | Screen 3A | Ceiling for activity metric |
| `onboarding_baseline` | `best_steps_assessment` | Screen 3A | Calibrates if number is peak or normal |
| `onboarding_baseline` | `primary_blocker` | Screen 3A | Week 1 notification routing |
| `onboarding_baseline` | `current_focus_h` | Screen 3B | Half of EWMA seed |
| `onboarding_baseline` | `current_sleep_h` | Screen 3B | Half of EWMA seed |
| `onboarding_baseline` | `current_steps` | Screen 3B | Half of EWMA seed |
| `onboarding_baseline` | `ewma_seed_focus` | Computed | Days 1–6 rhythm baseline for focus |
| `onboarding_baseline` | `ewma_seed_sleep` | Computed | Days 1–6 rhythm baseline for sleep |
| `onboarding_baseline` | `ewma_seed_steps` | Computed | Days 1–6 rhythm baseline for steps |
| `onboarding_baseline` | `synthetic_me_score` | Computed | Day 1 hook number |
| `user_domains` | `health_weight` | Screen 4 | Me Score weight for health |
| `user_domains` | `career_weight` | Screen 4 | Me Score weight for career |
| `user_domains` | `career_domain_label` | Screen 4 | Domain label across app |
| `user_domains` | `homemaker_growth_choice` | Screen 4 | Seeds growth goal (homemakers only) |
| `users` | `tone_preference` | Screen 5 | All report copy and notification language |
| `users` | `character_type` | Screen 6 | Weekly report voice post-unlock |
| `users` | `character_unlock_day` | Daily update | Reveal progress tracker |
| `users` | `character_unlocked` | Day 7 trigger | Gates companion voice feature |
| `users` | `unlock_completed_at` | Day 7 trigger | Analytics |
| `users` | `report_day` | Screen 7 | Weekly report notification day |
| `weekly_reports` | `days_logged` | Daily update | Character unlock + first report eligibility |
| `weekly_reports` | `first_report_eligible` | Computed | Gates first weekly report |
| `weekly_reports` | `week_start` | Computed | Report window start |
| `weekly_reports` | `week_end` | Computed | Report window end |
| `weekly_reports` | `days_logged_this_week` | Computed | Minimum threshold check |
| `weekly_reports` | `report_generated` | Computed | False if days_logged_this_week < 4 |
