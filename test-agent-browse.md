# OpenClaw Test Agent — Browse & Discovery Test

You are **OpenClaw**, testing all public browsing and discovery endpoints.

## Config
- **Base URL:** `http://localhost:3000`
- **Platform:** Windows — NEVER use single quotes in curl.

## Test Plan

Run these tests in order and report pass/fail for each:

### 1. List All Games — GET /api/games
```
curl -s "http://localhost:3000/api/games"
```
**Expect:** `success: true`, `data.games` is an array, `data.pagination` has total/limit/offset/hasMore

### 2. Pagination — GET /api/games?limit=2&offset=2
```
curl -s "http://localhost:3000/api/games?limit=2&offset=2"
```
**Expect:** Returns 2 games, different from the first page

### 3. Genre Filter — GET /api/games?genre=arcade
```
curl -s "http://localhost:3000/api/games?genre=arcade"
```
**Expect:** All returned games have `genre: "arcade"`

### 4. Invalid Genre — GET /api/games?genre=faketype
```
curl -s "http://localhost:3000/api/games?genre=faketype"
```
**Expect:** Validation error about invalid genre

### 5. Search — GET /api/games?search=space
```
curl -s "http://localhost:3000/api/games?search=space"
```
**Expect:** Returns games with "space" in title or description

### 6. Tag Filter — GET /api/games?tags=retro
```
curl -s "http://localhost:3000/api/games?tags=retro"
```
**Expect:** Returns games that have "retro" in their tags array

### 7. Sort by Plays — GET /api/games?sort=plays_count&order=desc
```
curl -s "http://localhost:3000/api/games?sort=plays_count&order=desc"
```
**Expect:** Games ordered by plays_count descending (highest first)

### 8. Trending — GET /api/discover/trending
```
curl -s "http://localhost:3000/api/discover/trending"
```
**Expect:** `success: true`, array of games with agent info, ordered by popularity

### 9. Featured — GET /api/discover/featured
```
curl -s "http://localhost:3000/api/discover/featured"
```
**Expect:** `success: true`, only games where `isFeatured` is true

### 10. Latest — GET /api/discover/latest
```
curl -s "http://localhost:3000/api/discover/latest"
```
**Expect:** `success: true`, games ordered by creation date descending

### 11. Game Details — GET /api/games/:id
```
curl -s "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
```
**Expect:** Full game object with agent info, plays_count, upvotes_count, etc.

### 12. Game Not Found — GET /api/games/:id
```
curl -s "http://localhost:3000/api/games/00000000-0000-0000-0000-000000000000"
```
**Expect:** 404 not found error

### 13. Genres List — GET /api/genres
```
curl -s "http://localhost:3000/api/genres"
```
**Expect:** Array of genre objects with id, name, count, icon

### 14. Popular Tags — GET /api/tags/popular
```
curl -s "http://localhost:3000/api/tags/popular"
```
**Expect:** Array of tag objects with name and count, sorted by count

### 15. Platform Stats — GET /api/stats
```
curl -s "http://localhost:3000/api/stats"
```
**Expect:** Object with games_created, active_agents, human_players, total_plays

### 16. Global Leaderboard — GET /api/leaderboard
```
curl -s "http://localhost:3000/api/leaderboard"
```
**Expect:** Leaderboard entries with user info and scores

## Report Format

After all tests, print a summary table:
```
Browse & Discovery Test Results:
| # | Test                  | Result | Notes |
|---|-----------------------|--------|-------|
| 1 | List all games        | PASS   |       |
...
Passed: X/16
```
