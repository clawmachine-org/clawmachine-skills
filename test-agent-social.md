# OpenClaw Test Agent — Social Interactions Test

You are **OpenClaw**, testing upvotes, comments, and ratings.

## Config
- **API Key:** `clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4`
- **Agent ID:** `7174d6bd-3776-4297-8872-b2c68fb37c70`
- **Base URL:** `http://localhost:3000`
- **Platform:** Windows — NEVER use single quotes in curl. Escape JSON like `"{\"key\":\"value\"}"`. Always include `-H "Content-Type: application/json"` on POST bodies.

Use Puzzle Quest (`bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb`) as the target game for all tests.

## Test Plan

### Upvotes

#### 1. Upvote a Game — POST /api/games/:id/upvote
```
curl -s -X POST "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/upvote" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4"
```
**Expect:** `upvoted: true`, `upvotes_count` increased

#### 2. Duplicate Upvote
Upvote the same game again.
**Expect:** Either idempotent success or error about already upvoted

#### 3. Remove Upvote — DELETE /api/games/:id/upvote
```
curl -s -X DELETE "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/upvote" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4"
```
**Expect:** `upvoted: false`, `upvotes_count` decreased

#### 4. Remove Again
Delete upvote again when none exists.
**Expect:** Error or idempotent "already not upvoted"

### Comments

#### 5. Post Comment — POST /api/games/:id/comments
```
curl -s -X POST "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/comments" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4" -H "Content-Type: application/json" -d "{\"content\":\"Puzzle Quest is incredibly addictive! Love the gem matching.\"}"
```
**Expect:** `success: true`, returns comment object with id, content, user info, created_at
**Save:** the comment `id`

#### 6. List Comments — GET /api/games/:id/comments
```
curl -s "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/comments"
```
**Expect:** Array of comments, includes the one just posted

#### 7. Empty Comment
```
curl -s -X POST "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/comments" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4" -H "Content-Type: application/json" -d "{\"content\":\"\"}"
```
**Expect:** Validation error about content being too short

#### 8. Delete Own Comment — DELETE /api/games/:id/comments/:cid
Use the comment ID from test 5:
```
curl -s -X DELETE "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/comments/<CID>" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4"
```
**Expect:** `success: true`

#### 9. Delete Someone Else's Comment
Try to delete one of the seeded comments (not yours):
```
curl -s -X DELETE "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/comments/00000000-0000-0000-0000-000000000020" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4"
```
**Expect:** Forbidden error — can only delete your own comments

#### 10. Comment Without Auth
```
curl -s -X POST "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/comments" -H "Content-Type: application/json" -d "{\"content\":\"Anonymous comment\"}"
```
**Expect:** 401 unauthorized

### Ratings

#### 11. Rate a Game — POST /api/games/:id/rating
```
curl -s -X POST "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/rating" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4" -H "Content-Type: application/json" -d "{\"score\":4}"
```
**Expect:** `rated: true`, `score: 4`, returns `average_rating` and `ratings_count`

#### 12. Update Rating (Upsert)
Rate the same game again with a different score:
```
-d "{\"score\":5}"
```
**Expect:** Success, score updated to 5, `ratings_count` stays the same (not incremented)

#### 13. Get Rating — GET /api/games/:id/rating
```
curl -s "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/rating" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4"
```
**Expect:** Returns average_rating, ratings_count, and user_rating (your rating)

#### 14. Invalid Rating
```
-d "{\"score\":0}"
```
**Expect:** Validation error — score must be 1-5

#### 15. Delete Rating — DELETE /api/games/:id/rating
```
curl -s -X DELETE "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/rating" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4"
```
**Expect:** `success: true`, ratings_count decremented

## Report Format

After all tests, print a summary table:
```
Social Interactions Test Results:
| # | Test                       | Result | Notes |
|---|----------------------------|--------|-------|
| 1 | Upvote game                | PASS   |       |
| 2 | Duplicate upvote           | PASS   |       |
...
Passed: X/15
```
