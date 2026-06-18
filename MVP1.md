# Me vs Me — MVP 1

## What MVP 1 Has to Prove

One question: **Does comparing yourself to your own rhythm feel meaningful enough to keep doing it?**

Not "does the algorithm work." Not "does the score feel accurate." Those are refinement questions. The MVP question is: does a user who gets through onboarding and reaches their first weekly report feel like the app understood them?

If yes → the core loop is validated. Build on it.  
If no → something in onboarding, logging UX, or report framing is wrong. Find it fast.

Target: **50 users, 4 weeks, first weekly report reached.**

---

## What Is In MVP 1

### 1 — Onboarding (All 7 Screens)

All 7 screens ship. This is not cuttable.

The onboarding seeds the EWMA baseline. Without it, the app has no rhythm to compare against and the first two weeks of data are meaningless. Cutting or simplifying onboarding breaks the product.

| Screen | What it captures | Why it cannot wait |
|--------|-----------------|-------------------|
| Screen 1 | Name, role type | Personalised copy throughout |
| Screen 2 | Open text — ideal rhythm day | Seeds EWMA via LLM extraction |
| Screen 3 | Goals setup (simplified — see below) | Labels for session logging |
| Screen 4 | Dynamic sliders — sleep/focus/steps targets | EWMA seed values |
| Screen 5 | Tone preference — Push me / Be straight / Be kind | Changes all report copy |
| Screen 6 | Notification time preference | Fallback before Thompson Sampling has data |
| Screen 7 | Weekly report day picker | Fixed day report scheduling |

**Goals in MVP 1 are simplified:**  
User types a goal name. Tags a domain (career / health / personal growth). No LLM decomposition into subtasks. No priority weighting between goals. Sessions are tagged to a goal by name. This gets logging working without shipping the full goals engine.

**LLM usage in onboarding (MVP 1):**  
Screen 2 open text → Haiku extraction → `ewma_seed_focus`, `ewma_seed_sleep`, extraction_confidence. If confidence < 0.4 → single follow-up question. This is the one place LLM is non-negotiable in MVP 1. Without it, Screen 2 is pointless.

---

### 2 — Focus Session Logging

The core daily action.

| What | Detail |
|------|--------|
| Start / pause / end timer | In-app session timer — user explicitly starts a focus block |
| Goal tag | Session tagged to one goal from their list |
| Quality tap | 5-word scale after session ends: Scattered / Distracted / Solid / Focused / Flow |
| `session_focus_score` computed | `duration_min × quality_rating` — stored immediately |

No subtask tracking in MVP 1. No interruption logging. No behavioral quality proxy cross-validation. These are v1.1 features — they refine the quality signal, not the core loop.

---

### 3 — Evening Check-In

One notification. One screen. Two taps.

| Field | Type | What user sees |
|-------|------|---------------|
| `day_type` | Tap — 3 options | Deep Focus / Mixed / Recovery |
| `contentment_rating` | Tap — 5 options | 1 Rough → 5 Great |

That is the entire check-in. No mood logging. No open text in check-in. No prompts about what went wrong.

Evening check-in is **mandatory for `days_logged` to increment.** If it is not completed, the day does not count toward the 7-day character unlock or the first weekly report threshold.

---

### 4 — Sleep and Steps (Passive Collection)

No user input required.

| Source | iOS | Android |
|--------|-----|---------|
| Sleep hours | Apple HealthKit — `HKCategoryTypeIdentifierSleepAnalysis` | Google Health Connect — `SleepSessionRecord` |
| Steps | Apple HealthKit — `HKQuantityTypeIdentifierStepCount` | Google Health Connect — `StepsRecord` |

**Fallback for MVP 1:** If health permissions are denied or the API returns null, show a one-field manual entry card in the evening check-in: "How many hours did you sleep?" and "Roughly how many steps today?" Manual entry is stored the same way. No separate codepath.

Steps can be entered as a number or the subjective tag (Low / Moderate / High) — stored as a number (Low = 3000, Moderate = 7000, High = 11000 as defaults). These defaults can be tuned after user research.

---

### 5 — Nightly Batch Job (11:59 PM)

Runs every night. Produces `daily_scores`. Updates baselines.

**MVP 1 batch steps (simplified from full 12-step version):**

```
1. Aggregate focus sessions → daily_focus_score
2. Pull sleep_h, steps from health sync or manual entry
3. Compute health_index = (sleep_score × 0.70) + (steps_score × 0.30)
4. Write daily_scores row
5. Update EWMA baselines (focus + health)
6. Run CUSUM — flag if C⁻_t > 5σ
7. Increment days_logged if evening check-in was completed
```

Pearson is excluded from MVP 1. It requires N≥21 and the first users will not hit that during the MVP window. Ship it in the first patch update at week 4.

---

### 6 — Me Score and Weekly Report

**When it generates:**
- User's chosen report day (from Screen 7)
- Requires minimum 4 logged days in the past 7 calendar days
- First weekly report requires 7 total logged days (may span more than one week)

**What the weekly report shows in MVP 1:**

| Section | Content |
|---------|---------|
| Me Score | Single number (20–100) with delta from last week |
| Personal best | Shown if this is a new high — "New personal best" |
| Domain breakdown | Health bar + Career/Growth bar, each vs EWMA baseline |
| One insight | Template-filled sentence. MVP 1: CUSUM alert only if triggered. Otherwise: a comparison line ("Your focus was 12% above your rhythm this week.") |
| Tone-adjusted copy | All text switches based on tone preference set in onboarding |

**What is NOT in the MVP 1 report:**
- Pearson insights ("your focus correlates with sleep") — needs 21 days
- Monthly report — not enough data during MVP window
- Personal Operating Manual — day 90 premium feature

---

### 7 — Character Unlock (7-Day Retention Mechanic)

Ships in MVP 1. This is not a nice-to-have.

The 7-day window before the first weekly report is a high-dropout risk. Without something accumulating visibly, users have no reason to complete a check-in on day 3.

| Day | What the user sees |
|-----|--------------------|
| 1 | Silhouette selected — "Your [Dad/Mom/Friend/Partner] is forming" |
| 2–6 | Progressive reveal — 1 feature visible per logged day |
| 7 | Full character revealed + first report unlocked |
| 7+ | Character gives daily reactions / weekly report insights |

Character choice: **Dad / Mom / Best Friend / Partner / Sibling / Mentor**  
Character lines are tone-adjusted (Push me / Be straight / Be kind changes what they say).

This ships as static illustrations in MVP 1. No animation required. The mechanic is what matters.

---

### 8 — Notifications (Fixed Schedule in MVP 1)

Thompson Sampling requires observed data to learn from. In MVP 1, there is no data yet.

**MVP 1 notification schedule:**

| Notification | Time | Logic |
|-------------|------|-------|
| Evening check-in reminder | User's chosen time from Screen 6 (default 9 PM) | Fires if check-in not yet completed |
| Weekly report ready | Morning of report day, 8 AM | Report generated |
| Character milestone | Immediate | When a new character feature unlocks |

No session reminders in MVP 1. No "you haven't logged a session today." Keep it to 1 notification per day maximum.

**Thompson Sampling ships in v1.1** once users have 2 weeks of open/ignore data to learn from.

---

### 9 — Tone Preference (Applied Throughout)

Set once in Screen 5. Affects:
- All weekly report copy
- All character lines
- All notification messages
- CUSUM alert phrasing

Three modes:
| Mode | Flavour |
|------|---------|
| Push me | Direct, challenging — "You were below your rhythm 4 out of 7 days." |
| Be straight with me | Neutral, honest — "Your focus was below your rhythm this week." |
| Be kind | Gentle, supportive — "It was a harder week. Here is what still went well." |

This is implemented as a copy layer — same data, different strings. Low engineering cost, high perceived personalisation.

---

## What Is NOT In MVP 1

| Feature | Why it is excluded | When it ships |
|---------|-------------------|--------------|
| LLM goal decomposition into subtasks | Adds complexity to session logging before the core loop is validated | v1.1 |
| Behavioral quality proxy (cross-validate quality tap) | Refinement — needs subtask data which is not in MVP 1 | v1.1 |
| Pearson correlation insights | Requires 21 days minimum — no MVP user will hit this during test window | First patch (week 4) |
| Thompson Sampling notifications | Needs observed open/ignore data to learn — ship once baseline data exists | v1.1 |
| Monthly report | No user will have enough data during MVP window | v1.1 |
| Multiple simultaneous goals (LLM weighted) | Goals are simplified to labels in MVP 1 | v1.1 |
| Personal Operating Manual (K-means) | Day 90 premium — not relevant to MVP | v2 |
| Year in Review | Data does not exist yet | v2 |
| Habit / streak tracking | Out of scope for this product's identity | Not planned |

---

## What MVP 1 Needs to Ship

### Backend
- User accounts (auth, device sync)
- Raw event tables: `focus_sessions`, `alarm_events`, `health_sync`, `checkin_events`, `goal_events`
- `daily_scores` table + nightly batch job
- `user_baselines` table (EWMA)
- `user_alerts` table (CUSUM)
- `weekly_reports` table + report generation logic
- `user_notification_slots` table (24 rows, seeded uniformly — Thompson Sampling reads from here in v1.1)
- Notification delivery (fixed schedule)
- Screen 2 extraction pipeline: Haiku → JSON → confidence check → follow-up question if needed

### Frontend
- 7 onboarding screens
- Session timer screen with quality tap
- Evening check-in screen
- Character reveal progression (7 screens / states)
- Weekly report screen
- Home screen: "Above / On / Below rhythm" status (no raw number on home)
- Settings: tone preference, notification time, report day

### Health API
- iOS HealthKit integration (sleep + steps)
- Android Health Connect integration (sleep + steps)
- Manual entry fallback screen

---

## Build Sequence (Suggested)

```
Week 1–2 — Foundation
  ✓ Auth + user table
  ✓ Onboarding screens 1–7 (static UI, no LLM yet)
  ✓ Raw event tables created
  ✓ Health API integration (sleep + steps)

Week 3–4 — Core Loop
  ✓ Focus session timer + quality tap
  ✓ Evening check-in
  ✓ Nightly batch job (EWMA + CUSUM)
  ✓ daily_scores writes correctly

Week 5 — LLM + Onboarding Live
  ✓ Screen 2 Haiku extraction pipeline
  ✓ EWMA seeded from onboarding sliders
  ✓ Confidence check + follow-up question

Week 6 — Reports + Retention
  ✓ Weekly report generation + display
  ✓ Me Score calculation end-to-end
  ✓ Character unlock mechanic (7 states)
  ✓ Tone-adjusted copy layer

Week 7 — Notifications + Polish
  ✓ Evening check-in notification (fixed schedule)
  ✓ Weekly report notification
  ✓ Character milestone notification
  ✓ Home screen status
  ✓ Manual entry fallback for health data

Week 8 — Test + Ship to 50 users
```

---

## The 4-Week User Test — What You Are Watching

| Metric | What it tells you |
|--------|-------------------|
| % who complete onboarding | Is Screen 2 too confusing? Is goal setup too much friction? |
| % who reach 7 logged days | Is the character mechanic working? Are check-ins completing? |
| % who receive first weekly report | Is the core loop actually running end-to-end? |
| Retention at day 14 | Do people come back after the first report? |
| Open text from Screen 2 — extraction confidence distribution | Is Haiku actually pulling signal? How often is confidence < 0.4? |
| Quality rating distribution | Is everyone tapping Flow? (Gaming signal) |

If **first weekly report rate < 40%**, the dropout is happening before the product's main output. Investigate check-in completion and character mechanic — not the algorithm.

If **first weekly report rate > 70%**, the loop works. The question shifts to: does the report feel accurate? Does it feel motivating? Run interviews. Use that to scope v1.1.
