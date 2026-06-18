# Me vs Me — Me Score

The Me Score is a single number (0–100) that represents how close you are to your personal rhythm this week. It is not compared to other people. It is not compared to your best day. It is compared to the version of you that your own logged data says is normal.

---

## The Full Calculation Chain

```
Focus Sessions logged during the day
        ↓
session_focus_score = duration × quality
        ↓
daily_focus_score = SUM of all sessions that day
        ↓
health_index = sleep(50%) + energy(30%) + steps(20%)
        ↓
Weekly averages of both
        ↓
career_domain = (week_avg_focus / ewma_baseline) × 100
health_domain = week_avg_health_index
        ↓
me_score = (health_domain × health_weight)
         + (career_domain × career_weight)
        ↓
Apply floor (20) → update personal best if exceeded
```

---

## Step 1 — Session Focus Score

Every focus session the user logs produces a score.

```
session_focus_score = actual_duration_min × quality_rating
```

**Quality rating mapping:**

| Word shown to user | Internal value |
|--------------------|---------------|
| Scattered | 1 |
| Distracted | 2 |
| Solid | 3 |
| Focused | 4 |
| Flow | 5 |

**Why this formula:**
A 60-minute Focused session = 60 × 4 = **240**
A 90-minute Scattered session = 90 × 1 = **90**

The shorter, higher-quality session wins. This is deliberate — the app rewards quality over raw hours. Sitting at a desk for 3 hours while distracted should not score higher than 90 minutes of real work.

**Data fields:**

| Field | Type | Value | Source |
|-------|------|-------|--------|
| `actual_duration_min` | INT | e.g. 72 | session end − session start |
| `quality_rating` | INT | 1–5 | user word tap after session ends |
| `session_focus_score` | FLOAT | e.g. 288 | computed: duration × quality |

---

## Step 2 — Daily Focus Score

At the end of each day, all session scores are summed.

```
daily_focus_score = SUM(session_focus_score) for all sessions that day
```

A day with two sessions — 60 min Focused (240) and 45 min Solid (135) — scores **375** total.

This goes into `daily_scores` as one field. The nightly batch job computes this at 11:59 PM.

**Data fields:**

| Field | Type | Value | Source |
|-------|------|-------|--------|
| `daily_focus_score` | FLOAT | e.g. 375 | Nightly computed from sessions |

---

## Step 3 — Health Index (Daily, 0–100)

Computed every night. Three sub-scores, each normalised to 0–100, then weighted.

```
sleep_score = min((sleep_h / 8.0) × 100, 100)
steps_score = min((steps / 10000) × 100, 100)

health_index = (sleep_score × 0.70)
             + (steps_score × 0.30)
```

**Why energy is removed:**
Energy changes constantly for factors outside the app's control — stress, food, weather, mood. A self-reported energy score at any fixed time of day is too noisy to be meaningful. Removed entirely.

**Why these weights:**
- Sleep at 70% — the dominant driver. Walker (2017) and Van Dongen et al. (2003) show sleep is the single strongest physical variable affecting next-day cognition. With energy gone, sleep carries more.
- Steps at 30% — passive data from Google Fit / Apple Health. No user input needed. Clean signal.

Both inputs are either passively collected or already logged for another reason. Zero extra notifications.

**Worked example:**

| Metric | Raw value | Calculation | Sub-score |
|--------|-----------|-------------|-----------|
| Sleep | 7h | (7 / 8) × 100 | 87.5 |
| Steps | 6,000 | (6000 / 10000) × 100 | 60.0 |

```
health_index = (87.5 × 0.70) + (60.0 × 0.30)
             = 61.25 + 18.00
             = 79.25
```

**Sources for benchmarks:**
- 8h sleep → NSF (2015) optimal adult sleep recommendation
- 10,000 steps → WHO (2020) physical activity guidelines

**Data fields:**

| Field | Type | Value | Source |
|-------|------|-------|--------|
| `sleep_h` | FLOAT | e.g. 7.0 | Alarm events / passive Health API |
| `steps` | INT | e.g. 6000 | Google Fit / Apple Health — passive, no user input |
| `sleep_score` | FLOAT | 0–100 | Computed |
| `steps_score` | FLOAT | 0–100 | Computed |
| `health_index` | FLOAT | 0–100 | Computed — stored in daily_scores |

---

## Step 4 — EWMA Baseline (The Rhythm)

The Me Score does not compare you to your best day. It compares you to your **personal rhythm** — the EWMA baseline built from real logged data.

```
S_t = (0.1 × today_score) + (0.9 × yesterday_baseline)
```

- λ = 0.1 → effective memory window of ~19 days
- Updated every night for every metric: focus_score, health_index, contentment_rating
- Stored in `user_baselines` table — one row per user per metric

**Three phases:**

| Phase | Days | Baseline source |
|-------|------|----------------|
| Cold start | 1–6 | EWMA seed from onboarding sliders — labelled as estimate |
| Transition | 7–20 | Seed weight fades, real data takes over |
| Real baseline | 21+ | Fully data-driven — seed has zero weight |

**Recovery days excluded:**
Days stamped as Recovery are excluded from EWMA computation. Intentional rest should not pull the rhythm down.

**Data fields:**

| Field | Type | Value | Source |
|-------|------|-------|--------|
| `ewma_focus` | FLOAT | e.g. 310 | Nightly computed — `user_baselines` |
| `ewma_health` | FLOAT | e.g. 74.0 | Nightly computed — `user_baselines` |
| `ewma_contentment` | FLOAT | e.g. 3.4 | Nightly computed — `user_baselines` |

---

## Step 5 — Weekly Domain Scores

Computed every week on the user's chosen report day. Reads from `daily_scores` for the last 7 calendar days.

```
career_domain = min((week_avg_focus_score / ewma_focus_baseline) × 100, 100)

health_domain = week_avg_health_index
              → already 0–100, no transformation needed
```

**What career_domain means:**
- 100 → matched or exceeded your rhythm this week — ceiling
- 85  → 15% below your rhythm
- 60  → well below your rhythm

Exceeding your rhythm is still a win — but the score just hits 100 and stops there. The reward for an exceptional week is your **personal best updating**, not a number above 100. A score above 100 has no intuitive meaning.

**Worked example:**

| Metric | Value |
|--------|-------|
| week_avg_focus_score | 340 |
| ewma_focus_baseline | 310 |
| career_domain | (340 / 310) × 100 = **109.7 → capped at 110** |
| week_avg_health_index | 74.0 |
| health_domain | **74.0** |

---

## Step 6 — Me Score (Final)

```
me_score = (health_domain × health_weight / 100)
         + (career_domain × career_weight / 100)
```

User set Health 40% / Career 60%:

```
me_score = (74.0 × 0.40) + (110 × 0.60)
         = 29.6 + 66.0
         = 95.6
```

**Two rules applied after calculation:**

```
Rule 1 — Floor:
me_score = max(20, me_score)
→ Score never drops below 20. Prevents the app from feeling punishing on hard weeks.

Rule 2 — Personal best:
if me_score > all_time_personal_best:
    all_time_personal_best = me_score
→ Personal best only ever moves up. Never resets. Never goes down.
```

**Data fields:**

| Field | Type | Value | Source |
|-------|------|-------|--------|
| `career_domain` | FLOAT | 0–100 | Computed weekly — capped at 100 |
| `health_domain` | FLOAT | 0–100 | Computed weekly |
| `me_score` | FLOAT | 20–100 | Computed weekly — stored in weekly_reports |
| `me_score_delta` | FLOAT | e.g. +6 or −4 | me_score − last week's me_score |
| `personal_best` | FLOAT | Highest ever me_score | Updated if new me_score exceeds it |
| `pct_of_personal_best` | FLOAT | (me_score / personal_best) × 100 | Shown in weekly report as context |

---

## How Me Score Appears to the User

**Daily (no score shown):**
> "Above your rhythm" / "On rhythm" / "Below your rhythm"
The raw number is not shown daily. Just the direction. Less pressure.

**Weekly report headline:**
> "Your Me Score this week: **81**"
> "+6 from last week · 89% of your personal best (91)"

**Monthly report:**
> "Your average Me Score this month: **77**"
> "+4 from last month · Your best month ever was March at 84"

**Year in Review:**
> "You averaged **73** across the year. Your best week: 91. Your growth: January you scored 54. December you scored 81."

---

## Me Score — Complete Field Reference

| Table | Field | What it is |
|-------|-------|-----------|
| `focus_sessions` | `session_focus_score` | duration × quality per session |
| `daily_scores` | `daily_focus_score` | SUM of sessions that day |
| `daily_scores` | `sleep_h` | Hours slept |
| `daily_scores` | `sleep_h` | Hours slept — from alarm events / Health API |
| `daily_scores` | `steps` | Step count from health API |
| `daily_scores` | `health_index` | Weighted sub-score composite 0–100 |
| `user_baselines` | `ewma_focus` | Rolling 19-day average of focus score |
| `user_baselines` | `ewma_health` | Rolling 19-day average of health index |
| `weekly_reports` | `career_domain` | Week avg focus vs EWMA, 0–110 |
| `weekly_reports` | `health_domain` | Week avg health index, 0–100 |
| `weekly_reports` | `me_score` | Final weighted composite, 20–110 |
| `weekly_reports` | `me_score_delta` | Change from previous week |
| `weekly_reports` | `personal_best` | All-time highest me_score |
| `weekly_reports` | `pct_of_personal_best` | me_score as % of personal best |
| `user_domains` | `health_weight` | User-defined weight for health domain |
| `user_domains` | `career_weight` | User-defined weight for career domain |
