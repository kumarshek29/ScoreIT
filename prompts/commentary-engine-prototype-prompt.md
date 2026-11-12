# Build Prompt: AI Cricket Commentary Engine — Video-Input Prototype

Copy everything below this line into Claude Code / Cursor / your agent of choice.

---

## Mission

Build a working prototype of an AI cricket commentary engine. Input: a **video file of a full cricket match recording** (local .mp4/.mkv). Output: **live-style text commentary** and **spoken commentary (TTS)** synced to the video timeline, as if the engine were commentating the match live.

I am validating the core thesis: given a stream of match events + full game context, can an AI produce commentary that feels expert, timely, and non-repetitive? Optimize for proving that, not for production polish.

## Critical architecture decision — how to get events from video

Do NOT attempt ball-tracking or player-detection computer vision. Instead, exploit the fact that every broadcast has an on-screen **score bug/overlay** (score, wickets, overs, batter names, bowler figures). The extraction pipeline:

1. **Frame sampling**: Use ffmpeg to extract frames every ~2 seconds from the video.
2. **Score-bug reading**: Send the cropped score-bug region of each frame to a vision LLM (Claude API) to read: total runs, wickets, overs.balls, striker + score, non-striker + score, bowler + figures, target/required rate if shown. Return strict JSON. Include a one-time calibration step: show me 3 sample frames, let me confirm/adjust the crop region (score bug position varies by broadcaster).
3. **Event derivation by diffing**: Compare consecutive readings. Score +4 with same wickets → boundary. Wickets +1 → wicket (batter who disappeared = dismissed). Overs advanced with +0 → dot ball. Etc. Emit canonical events.
4. **Noise handling**: OCR/vision reads will occasionally be wrong or the bug will be off-screen (replays, ads). Reject readings that are impossible given current state (score going down, overs jumping). Use a small state machine that only accepts monotonically consistent transitions; skip/interpolate gaps.

This gives ball-by-ball events from any recording without a licensed data feed. Cache all vision-call results to disk keyed by frame timestamp so re-runs are free.

## Canonical event schema

```json
{
  "match_id": "str", "innings": 1, "over": 12, "ball": 4,
  "event_type": "runs|wicket|wide|no_ball|bye|leg_bye|milestone|end_of_over",
  "batter": "str", "bowler": "str", "runs": 0,
  "wicket": {"dismissed": "str", "how": "str|null"},
  "score_after": {"runs": 87, "wickets": 3},
  "video_ts": 4512.0
}
```

## Engine pipeline (runs on the derived event stream)

### 1. State store
- **Dynamic state** (updated per event): score, wickets, current partnership (runs/balls), run rate, required rate + balls remaining (if chase), each batter's runs/balls/SR, bowler's spell figures, innings phase (powerplay/middle/death), dot-ball streak, last-3-overs vs last-10-overs run rate, distance to next milestone (individual 50/100, team 50s).
- **Static context**: a hand-written `match_context.json` I'll fill in for the test match — teams, venue, tournament stakes, 2–3 line bio/form note per key player, head-to-head notes. The engine must load and use this; do not invent player facts.

### 2. Significance scoring (rule-based, 0–100)
Score each event from event type + context modifiers. Examples of required behavior:
- Wicket base 60; +20 if partnership was 50+; +15 if in death overs of a close chase; −20 if match already one-sided.
- Boundary base 30; +10 if it breaks a 10+ dot-ball drought; +10 if it brings required rate under a run a ball.
- Milestones (individual 50/100, team milestone, 5-for) always ≥ 80.
- Dot balls base 5; escalate with consecutive-dot pressure late in a chase.
Make weights a config dict, log every event's score + reasons (explainability matters — I will tune these).

### 3. Tiered generation
- **Tier 0 (score < 25)**: templated text, no LLM call. Maintain ~15 varied templates per event type; never use the same template twice in a row.
- **Tier 1 (25–60)**: fast/cheap LLM call (Claude Haiku), compact prompt: event + one-line state summary + last 2 commentary lines.
- **Tier 2 (> 60)**: full LLM call (Claude Sonnet), rich prompt: event, full dynamic state, relevant static context (pull dismissal/bowler bio on wickets, batter form on milestones), active storylines, last 5 commentary lines.

### 4. Narrative memory
- Rolling buffer of last N commentary lines, injected into prompts with instruction: do not repeat phrasing or observations.
- **Storyline tracker**: maintain 2–4 active threads (e.g., "required rate climbing", "bowler working over this batter", "partnership rebuilding after collapse") as short strings updated per over; Tier 2 prompts must reference at least one active thread when relevant.

### 5. Grounding check
Post-generation: extract every number from generated commentary and verify it appears in the state object (± allow spelled-out numbers). On mismatch, regenerate once with a correction note; if it fails again, fall back to a template. Never ship a wrong stat.

### 6. Commentary persona
One consistent persona: knowledgeable, warm, builds tension, economical with words on routine balls, expansive on big moments. Do NOT imitate any real named commentator. Define the persona in one system prompt shared across tiers.

### 7. TTS + sync
- TTS per commentary line. Use a streaming-capable TTS API (or `edge-tts` for a free start). Map significance to delivery: pass excitement level into voice settings/SSML (rate, pitch) — 3 levels is enough.
- **Playback mode**: a simple local web page (single HTML file) with: the video element playing the recording, the text commentary feed appearing at each event's `video_ts`, and audio commentary playing in sync (duck/skip if a new high-significance event arrives before the previous audio finishes — never overlap two voice clips).
- Processing can be offline-first: run the whole pipeline on the video, produce a `commentary_track.json` (+ audio files), then the web page just replays it in sync. Add a `--simulate-live` flag later that processes with only past-frames visible.

## Tech stack
Python 3.11+, ffmpeg, Claude API (vision + generation), edge-tts or ElevenLabs, plain HTML/JS for the player. SQLite or JSON files for caching. No frameworks unless necessary.

## Milestones (build and show me each before proceeding)
1. **M1**: Frame sampler + score-bug reader working on my video; show me 10 consecutive readings as JSON.
2. **M2**: Event derivation — full first innings as a canonical event list; I'll spot-check against the actual scorecard.
3. **M3**: State store + significance scoring; show per-event scores with reasons for one full innings.
4. **M4**: Tiered text generation with memory + grounding check; show me a 5-over stretch including a wicket.
5. **M5**: TTS with excitement mapping.
6. **M6**: Synced playback page.

## Success criteria
- Event extraction ≥ 90% accurate vs the real scorecard on ball outcomes.
- Zero factually wrong stats in generated commentary (grounding check works).
- A wicket in a tense chase reads/sounds noticeably bigger than a mid-innings dot ball.
- No repeated phrasing within any 2-over window.
- Cost: a full T20 innings processes for under a few dollars of API spend (report actuals).

## Notes
- Ask me for the video file path and the match's real scorecard URL (for validation) before starting.
- Keep cricket-specific logic (event taxonomy, scoring rules, phase definitions) in isolated modules — the core pipeline should be sport-agnostic.
