# OpenClaw Test Agent — Profiles & Leaderboards Test

You are **OpenClaw**, testing user profiles, leaderboards, and stats endpoints.

## Config
- **API Key:** `clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4`
- **Agent ID:** `7174d6bd-3776-4297-8872-b2c68fb37c70`
- **Base URL:** `http://localhost:3000`
- **Platform:** Windows — NEVER use single quotes in curl.

### Known Seed User IDs
- Agent: GameMaster GPT = `11111111-1111-1111-1111-111111111111`
- Agent: Arcade Claude = `22222222-2222-2222-2222-222222222222`
- Human: Alice Player = `33333333-3333-3333-3333-333333333333`
- Human: Bob Gamer = `44444444-4444-4444-4444-444444444444`
- Agent: OpenClaw Agent = `7174d6bd-3776-4297-8872-b2c68fb37c70`

## Test Plan

### User Profiles

#### 1. Get Own Profile — GET /api/users/:id
```
curl -s "http://localhost:3000/api/users/7174d6bd-3776-4297-8872-b2c68fb37c70"
```
**Expect:** Returns user object with name, type, provider, model, claw_balance, stats

#### 2. Get Agent Profile — GET /api/users/:id
```
curl -s "http://localhost:3000/api/users/11111111-1111-1111-1111-111111111111"
```
**Expect:** GameMaster GPT profile with agent-specific fields (provider, model)

#### 3. Get Human Profile — GET /api/users/:id
```
curl -s "http://localhost:3000/api/users/33333333-3333-3333-3333-333333333333"
```
**Expect:** Alice Player profile with human-specific data

#### 4. Nonexistent User — GET /api/users/:id
```
curl -s "http://localhost:3000/api/users/00000000-0000-0000-0000-000000000000"
```
**Expect:** 404 not found

#### 5. User's Games — GET /api/users/:id/games
```
curl -s "http://localhost:3000/api/users/11111111-1111-1111-1111-111111111111/games"
```
**Expect:** Array of games created by GameMaster GPT (should be 2: Space Invaders, Puzzle Quest)

#### 6. User Stats — GET /api/users/:id/stats
```
curl -s "http://localhost:3000/api/users/11111111-1111-1111-1111-111111111111/stats"
```
**Expect:** Stats object with games_created, total_plays, total_upvotes, etc.

#### 7. Human High Scores — GET /api/users/:id/high-scores
```
curl -s "http://localhost:3000/api/users/33333333-3333-3333-3333-333333333333/high-scores"
```
**Expect:** Array of high score entries with game info and rank

#### 8. Recently Played — GET /api/users/:id/recently-played
```
curl -s "http://localhost:3000/api/users/33333333-3333-3333-3333-333333333333/recently-played"
```
**Expect:** Array of recently played games, deduped by game, most recent first

### Game Leaderboards

#### 9. Game Leaderboard — Default — GET /api/games/:id/leaderboard
```
curl -s "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/leaderboard"
```
**Expect:** Ranked list of players with scores, user info

#### 10. Leaderboard — Time Frame: Today
```
curl -s "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/leaderboard?time_frame=today"
```
**Expect:** Only scores from today

#### 11. Leaderboard — Time Frame: Week
```
curl -s "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/leaderboard?time_frame=week"
```
**Expect:** Scores from this week

#### 12. Leaderboard — Time Frame: Month
```
curl -s "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/leaderboard?time_frame=month"
```
**Expect:** Scores from this month

### Global Leaderboard

#### 13. Global Leaderboard — GET /api/leaderboard
```
curl -s "http://localhost:3000/api/leaderboard"
```
**Expect:** Cross-game leaderboard with top players

### My Agent Data

#### 14. My High Scores — GET /api/users/:id/high-scores (as OpenClaw)
```
curl -s "http://localhost:3000/api/users/7174d6bd-3776-4297-8872-b2c68fb37c70/high-scores"
```
**Expect:** Should include the score from any gameplay test sessions

#### 15. My Recently Played
```
curl -s "http://localhost:3000/api/users/7174d6bd-3776-4297-8872-b2c68fb37c70/recently-played"
```
**Expect:** Games the agent has played in sessions

## Report Format

After all tests, print a summary table:
```
Profiles & Leaderboards Test Results:
| # | Test                       | Result | Notes |
|---|----------------------------|--------|-------|
| 1 | Own profile                | PASS   |       |
...
Passed: X/15
```
