# Me vs Me — CTO Pitch Q&A

Questions grouped by attack surface. The twisted ones are marked ⚡.

---

## Architecture

**"Why batch at 11:59 PM? What if a user opens the app at 10 PM and wants their status?"**

The app does not show Me Score daily — only weekly. What the home screen shows daily is directional: "Above rhythm / On rhythm / Below rhythm." That is computed from the most recent `daily_scores` row, which may be yesterday's. A user opening at 10 PM sees yesterday's direction. The full score only generates on report day. The batch is not blocking the user experience — it is running the analytics for a weekly output.

---

**"What happens if the nightly batch fails?"**

Retry with exponential backoff — 3 attempts before the job is flagged as failed. If it fails completely, the next night's batch processes both days in sequence. EWMA handles a one-day gap cleanly — the formula is just applied twice. CUSUM consecutive-day counter pauses rather than resets. The only user-visible impact: if report day is the failed night, report generation delays by 24 hours and the user gets a notification: "Your report is being prepared."

---

**⚡ "How do you handle timezones? Your batch runs at 11:59 PM — whose 11:59 PM?"**

The user's local 11:59 PM. All events are stored in UTC with the user's timezone offset recorded at signup. The batch job scheduler reads the user's timezone and triggers per-user, not per-server. This is non-trivial engineering — batch workers need to fan out by timezone group. For MVP 1 with 50 users this is fine. At scale you bucket timezones and run batches in waves. This is a known solved problem in any analytics pipeline.

---

**"What database?"**

PostgreSQL for everything relational: `daily_scores`, `user_baselines`, `weekly_reports`. `daily_scores` is simple enough — one row per user per day, 20 columns, no joins needed for the analytics layer. Redis for notification scheduling (lightweight, TTL-based). No time-series database needed in MVP 1 — Postgres handles this volume easily. We would revisit at 100K+ users.

---

**"What if the Health API goes down or the user denies permissions?"**

Two layers. First, if permissions are denied in onboarding, we surface a manual entry card in the evening check-in: sleep hours and steps as a number or subjective tag. This is stored identically to API data — no separate codepath. Second, if the API goes down mid-product: the most recent cached value from the last successful sync is carried forward for up to 2 days, flagged as `estimated`. After 2 days the field is null and health_index is computed from sleep only until sync recovers. CUSUM ignores health_index on null days.

---

## EWMA and Baseline

**"Why λ=0.1 specifically? Why not 0.2 or 0.05?"**

λ=0.1 gives approximately 19 days of effective memory. That maps to Phillippa Lally's 66-day habit research — the first 21 days is when a pattern is still forming. λ=0.2 would mean the baseline reacts strongly to a single good or bad week, making it too volatile for a comparison baseline. λ=0.05 would mean the baseline barely moves — a user who genuinely improves over 6 months would still be compared to their starting point. 0.1 is the standard convention in behavioral EWMA tracking. If testing shows users feel it too slow or too fast to adapt, we can expose this as a sensitivity setting in advanced settings.

---

**⚡ "Your EWMA adapts downward when a user has a bad month. So after a terrible month, their bad days start looking like wins. The app congratulates them for being mediocre."**

This is the self-reference problem with any adaptive baseline, and it is real. Two mitigations. First, CUSUM is specifically designed to catch this: it compares against the historical σ, not the current EWMA. If the user's focus drops consistently for 7 days, CUSUM flags it as a sustained decline before the EWMA has adapted enough to normalise it. The alert surfaces a check-in prompt — not a congratulation. Second, personal best never resets. The weekly report always shows "X% of your personal best." Even if the rhythm baseline drifts down, the personal best ceiling stays fixed. The user can always see the distance between where they are and their highest recorded output.

---

**⚡ "What if someone inflates their onboarding sliders to get a favorable EWMA seed — low baseline, easy to beat?"**

If they inflate their best slider, the seed is high. The first few weeks will consistently show "below rhythm." That is a punishing start that will self-correct — no one who wants to feel good about the app would keep an inflated seed. If they deflate the seed to make it easy to beat: the rhythm baseline is low, every day looks like a win, and the weekly report says "90 of personal best" with a personal best of 40. The score is meaningless to them. Either way, by day 21 the seed has zero weight — real logged data owns the baseline completely. Gaming has a 21-day shelf life and the main person it affects is the user.

---

**⚡ "What if someone just does not log bad days — they only check in when they had a good day? Their EWMA baseline inflates artificially."**

It does. We cannot force logging. What we can do: `days_logged` only counts completed check-ins, and the weekly report requires a minimum of 4 logged days out of 7. If a user only logs good days, they get fewer reports and the character does not unlock on schedule. The product creates natural incentive to log consistently rather than selectively. We also plan to add a line in week 3 reports: "Your baseline is built from 21 logged days. Logging on harder days makes your rhythm more accurate." Transparency over enforcement.

---

## Pearson and CUSUM

**"p < 0.05 threshold for Pearson — that is standard academic criticism territory, especially with N=21."**

We use p < 0.05 AND |r| > 0.5 AND N ≥ 21. The |r| > 0.5 guard is the conservative one. At N=21, a p=0.05 can occur with |r| ≈ 0.43 — too weak to surface as an insight. By requiring |r| > 0.5, we are demanding a practically meaningful effect size, not just statistical significance. We also do not publish statistical language to users. We say "your focus tends to be higher on days after good sleep" — not "r=0.61, p=0.003." The correlation is a directional nudge, not a scientific claim. If the correlation weakens or reverses at N=60, the insight updates or disappears.

---

**⚡ "CUSUM fires after 7 consecutive days below baseline. But a user just started a new exercise routine and their focus is dropping as they physically adjust. Your app fires a wellbeing check-in for what is actually a deliberate change."**

This is a false positive case and it is a real limitation. Our current design has no way to distinguish voluntary disruption from involuntary decline. What we have: the CUSUM alert copy is written as an observation, not a diagnosis — "Your focus has been below your rhythm for 7 days. Anything going on?" The user can respond "Yes, intentional" or ignore it. We are not blocking them or sending repeated alerts. In v1.1, we could add a "I'm in a planned transition" flag that pauses CUSUM for 14 days. For MVP 1, the alert is soft enough that false positives are not harmful.

---

## K-means and Personal Operating Manual

**"Why K=2? What if someone has three clearly distinct day profiles?"**

K=2 is a product decision, not a statistical one. We are building a feature a user reads in 30 seconds and acts on tomorrow morning. Two profiles are actionable: "your best days look like this, your strong alternate days look like this." Three profiles would require a data science background to interpret. If the silhouette score for K=2 on a particular user's data is weak (below 0.5), we flag it internally but do not surface it differently to the user. In a future version we can expose the optimal K and allow users to see more profiles. For MVP 2, two profiles are the right call for readability.

---

**⚡ "K-means is sensitive to outliers. If someone went on a vacation and had exceptional step counts and zero focus sessions, that outlier day could distort the cluster centroid."**

Top 25% selection by Me Score partially guards this. Vacation days typically have unusual step counts (very high) and low focus session counts — they would produce a low career_domain score and therefore rarely appear in the top quartile. Days tagged as Recovery are excluded from the top-quartile pool entirely. For statistical robustness in v2, we can apply z-score filtering to exclude days where any feature is more than 2σ from the median — those days are structural outliers that would skew the centroid. We acknowledge this as a refinement item.

---

**"What is the minimum N for K-means to produce meaningful clusters?"**

90 days logged AND 15+ top-quartile days. With K=2 and 15 samples, each cluster has an expected size of ~7–8 days. That is thin but workable for computing a centroid across 9 features. Below 15 top-quartile days, the centroids are not stable — small perturbations would reorganise the clusters. We surface nothing. No empty state, no "almost there." The feature simply does not appear until the threshold is met.

---

## Thompson Sampling

**"How many notification interactions before Thompson Sampling actually personalises meaningfully?"**

Each hour slot starts at α=1, β=1 (uniform prior). After one notification sent and ignored, that slot is at α=1, β=2 — it is being down-sampled. After five sends with four responses, a slot is at α=5, β=2 — it is strongly favoured. With one notification per day: by week 2 (14 interactions), the algorithm has enough signal to consistently favour 2–3 hour slots. Full personalisation (confidence interval < 0.1) typically needs 30–40 interactions — about 5 weeks of daily use. The user's Screen 6 preference is the fallback for weeks 1–4. Thompson Sampling ships in v1.1 once users have enough data.

---

**⚡ "You only get one notification per day. If Thompson Sampling is always exploring one new slot per few days, it never fully exploits. Where is the balance?"**

Beta distribution handles this automatically. A slot with α=1, β=0 (tried once, succeeded) draws samples ranging from 0.1 to 0.99 — high uncertainty means it will sometimes be selected but not always. A slot with α=30, β=5 draws tightly around 0.85 — it almost always wins the sampling. Exploration happens naturally when confident slots have similar distributions. You do not need to manually schedule an "exploration day." The bandwidth issue — one notification per day is slow to learn — is real, but 5 weeks to meaningful personalisation is acceptable for a habit product. Users who churn in week 2 do not get to Thompson Sampling anyway.

---

## LLM and Cost

**"You say LLM only in 3 places. What is your actual cost model?"**

Haiku at $0.25 per 1M input tokens / $1.25 per 1M output tokens.

| Use case | Approx tokens | Frequency |
|----------|--------------|-----------|
| Screen 2 extraction | ~500 input, ~100 output | Once per user at signup |
| Goal decomposition (v1.1) | ~300 input, ~150 output | Once per goal, avg 2 goals |
| Post-sleep label (v1.1) | ~100 input, ~50 output | Once per sleep session if ambiguous |

For 10,000 active users signing up: roughly $1.25 in LLM costs for onboarding alone. Goal decomposition and sleep labels are triggered by user actions — they scale with engagement, not with user count. A highly active user might trigger 10 goal decompositions across a month. Cost is negligible at this scale. The concern is not cost — it is accuracy and hallucination risk, which is why LLM is kept out of reports entirely.

---

**⚡ "Haiku is making an inference from a few sentences of open text. How confident are you in extraction accuracy?"**

We are not confident — that is why we built the confidence field. Haiku returns extraction_confidence as a float 0–1 in JSON alongside the extracted values. Below 0.4, a single follow-up question fires. The extracted seed is one of two inputs (alongside explicit slider values). If the extraction is badly wrong, the EWMA seed diverges from reality and the user will immediately notice "below rhythm" on days they know were good. By day 21 the seed has zero weight — real data corrects it. Bad extraction has a 21-day self-healing window. We are also not using the extracted values for any user-facing claim — they are only used internally to seed the EWMA. If the seed is slightly off, the output is a mildly misaligned first 3 weeks. Not a product-breaking error.

---

## Data and Privacy

**"You are storing health data — sleep, steps, focus patterns. What is your data story?"**

All health data stored encrypted at rest (AES-256). Health API access requires explicit user permission — iOS asks for HealthKit access, Android asks for Health Connect. Users can request full data export or deletion at any time (GDPR Article 17). Raw events are purged after 30 days — we retain only `daily_scores`, which is the computed aggregate. We do not retain raw sleep timing data or raw step-by-step sensor readings. `daily_scores` contains sleep hours and step count — not raw sensor streams. We confirm the exact data retention policy with legal before launch. This is a placeholder architecture decision, not a final compliance statement.

---

## Product

**⚡ "Apple Health already shows sleep, steps, and trends. Why would someone use Me vs Me?"**

Apple Health shows you what happened. Me vs Me tells you whether what happened was close to your personal normal. Apple Health has no concept of a rhythm or baseline — it has no EWMA, no CUSUM, no comparison layer. It shows you 7h sleep on a chart. We tell you: "7 hours is 12% above your rhythm — that is why your focus score was stronger this week." The comparison layer is the entire product. Health apps are data displays. We are a self-comparison engine.

---

**⚡ "What if the Me Score feels arbitrary to users? How does someone know if 73 is good or bad?"**

73 is never shown without context. It is always shown as:
- Delta from last week (+6)
- % of personal best (89% of 82)
- Direction narrative ("slightly above your rhythm")

The number is an anchor for the narrative, not the message itself. A user who finds 73 confusing reads "slightly above your rhythm this week, +6 from last week." That is interpretable without understanding the formula. The score feels arbitrary when shown alone. We never show it alone.

---

**⚡ "Your floor is 20. That means a completely disengaged user still gets a 20 and the app says nothing is wrong."**

The floor does not mean nothing is wrong. The floor prevents the score from being used as a punishment number. A user with Me Score 20 for three consecutive weeks hits a CUSUM alert on day 7 of consistent below-baseline performance. CUSUM fires regardless of the score floor — it tracks the raw daily_focus_score and health_index against the EWMA baseline. The wellbeing check-in surfaces. The floor and CUSUM work together: floor stops the score from feeling crushing, CUSUM still catches sustained decline. The 20 is a floor for UX, not a floor for concern.

---

**⚡ "Tone preference — what if someone picks 'Push me' and has a catastrophic week? Are you making it worse?"**

Push me changes framing, not content. "You were below your rhythm 4 of 7 days" and "It was a harder week" are the same data point. We do not fabricate harsher language or amplify negative framing — we just remove softening. More importantly: if CUSUM triggers a wellbeing check-in, all tone modes converge to a supportive message regardless of preference. The wellbeing alert overrides tone preference. "Push me" does not mean "tell me I failed at a vulnerable moment."

---

**⚡ "After day 7 the character unlocks. What keeps the user coming back on day 8?"**

Honest answer: day 8 to week 4 is the hardest window. The character begins giving daily reactions from week 2 — one-line observations based on yesterday's direction ("You were above rhythm. That is what I expected of you."). The primary retention hook is the weekly report — users want to know their Me Score. If the first report on day 7 feels accurate and personal, they come back for week 2. If it does not, they do not. That is the exact thing MVP 1 is testing. If the first report retention rate is below 40%, the report framing is wrong, not the algorithm.

---

**"No social layer. Is that a retention risk?"**

Intentionally. Comparison to others is what this product is designed to remove. A leaderboard would contradict the core value proposition in the first product screen. Our retention mechanic is the data flywheel: the longer you use the app, the more accurate your baseline, the more personalised your operating manual. The product becomes more valuable over time by design. That is the anti-churn mechanism — the opposite of a social layer, which creates retention through external pressure rather than internal insight.

---

**⚡ "What stops someone from always tapping 'Flow' on quality rating to game their career domain score?"**

Nothing stops them in MVP 1. The behavioral quality proxy (v1.1) cross-validates quality_rating against session duration, completion ratio, and interruptions. A user who taps Flow on a 12-minute session with 3 interruptions gets flagged internally — the divergence is stored as data. We never penalise the user or correct their rating publicly. We just know the self-reported quality is unreliable for that user and weight behavioral_quality higher in the career_domain calculation. The main consequence of always tapping Flow: their career_domain inflates, their Me Score looks great, and the app becomes a mirror that tells them what they want to hear. That user will churn eventually — the product stops feeling true.

---

**⚡ "You have no LLM in reports. But template strings filled with numbers will feel robotic. Users will stop reading them."**

This is a deliberate trade-off. LLM-generated report copy introduces hallucination risk — an LLM might write "your focus correlates with sleep" when our Pearson r is 0.38 and we flagged it as below threshold. Template filling guarantees the insight is factually correct. We would rather have a report that is accurate and slightly clinical than fluent and occasionally wrong. The antidote to robotic copy is not LLM — it is better template writing and tone-adjusted copy layers. That is an editorial problem, not an engineering one. In v2, if we add LLM to reports, every LLM-generated sentence is grounded by a structured data object that constrains what it can say. We do not do this in MVP 1.

---

## Wild Cards

**⚡ "Your whole product depends on the user being honest. What is your honesty rate assumption?"**

We assume partial honesty, not perfect honesty. The design accounts for it:
- Quality rating divergence is detectable via behavioral proxy (v1.1)
- Slider gaming self-corrects by day 21
- Selective logging creates sparse reports, which incentivises consistent logging
- Health data is passive (no gaming possible for sleep/steps)

The app does not need the user to be perfectly honest. It needs them to be roughly honest over a sustained period. The EWMA and Pearson are robust to occasional noise — they are sensitive to sustained patterns, not single outlier inputs.

---

**⚡ "What if the Me Score is clinically sensitive — someone with depression sees their score crater, CUSUM fires, the app asks 'what's getting in the way?' Is that appropriate for a consumer app?"**

This is the most important ethical question in the product. Our position: the app is a mirror, not a therapist. The CUSUM wellbeing prompt is worded as observation, not diagnosis: "This has been going on a while. What's getting in the way?" It does not suggest clinical intervention. It does not offer mental health resources unprompted. It creates space for reflection. If a user is in genuine crisis, the appropriate response is not a productivity app — and we are not designed to serve that case. We are designed to reflect patterns, not to counsel. In a future version, if CUSUM triggers for 14+ consecutive days, we can surface a single line: "If you are going through something difficult, these resources might help" with standard signposting. For MVP 1, the prompt is soft and non-clinical by design.
