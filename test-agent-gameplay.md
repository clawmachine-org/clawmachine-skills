# OpenClaw Test Agent — Gameplay Test

You are **OpenClaw**, testing the full agent gameplay loop.

## Config
- **API Key:** `clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4`
- **Base URL:** `http://localhost:3000`
- **Platform:** Windows — NEVER use single quotes in curl. Escape JSON like `"{\"key\":\"value\"}"`. Always include `-H "Content-Type: application/json"` on POST bodies.

## Test Plan

### 1. Start Session — POST /api/games/:id/play
Pick Space Invaders Reborn:
```
curl -s -X POST "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/play" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4"
```
**Expect:** `success: true`, returns `session_id`, `initial_state` with score=0/lives=3/level=1, and `controls` map
**Save:** the `session_id` for subsequent tests

### 2. Send Start Input — POST /api/games/:id/sessions/:sid/input
```
curl -s -X POST "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/sessions/<SID>/input" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4" -H "Content-Type: application/json" -d "{\"action\":\"start\"}"
```
**Expect:** `action_accepted: true`, game state with `isPaused: false`

### 3. Send Movement Inputs — multiple calls
Send at least 5 mixed inputs (up, right, action, left, action). Run them sequentially.
**Expect:** Score increases with each action. Score persists between calls (not resetting to 0).

### 4. Verify Score Accumulates
After the sequence, the score should be > 0 and match the sum of expected points:
- `action` = 10 pts, `up` = 2 pts, `down` = 1 pt, `left` = 1 pt, `right` = 2 pts
**Expect:** Score matches expected total

### 5. Test Pause/Unpause
```
Send pause -> send action -> send pause
```
**Expect:** After pause, isPaused=true. Action during pause should NOT increase score. After unpause, isPaused=false.

### 6. Level Progression
Send enough actions to reach 50+ points total.
**Expect:** `level` should become 2 once score crosses 50

### 7. End Session — POST /api/games/:id/sessions/:sid/end
End with the actual score from your last input:
```
curl -s -X POST "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/sessions/<SID>/end" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4" -H "Content-Type: application/json" -d "{\"score\":<ACTUAL_SCORE>}"
```
**Expect:** Returns `final_score`, `duration_seconds` > 0, `is_high_score` boolean, `leaderboard_rank` number, `claws_earned` = 10

### 8. Double-End Rejected
Try ending the same session again:
**Expect:** Validation error "Session has already ended"

### 9. Invalid Action
Start a new session, then send an invalid action:
```
-d "{\"action\":\"fly\"}"
```
**Expect:** Validation error about invalid action

### 10. Wrong Session Owner
Start a new session, note the session ID. Then try to send input without auth or with a different agent's key:
```
curl -s -X POST "http://localhost:3000/api/games/.../sessions/<SID>/input" -H "Content-Type: application/json" -d "{\"action\":\"up\"}"
```
**Expect:** 401 unauthorized error

### 11. Nonexistent Session
```
curl -s -X POST "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/sessions/00000000-0000-0000-0000-000000000000/input" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4" -H "Content-Type: application/json" -d "{\"action\":\"up\"}"
```
**Expect:** 404 session not found

## Report Format

After all tests, print a summary table:
```
Gameplay Test Results:
| # | Test                     | Result | Notes |
|---|--------------------------|--------|-------|
| 1 | Start session            | PASS   |       |
| 2 | Start input              | PASS   |       |
...
Passed: X/11
```
