# AI Live Cricket Commentary Engine — Discussion Notes

## Product idea
An engine that ingests a live cricket match, holds full context of the game, and generates expert-level commentary in near real time — output as both text and voice (TTS). Long-term intent: if the core engine is solid, the same architecture should generalize to other live sports or any live structured-event stream (not cricket-specific).

## Scope decisions made so far
- Stage: brainstorming/architecture discussion, no code or doc built yet.
- Data source: engine consumes a **ball-by-ball structured feed** (scores, wickets, overs, extras, player IDs) — not raw video/audio analysis. Video-based event detection was flagged as a possible future phase, not the starting point.
- Output: both **text commentary** (running feed) and **voice/audio commentary** (TTS with prosody matching excitement level).
- Reusability: architecture should be generic — cricket-specific parts should be isolated to the event taxonomy and domain knowledge, so the same engine core could later drive other sports/domains.

## Engine architecture (as discussed)

### 1. Ingestion & normalization
Raw feed (format varies by provider) → normalizer → canonical event schema: match_id, innings, over/ball, event_type (runs/wicket/wide/no-ball/bye/milestone), batter, bowler, runs, extras, wicket details, timestamp. Sits behind a queue/event bus to absorb feed lag, hiccups, and replay — keeps ingestion instability isolated from the rest of the pipeline.

### 2. State store
Two layers:
- **Static/pre-match context**: player career stats, recent form, head-to-head, venue records, pitch report, weather, team news.
- **Dynamic/live state**: score, wickets, current partnership, run rate vs. required run rate, current bowler's spell figures, current batter's strike rate/balls faced, phase of innings (powerplay/middle/death), proximity to milestones.

This state is what turns a raw event into something meaningful (a dot ball in over 3 vs. over 49 of a chase means something different).

### 3. Significance scoring (the core "expert feel" component)
Rule-based (not ML) to start — explainable, tunable, avoids a black box misfiring at the wrong moment. Each event scored using match context, not just event type:
- Wicket significance varies by match situation (death-over wicket in a tight chase >> wicket when one side is already dominant).
- Milestones (50s, 100s, 5-wicket hauls) always trigger regardless of scoreline.
- Momentum tracked via rolling comparisons (last 3 overs run rate vs. last 10, dot-ball streaks signaling building pressure).
- Close calls/DRS if the feed exposes them.

The score drives two things: how much "airtime"/length the commentary gets, and what extra context gets pulled into the prompt (a wicket pulls dismissal pattern, a boundary pulls scoring zones).

### 4. Tiered generation
Can't run a full LLM call on every ~500-600 balls in a match and stay live. Proposed tiers:
- **Tier 0** — routine dot balls/singles: cheap/templated text, minimal generation cost.
- **Tier 1** — boundaries, moderate events: fast LLM call, light context.
- **Tier 2** — wickets, milestones, last-over drama: full LLM call with richer injected context (stats, historical parallels, tactical read).

Controls both cost and latency.

### 5. Narrative memory
Short-term memory to avoid repeating itself within an over, plus a "storyline tracker" for threads spanning many balls (required rate creeping up, a bowler probing a weakness, a partnership building) that need periodic reinforcement rather than one mention and silence.

### 6. Grounding / anti-hallucination
Model should only state facts present in the state object or static context — never invent a stat. Flagged need for a post-generation check that any numbers mentioned match actual state, since a wrong score/milestone breaks trust immediately.

### 7. Latency
Feed provider lag is usually the dominant delay (cheap feeds ~5-15s behind live play, premium feeds near real-time). The engine's own pipeline (state update → scoring → generation → TTS) needs to add minimal overhead on top of that to keep voice commentary paced with actual play.

## Open threads / not yet discussed in depth
- Significance scoring rules in detail (exact scoring logic, thresholds, how context modifiers combine).
- Tiered generation + prompt design specifics (what exactly goes into each tier's prompt, persona design).
- Data/feed licensing and sourcing (ball-by-ball feeds and commentary rights are typically licensed by broadcasters/boards — flagged as a likely business constraint, not yet explored).
- Voice/TTS implementation specifics (which engine, streaming approach, prosody control mechanics).
- Persona design (tone, avoiding impersonation of real named commentators).

## Suggested next step
Pick up with either: (a) detailed significance-scoring rules and thresholds, or (b) tiered generation/prompt design — these were the two threads offered as "what to dig into next" when this summary was requested.
