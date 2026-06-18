Me vs Me — Complete Product & Technical Document

PART 1 — ONBOARDING
What onboarding needs to accomplish
One job only. Understand who this person is and what they want to improve — in their own words. By the end of onboarding, before they log a single day, they should see one sentence that feels uncomfortably accurate about themselves. That moment of recognition is what makes them stay.

Screen 1 — The basics
Are you a student or working?
Two options. That's it. This one answer changes almost everything — the language, the examples, the insights, the goal templates.
If student → follow-up: What are you studying toward? Options: Competitive exam (JEE/NEET/UPSC/CA), Degree (engineering/medical/arts/commerce), Skill building (coding/design/language), School (class 9–12), Other.
If working → follow-up: What describes your work? Options: Job (salaried), Freelance/gig, Running a business, Teaching/coaching, Other.
Why we collect this: Every single piece of language in the app changes based on this. A medical student sees "study session" and "revision block." An SDE sees "deep work" and "coding session." A freelancer sees "client work" and "deliverable." Same underlying data. Completely different experience.

Screen 2 — The real question
"Think about the last time you felt really on top of your life — even if it was just two weeks. What was different about that time?"
Free text field. No character limit. No options to select. Just write.
This is the most important question in the entire app. Not because of what they type — because of what typing it makes them do. They have to remember a version of themselves that felt better. That memory becomes the anchor for everything.
What we extract from this text using Claude Haiku: which domains they mentioned (sleep? focus? less phone?), what conditions they described (woke early? studied in library? ate better?), what emotional tone they used (calm? proud? energetic?). This becomes the seed of their personal best baseline.

Screen 3 — Calibration sliders
For students:

On your best study day, how many hours were you truly focused? (slider 1–12)
Right now on an average day, how many hours do you actually focus? (slider)
During that good period, how many hours were you sleeping? (slider 4–10)
Right now, how many hours do you sleep most nights? (slider)
What gets in your way most? (Phone / Low energy / Inconsistency / No motivation / Can't focus — tap one)

For working professionals:

On your most productive day, how many hours of deep focused work do you get? (slider)
Right now on an average day, how many deep hours? (slider)
Sleep during your best period vs now (same sliders)
Biggest blocker (same options)

Why every field matters:
Best focus hours → this becomes the ceiling for the focus domain. The gap between this and current is the first insight they see.
Current focus hours → shows how far they have drifted from their own best. Not compared to anyone else. Compared to their own history.
Best sleep hours → anchor for the health domain. Without a reference point, we cannot compute a gap.
Current sleep → actual health baseline today.
Primary blocker → routes the first week of notifications. If they say phone, first week nudges are about screen use. If they say low energy, first nudges are about sleep. The app feels immediately personalised.

Screen 4 — Domain weights
Two domains only right now. Health and Career/Learning. User sees two sliders that must sum to 100.
Default: Health 40%, Career 60%.
The user can adjust. A student in exam season might push career to 80%. Someone recovering from burnout might push health to 70%.
Why this matters: Their Me Score is literally built from these weights. If they said career matters 70% and health 30%, then a great study week with poor sleep scores higher than a healthy but unproductive week. Their definition. Their score. Never a generic formula imposed on them.

What they see at the end of onboarding
Immediately after completing these screens, the app shows:
"Based on what you told us — you used to study around 5–6 hours a day, sleep around 7 hours, and you felt sharp. Right now you're at roughly 2–3 hours of focus and 5–6 hours of sleep. You're at about 54% of your best self. That means 46% is still available."
This number is built purely from their slider answers. It is an honest estimate, clearly labeled as such. It gets replaced by real data within 7 days of logging. But seeing it on day 1, before logging anything, is the hook that makes them want to find out if it's accurate.

PART 2 — THE DAILY FLOW, MORNING TO NIGHT

Night before — setting tomorrow's intention
Before sleeping, the user does two things:
1. Set tomorrow's wake time intention
Not an alarm. Not a reminder. An intention. They type or tap what time they want to wake up tomorrow. 6:00 AM. That's stored as a commitment to their future self.
Why this framing matters: the word "intention" changes the psychology. An alarm is something that happens to you. An intention is something you set. The responsibility is different. The ownership is different.
Data stored: intended_wake_time (timestamp), date (for tomorrow), set_at (when they set it — tells us whether they planned ahead or set it last minute).

2. Plan tomorrow
Free text field: what do you want to get done tomorrow?
They write anything. "Finish chapter 4 of organic chemistry. Solve 10 thermodynamics problems. Read 20 pages of Clean Code."
Time slots: when are you working? They add one or more windows. 9 AM to 11 AM. 2 PM to 5 PM. Whatever fits their day.
Both the goals text and the time slots are sent together to Claude Haiku. Haiku maps the goal chunks into the time windows and returns a structured plan. The user sees it, edits freely, and confirms.
What the app stores from this planning session:
goals_raw → exact text they typed, preserved verbatim. Never modified by the app.
time_slots → the windows they declared as work time.
time_blocks → the AI-generated breakdown. Each block has a title (AppSync), a start time (9:00), an end time (9:30), and a weight (what percentage of the parent goal this block represents).
created_how → did they plan the night before or morning of? This alone becomes a weekly insight: "You planned the night before on 5 of 7 days this week. Your goal completion rate on those days was 40% higher than days you planned the morning of."
ai_generated and user_edited flags on each block → tells us how much the user trusts Claude's decomposition vs adjusts it. After 30 days we can say "you consistently extend the time Claude gives to coding problems and shrink reading blocks — next time we'll suggest that distribution."

Morning — wake up flow
Around the intended wake time, the app sends a warm notification. Not a loud alarm. Something like:
"Hey sunshine ☀️ It's 6:00 AM. Woke up?"
Two buttons: Yes and Not Yet.
If they tap Yes, the app records the tap time. If they tapped at 6:04, the app shows: "Good morning 🌞 Looks like you woke around 6:04. Is that right?" They can edit this to their actual wake time.
If they tap Not Yet, no judgment. The notification quietly disappears. If they open the app later and log their actual wake time, that gets recorded. If they never log it, that day has a null wake time — which itself is data.
What we now know from just this one interaction:
actual_wake_time → when they actually woke.
wake_delta_min → actual_wake_time minus intended_wake_time. Positive means they woke late. Negative means they woke early. This is the first signal of whether intention translated to action today.
logged_wake → did they tap the notification or open the app manually? Tells us engagement pattern.
Why this data matters, concretely:
After 3 weeks the app can say: "Days you woke within 15 minutes of your intended time had 30% higher goal completion. Days you woke more than 45 minutes late, you completed on average 1.2 of your planned blocks vs 2.8 on early days." That is a real, personal, data-backed insight. Nobody else gives this to a user.

Morning — intention stamp (replaces old energy rating)
Shortly after they log waking up, one more question. Not a 1–5 scale. A real question.
"What kind of day is today?"
Three options:

Deep focus — I want to lock in today
Mixed — some work, some other things
Recovery — I need a lighter day

This one tap does more than the energy rating ever did. It tells us the user's intent for the day. Recovery days should not be scored the same as deep focus days. If someone plans a recovery day and actually rests, that is not a failure — that is exactly right. The day_type field protects rest from being penalised.
Data stored: day_type, stamp_time (late morning stamps — after 10 AM — correlate with lower goal completion, surfaced after 21 days).

During the day — focus sessions
When they start a work block, they tap Start in the app or from the sticky widget on their home screen. When they finish, they tap End. Three seconds of interaction total.
Immediately after End, one question: "How was that session?" Five options shown as words, not numbers: Scattered / Distracted / Solid / Focused / Flow. (This solves the original complaint about meaningless numbers. Words have texture. "Scattered" means something. "3" does not.)
Data stored per session:
start_time, end_time, actual_duration_min → how long it actually was.
planned_duration_min → how long they intended when they started. The gap between planned and actual is the planning accuracy signal.
quality_rating → the word they chose, mapped internally to 1–5. Scattered=1, Distracted=2, Solid=3, Focused=4, Flow=5.
focus_score → actual_duration_min × quality_rating. A 72-minute Focused session = 288. A 180-minute Scattered session = 180. The 72-minute session wins. This is the core of why quality beats quantity in this app.
block_id → which time block from last night's plan this session belongs to. This links planning to execution, which is what makes the weekly report meaningful.
interrupted → did they stop early unexpectedly?
interruption_type → if yes: "Something came up" or "Got distracted." These are never treated the same. A friend visiting is not a distraction. A phone spiral is. The user labels it. The app never assumes.
interruption_note → optional free text. "Friend came over." "Power cut." "Felt sick." Stored verbatim. Never judged.
context_location → optional. Home / Library / Café / Other. After 30 days: "Your library sessions average quality 4.1 versus home sessions 2.9."

Notifications during the day — block tick system
At the end of each scheduled block, a notification arrives.
"9:30 — GraphQL block is up. Done?"
Three options directly in the notification, no app open needed: Done ✓ / Need more time / Skip it
If Done → progress bar moves. Block marked complete. Goal progress updates.
If Need more time → block extends. Overrun tracked. After 10 days: "You consistently underestimate GraphQL-type tasks by 18 minutes."
If Skip it → block marked skipped. End of day, if 2+ blocks were skipped, app asks: "A few blocks got skipped today. Want to carry them to tomorrow?"
10 minutes before each block ends, a heads-up: "10 minutes left in GraphQL. On track?" User can say On track or Will need more time. The pre-signal turns out to be more predictive than the post-signal — if someone says "will need more time" at minute 20 of a 30-minute block, that block almost always runs over. We track this.

Afternoon — optional midday check
One notification, optimally timed by Thompson Sampling (explained in technical section). One question.
"Quick check — how's your energy right now?"
Five words again: Drained / Low / Okay / Good / Sharp.
This builds the intraday energy arc. Morning energy + midday energy + evening energy = three data points that, over weeks, reveal the user's personal energy pattern. After 21 days: "Your energy peaks between 9 and 11 AM and drops sharply between 1 and 3 PM. Your best sessions start before 11."

Evening — the daily mirror
Around 9–10 PM. The most important screen of the day. The user sees a card showing what their day actually looked like in data: when they woke, how close to their intention, how many blocks they completed, total focused time, any interruptions they labeled.
They look at this data. Then two questions.
"How productive did you feel today?" — five words: Wasted / Low / Average / Good / Locked in
"How content are you with how you spent today?" — five words: Regretful / Uneasy / Okay / Satisfied / Great
Why two ratings and not one. A student can complete all their study blocks (productive = 5) and still feel hollow because they missed a friend's birthday (content = 2). That divergence IS the insight. "High output, low satisfaction" is a real and important pattern that no other app measures. When this gap appears consistently — three or more weeks in a row — the weekly report names it directly and asks the user to sit with it.
The word "regret" never appears in the app. We use "contentment." The insight can be about what they might regret. The language never is.
Optional: "What got in the way today?" Free text, skippable. "YouTube after lunch." "Got tired at 3 PM." Stored as-is.
Optional: "You had about 2 hours with your phone off today. What were you doing?" Free text. "Watching TV." "Playing cricket." "At the hospital with dad." "Reading." This is the phone-free label. Their words. We categorise it silently using Haiku — outdoor activity, screen-adjacent (TV/laptop counts as screen time), social/in-person, rest. But we always show their words, never our category.

Night — declaring sleep
When they're ready to sleep, one tap: "Going to sleep." This starts a passive timer. On Android, we check if any guarded apps are opened after this declaration. On iOS, we track screen interaction proxy. The data is only shown in the weekly report — never as same-night judgment. "You spent 22 minutes on YouTube after declaring sleep on 4 nights this week. On those nights your morning wake delta was 34 minutes late on average."

PART 3 — TECHNICAL SOLUTION

The architecture principle
Every raw event goes into the events table. Every night at 11:59 PM, a background job reads all events for that day and produces one clean row in daily_scores. One row per user per day. Twenty-something columns. Every algorithm reads only from daily_scores. Never from raw events. This means the entire analytics and ML layer is fast, testable, and explainable. If a weekly report insight looks wrong, you trace it to one column in one row.

EWMA — Exponentially Weighted Moving Average
What it is, plainly: A rolling average that gives more weight to recent days and less to older ones. Your baseline from 3 months ago matters less than your baseline from last week.
The formula: Today's baseline = (0.1 × today's score) + (0.9 × yesterday's baseline)
The 0.1 and 0.9 are the smoothing parameters. λ = 0.1 gives approximately 19 days of effective memory — recent enough to reflect genuine change, stable enough not to panic on one bad day.
What we apply it to: me_score, focus_score, health_index, contentment_rating — one EWMA per metric per user, updated every night.
What it produces: The user_baselines table. One row per user per metric. This is the "current you" reference. Every other calculation compares against this.
Why we need it: Without a rolling baseline, a single terrible week makes a recovered user look permanently broken. With EWMA, a bad week shifts the baseline only slightly. Recovery is visible quickly and naturally.
Is it ML? No. It is a statistical smoothing formula. There is no training, no model, no parameters learned from data. It is a weighted average with a fixed decay rate. Be precise about this — your friend is right to distinguish it.
CTO question: why 0.1 and not 0.2?

λ = 0.1 gives 19 days of memory. λ = 0.2 gives 9 days. At 9 days, one bad week swings the baseline so much that comparisons lose meaning. At 19 days, the baseline is stable enough to be a reliable reference while still updating within 3 weeks when genuine behaviour changes. Source: Hunter (1986), Journal of Quality Technology — this exact λ choice for 2–3 week behaviour cycle tracking.

Pearson Correlation with 1-day lag
What it is, plainly: A number between −1 and +1 that tells you how strongly two signals move together. If sleep hours yesterday and focus score today always move in the same direction — both up together, both down together — the correlation is close to 1. If they move randomly relative to each other, the correlation is close to 0.
The 1-day lag: We deliberately look at yesterday's sleep against today's focus. Not the same day. This captures the causal direction — sleep affects the next day's performance. This is Granger causality applied simply.
What pairs we test: Sleep hours (yesterday) vs focus score (today). Morning phone-free time vs session quality. Snooze count vs contentment rating. Mindless screen ratio vs productiveness rating. Boredom opens vs energy next morning.
When we show it: Three guards must all pass. N ≥ 21 data points (from Cohen 1988 power analysis — minimum sample for reliable correlation). |r| > 0.5 (moderate to strong effect — weak correlations are not surfaced). p < 0.05 (less than 5% probability the pattern is noise). All three must pass. If any fails, nothing is shown.
Is it ML? No. Pearson correlation is a statistical calculation. No training, no model. It computes a coefficient from two arrays of numbers. Statistical indicator.
The insight this produces — real examples:
"Your data from the last 6 weeks shows a pattern: on days that follow a night of less than 6 hours of sleep, your focus score is 61% lower than your baseline. This has been consistent. It appeared on 14 of the 18 low-sleep nights in your history."
"Your boredom phone opens cluster almost exclusively between 10 PM and midnight. On nights with 3 or more boredom opens after 10 PM, your next morning's first session starts 47 minutes later than your average. The pattern is consistent enough to be personal, not coincidence."
"Days you planned your schedule the night before had an average block completion rate of 2.8 out of 3.4 planned blocks. Days you planned the morning of: 1.6 out of 3.1. Planning ahead does not guarantee success — but in your data, it correlates strongly with it."

Thompson Sampling — the actual ML
What it is, plainly: Imagine trying to find the best time to knock on your friend's door. You try 7 AM, 8 AM, 9 AM, 10 AM over several days. You notice they almost always answer at 8 AM. Thompson Sampling is the algorithm that does this automatically — it explores different notification times, tracks responses, and gradually focuses on what works for each individual user.
The technical mechanism: For each notification time slot (6 AM, 7 AM, 8 AM, 9 AM, 10 AM for morning notifications), we maintain two counts: α (times user responded within 30 minutes) and β (times they did not). Each time we need to decide when to send, we sample a random value from Beta(α+1, β+1) for each slot and send at the slot with the highest sampled value. This naturally balances exploitation (sending at the known best time) with exploration (occasionally trying other times in case the user's schedule changed).
Why this matters for retention: A notification that arrives when the user is already distracted or asleep gets ignored, which breaks the habit loop. A notification that arrives at the exact right moment for this specific user gets tapped. Over 2 weeks the algorithm learns the difference. Duolingo published research showing 2–3× higher daily engagement from personalised notification timing vs fixed schedules.
Is it ML? Yes. This is genuine Bayesian reinforcement learning. The model updates from observed outcomes. It learns.

Claude Haiku — the LLM call
When it runs: Twice per day maximum per user. Once for goal decomposition when the user submits their nightly plan. Once in a batch job at midnight to categorise phone-free activity labels and blocker text.
What it does for goal decomposition: Receives the user's raw goal text and their declared time slots. Returns a structured JSON of time blocks — which goal chunk goes into which time window, with estimated durations. The user edits this freely. Haiku's job is to make a reasonable first pass, not to be correct.
What it does for label categorisation: Reads free text entries like "playing cricket with friends" or "watching Netflix on laptop" and assigns a category (outdoor activity / screen-adjacent / social / rest / other). This categorisation is used only for the weekly chart. The user's own words are always shown alongside it.
Cost: Approximately ₹0.01 per plan generation. ₹0.001 per label categorisation. For 10,000 active users, roughly ₹100 per day total. Negligible.
What it does NOT do: Generate insights. Write report text. Make recommendations. All of those are deterministic arithmetic. The LLM is only used where understanding free natural language text is genuinely required.

Everything else — deterministic, no ML
Streaks: Simple counts. N consecutive days above a threshold. No model. Just arithmetic.
Rule-based triggers:

sleep < 6 hours for 3 days running → article on sleep debt
snooze count > 3 for 5+ days → article on sleep inertia
morning phone-free < 5 minutes for 5+ days → article on morning dopamine
productiveness high, contentment low for 3+ days → divergence insight surfaced
goal dropped at same completion percentage 3+ times → planning insight surfaced

All of these are if-then rules applied to daily_scores. No training required. Accurate from day 1.
Trend lines: Weekly average this week vs 4-week rolling average. Direction defined as: above 10% = improving, below −10% = declining, between = stable. Pure arithmetic shown as up/flat/down arrows.
CUSUM — drift detection:

One statistical indicator worth naming specifically. It catches sustained decline — not one bad day, but a genuine downward drift that a daily check would miss. Think of a weight scale. One bad day — 200g heavier — not a concern. But if the scale goes up 200g every day for 10 days straight, something real is happening. CUSUM keeps a running sum of how far below baseline the user is. When that sum crosses a threshold, we know the drift is real and sustained. Source: Page (1954), used in hospital patient deterioration monitoring. Same logic applied to behavioural data.
K-means clustering — the Personal Operating Manual (day 90+):

This is actual ML. Take the user's top 25% of days by Me Score. Cluster their conditions — sleep, energy, session start time, screen use. The centroid of the best-day cluster is the Personal Operating Manual: "Your best days happen when you sleep 7–7.5 hours, start your first session before 9:30 AM, and keep phone-free morning time above 20 minutes. These conditions were present on 87% of your top days." This unlocks at day 90 as a premium feature. The premium paywall is not arbitrary — you need 90 days of data for the clustering to be statistically meaningful.

PART 4 — INSIGHTS THAT ACTUALLY MATTER
The difference between this app and everything else is not the data collected. It is the meaning we attach to it. Here are real examples of insights that would genuinely stop a user mid-scroll:
Week 3:

"Your best focus sessions this week all started before 9:30 AM. Your worst started after 11. This isn't about discipline — your data suggests you have a natural productivity window and you're missing it on 4 of 7 days."
Week 5:

"You've completed your morning goal 11 out of 14 days when you planned it the night before. You've completed it 3 out of 9 days when you planned it the morning of. Planning ahead is the single strongest predictor of your goal completion — stronger than sleep, stronger than energy level."
Week 6:

"Three nights this week you used YouTube for 20+ minutes after declaring sleep. On those three nights your actual wake time was 38 minutes later than your intention. On the other four nights it was 7 minutes later. The pattern is specific to you."
Week 8:

"Your productiveness rating has averaged 4.1 over the last month. Your contentment rating has averaged 2.4. That 1.7 point gap has appeared in 6 of the last 8 weeks. You are producing a lot. You are not satisfied by it. That's worth sitting with."
Month 3 — Personal Operating Manual:

"We've been watching your data for 90 days. Your best days — the top 25% — have three things in common that your other days don't. You slept between 7 and 7.5 hours. You started your first session before 9:15 AM. You had at least 20 minutes off your phone after waking. All three conditions were present on 91% of your best days and only 12% of your below-average days. This is your operating manual."

PART 5 — THE UNIQUENESS
Every app in this space does one thing well. Oura knows your body. Duolingo knows your learning. Todoist knows your tasks. Screen Time knows your phone use. None of them connect the dots.
The uniqueness of Me vs Me is not any individual feature. It is the vertical integration across all of them — and the personal comparison anchor.
Nobody else uses intention as a data type. Every other app measures what happened. We measure what you planned versus what happened. That gap — the intent-execution delta — is where the real behavioural signal lives.
Nobody else uses your personal best as the benchmark. Every other app either measures against yesterday (not meaningful enough) or against population averages (wrong comparison entirely). We measure against the best version of you that actually existed. You already know you can do it. You did it in September. The question is what conditions made that possible.
Nobody else connects health to career to screen to real-world time in one score. The productivity apps do not know you slept badly. The health apps do not know you had a high-output day. We know both, and we show the connection: "On days your health index was below 60, your focus score was 40% lower. You do not have a discipline problem. You have a recovery problem."
The data moat. After 90 days, a user's Personal Operating Manual is irreplaceable. It is built from their specific history. It cannot be recreated by switching apps. The switching cost is losing your own self-knowledge. That is the stickiest possible retention mechanism — not gamification, not streaks, not social features. Just your own history of yourself.

PART 6 — PPT FLOW
Slide 1 — Problem statement + validation
The problem: you wake with intention and end the day feeling like it slipped away. Not because you're lazy. Because nobody has ever shown you a clear picture of yourself. Validation: search Reddit threads about feeling unproductive despite trying. App Store reviews of Notion, Todoist, Habitica — what users say is missing. The gap is not another tracker. The gap is a mirror.
Slide 2 — Onboarding: who are you?
Student vs working. The two paths. Show how one question changes the entire experience. Diagram of the two flows.
Slide 3 — Onboarding: the calibration
Best self vs current self. The sliders. The free-text "best month" question. What we extract. The synthetic Me Score shown at the end. Make this visual and emotional — not technical.
Slide 4 — The daily flow: morning
The intention-setting night before. The warm wake-up notification at 6 AM instead of an alarm. The wake delta calculation. Show the actual notification and the actual data it creates.
Slide 5 — The daily flow: through the day
Focus session tap. The quality word picker (not numbers). The block tick notification. The evening mirror — productiveness and contentment ratings and why both. Keep this one slide, show the flow as a simple timeline.
Slide 6 — Technical solution: the data architecture
Raw events → daily_scores row. One clean row per user per night. All algorithms read from here. Show this as a simple flow diagram. Nothing complex.
Slide 7 — Technical solution: the analytics
Two columns. Left: deterministic (EWMA, Pearson correlation, CUSUM, rule-based triggers, trend lines). Right: actual ML (Thompson Sampling, K-means at day 90). Be explicit about which is which. The honesty impresses CTOs.
Slide 8 — Technical solution: real insights
Show 3 actual insight examples from Part 4 above. Not feature descriptions. Actual sentences the app would say. This is the slide that makes people feel the product. "Days you planned the night before had 40% higher goal completion" is more compelling than "we use Pearson correlation."
Slide 9 — MVP scope
What V1 actually contains. Student and working user. Health domain and career domain. Wake intention, daily plan, focus sessions, block tick notifications, evening check-in. Weekly report with 8 sections. No app blocking, no wealth, no AI character yet. Two platforms but Android-first for screen features. Build time: 8–10 weeks for two developers.
Slide 10 — Future scope
Wealth domain (impulse tracking, savings behaviour). AI companion character with tone selection. Hard app blocking — Android first using UsageStatsManager, iOS via Screen Time API with entitlement. Personal Operating Manual at day 90. B2B channel — medical colleges, coding bootcamps. Anonymised cohort analytics for institutions.

That is everything. Read Part 3 twice before the CTO meeting. The EWMA and Pearson sections especially — know those cold. The one thing to memorise is the distinction your friend pointed out: only Thompson Sampling and the LLM call are actual ML. Everything else is statistical indicators. Say that clearly and confidently if asked. It shows you understand what you built.