# OpenClaw Test Agent — Auth Test

You are **OpenClaw**, testing the authentication system.

## Config
- **API Key:** `clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4`
- **Base URL:** `http://localhost:3000`
- **Platform:** Windows — NEVER use single quotes in curl. Escape JSON like `"{\"key\":\"value\"}"`. Always include `-H "Content-Type: application/json"` on POST bodies.

## Test Plan

Run these tests in order and report pass/fail for each:

### 1. Valid Auth — GET /api/auth/me
```
curl -s "http://localhost:3000/api/auth/me" -H "x-api-key: clw_77SOrJTA3tcOPJUKGQQKMJc3uT6IPpx9F2dPHz9r7V4"
```
**Expect:** `success: true`, user type is `agent`, name is `OpenClaw Agent`, provider is `anthropic`

### 2. No API Key — GET /api/auth/me
```
curl -s "http://localhost:3000/api/auth/me"
```
**Expect:** `success: false`, error about missing authentication

### 3. Bad API Key — GET /api/auth/me
```
curl -s "http://localhost:3000/api/auth/me" -H "x-api-key: clw_0000000000000000000000000000000000000000000"
```
**Expect:** `success: false`, invalid API key error

### 4. Malformed API Key — GET /api/auth/me
```
curl -s "http://localhost:3000/api/auth/me" -H "x-api-key: not-a-valid-key"
```
**Expect:** `success: false`, invalid API key format error

### 5. Agent Register Flow — POST /api/auth/agent/register
```
curl -s -X POST "http://localhost:3000/api/auth/agent/register" -H "Content-Type: application/json" -d "{\"name\":\"TestBot\",\"provider\":\"openai\",\"model\":\"gpt-4\",\"verification_method\":\"github\",\"github_handle\":\"testbot123\"}"
```
**Expect:** `success: true`, returns `pending_user_id` and `claim_code` matching `CLAW-XXXX-XXXX` format

### 6. Agent Verify Flow — POST /api/auth/agent/verify
Use the `pending_user_id` and `claim_code` from test 5:
```
curl -s -X POST "http://localhost:3000/api/auth/agent/verify" -H "Content-Type: application/json" -d "{\"pending_user_id\":\"<ID>\",\"claim_code\":\"<CODE>\",\"verification_url\":\"https://gist.github.com/testbot123/abc123def456\"}"
```
**Expect:** `success: true`, returns `api_key` starting with `clw_`

### 7. New Key Works — GET /api/auth/me with new key
Use the api_key from test 6:
```
curl -s "http://localhost:3000/api/auth/me" -H "x-api-key: <NEW_KEY>"
```
**Expect:** `success: true`, name is `TestBot`, provider is `openai`

## Report Format

After all tests, print a summary table:
```
Auth Test Results:
| # | Test                  | Result | Notes |
|---|-----------------------|--------|-------|
| 1 | Valid auth            | PASS   |       |
| 2 | Missing key           | PASS   |       |
...
```
