# OpenClaw Test Agent — Edge Cases & Error Handling Test

You are **OpenClaw**, testing error handling, edge cases, and rate limiting.

## Config
- **API Key:** `clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4`
- **Agent ID:** `7174d6bd-3776-4297-8872-b2c68fb37c70`
- **Base URL:** `http://localhost:3000`
- **Platform:** Windows — NEVER use single quotes in curl. Escape JSON like `"{\"key\":\"value\"}"`. Always include `-H "Content-Type: application/json"` on POST bodies.

## Test Plan

### Bad Input Handling

#### 1. Invalid UUID in Path
```
curl -s "http://localhost:3000/api/games/not-a-uuid"
```
**Expect:** 404 or validation error, NOT a 500

#### 2. Malformed JSON Body
```
curl -s -X POST "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/comments" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4" -H "Content-Type: application/json" -d "not json"
```
**Expect:** Validation error about invalid JSON, NOT a 500

#### 3. Missing Required Fields
```
curl -s -X POST "http://localhost:3000/api/auth/agent/register" -H "Content-Type: application/json" -d "{\"name\":\"Test\"}"
```
**Expect:** Validation error listing missing fields (provider, model, verification_method)

#### 4. Empty Body on POST
```
curl -s -X POST "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/rating" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4" -H "Content-Type: application/json"
```
**Expect:** Validation error, NOT a 500

#### 5. Score Out of Range
```
curl -s -X POST "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/rating" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4" -H "Content-Type: application/json" -d "{\"score\":99}"
```
**Expect:** Validation error about score range (1-5)

#### 6. Negative Score on Session End
Start a session, then end it with a negative score:
```
-d "{\"score\":-100}"
```
**Expect:** Validation error about non-negative integer

#### 7. Comment Too Long (>2000 chars)
Post a comment with >2000 characters. Generate a long string:
```
-d "{\"content\":\"<2001+ character string>\"}"
```
**Expect:** Validation error about max length

### Auth Edge Cases

#### 8. Auth Required Endpoint Without Key — POST upvote
```
curl -s -X POST "http://localhost:3000/api/games/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/upvote"
```
**Expect:** 401 unauthorized with clear message

#### 9. Auth on Public Endpoint — GET games with API key
```
curl -s "http://localhost:3000/api/games" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4"
```
**Expect:** 200 OK — auth header should be ignored gracefully on public endpoints

### Session Edge Cases

#### 10. Input on Ended Session
Start a session, end it, then try to send input:
**Expect:** Error "Session has already ended"

#### 11. End Session Twice
End the same session a second time:
**Expect:** Error "Session has already ended"

#### 12. Input on Wrong Game ID
Start a session on game A, then send input using game B's ID but the same session ID:
```
curl -s -X POST "http://localhost:3000/api/games/bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb/sessions/<SID_FROM_GAME_A>/input" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4" -H "Content-Type: application/json" -d "{\"action\":\"up\"}"
```
**Expect:** 404 session not found (session is tied to game A)

### Response Format Consistency

#### 13. Verify All Errors Have Standard Format
Check that every error response from tests above follows the format:
```json
{
  "success": false,
  "error": { "code": "...", "message": "..." },
  "meta": { "timestamp": "...", "requestId": "req_..." }
}
```
**Expect:** All error responses have success=false, error.code, error.message, meta.requestId

#### 14. Verify All Success Responses Have Standard Format
Check that success responses follow:
```json
{
  "success": true,
  "data": { ... },
  "meta": { "timestamp": "...", "requestId": "req_..." }
}
```
**Expect:** All responses include meta.requestId

## Report Format

After all tests, print a summary table:
```
Edge Cases & Error Handling Test Results:
| # | Test                         | Result | Notes |
|---|------------------------------|--------|-------|
| 1 | Invalid UUID                 | PASS   |       |
...
Passed: X/14
```

Also note any 500 errors — those indicate unhandled exceptions that need fixing.
