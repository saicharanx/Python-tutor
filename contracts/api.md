# API contract (shared)

Status: AGREED by all three members for version 1. Items marked **Proposed** still need a final yes from the team before the first integration. Change this file only after all three agree.

## Conventions
- **Proposed:** Every protected endpoint needs `Authorization: Bearer <session_token>`. The server finds the student from the token. The browser never sends a student ID.
- **Decided:** Errors return `{"message": "..."}`. Status codes: 200 success, 400 bad input, 401 not logged in, 500 server error.
- **Proposed:** Keys are never sent to the browser after saving.

## Accounts (Person 3: Praharshini)

### POST /auth/register — Proposed
Request: `{"email": "...", "password": "..."}`
Success: `{"message": "Account created successfully"}`
Duplicate: `{"message": "Email already registered"}`

### POST /auth/login — Proposed
Request: `{"email": "...", "password": "..."}`
Success: `{"session_token": "...", "expires_in": 86400, "student": {"id": "...", "email": "..."}}`
Failure: `{"message": "Invalid email or password"}`
Session expires after 24 hours of inactivity.

### POST /auth/logout — Proposed
Header: Bearer token. Success: `{"message": "Logged out successfully"}`

### POST /auth/google — REMOVED from version 1
Google login is out of scope for version 1.

### POST /api-key/check — Proposed
Request: `{"api_key": "..."}`
Pass: `{"status": "pass"}`
Fail: `{"status": "fail", "message": "Your API key is invalid"}`
Provider down or network error: `{"status": "fail", "message": "..."}`
The check is a real request to the provider. A format check is only a hint in the UI.

### POST /api-key/save — Decided (check, then save)
Owner: Person 3. The page calls `/api-key/check` first. Only if it passes does it call this endpoint with the same key.
Request: `{"api_key": "..."}`. Response: `{"status": "saved"}`. The key is encrypted before storage and never returned.

### Internal: get_decrypted_key — Decided
Not a public endpoint. Used by ai/ code to call the provider. Must never be reachable from the browser.

### GET /progress — Confirmed
Success: `{"progress_percentage": 30, "modules_passed": 3, "total_modules": 10, "current_module": "Module 4"}`
The bar counts modules out of 10, the whole course. progress_percentage = modules_passed / total_modules × 100.
Decided: Sai Charan's ai/ code calculates progress, based only on passed module final exams and checkpoints. Quizzes never change progress. It sends the result to accounts (see POST /internal/progress below). GET /progress returns the stored value to the frontend.

### POST /internal/progress — Confirmed
Owner: Person 3 (accounts/). Called only by ai/. Not reachable from the browser.
Header: `X-Service-Secret: <shared secret from .env>`. Missing or wrong secret: 401.
Request: `{"student_id": "...", "modules_passed": 3, "total_modules": 10, "progress_percentage": 30, "current_module": "Module 4"}`
Success: `{"status": "saved"}`
Unknown student: 400 with `{"message": "Student not found"}`
Rules: progress never goes down. The progress bar increases only when a module is finished. Each quiz result is kept in quiz_results.

### POST /notes — Confirmed
Owner: Person 3 (accounts/). Called only by ai/. Not reachable from the browser.
Header: `X-Service-Secret: <shared secret from .env>`.
Request: `{"student_id": "...", "module_name": "Module 1", "topic": "Variables", "main_points": ["...", "..."]}`
Accounts turns `main_points` into clean Markdown and stores it as `content`.
Success: `{"status": "saved"}`

### GET /notes — Confirmed
Owner: Person 3. Requires the student's Bearer token. Read-only for students.
Response: a list of notes, newest first, each with `module_name`, `topic`, and `content` (Markdown).
Example: `{"notes": [{"module_name": "Module 1", "topic": "Variables", "content": "## Variables\nA variable stores a value..."}]}`

Limits (proposed): content up to 10,000 characters; up to 100 saved notes per student.
Decided: Sai Charan's ai/ code picks the main points of each explanation and sends all of that information to accounts (POST /notes). Praharshini designs the final note format and returns it through GET /notes. Jeevan displays it read-only. The notes are for students to revise from, so they must be clear without extra help.
Timing decided: ai/ asks the student whether the information should be saved as a note for later reading. Only if the student says yes is it sent to accounts. The exact note fields are decided by Praharshini.
Flowcharts in notes: not in v1. Markdown text only.

## AI tutor (Person 2: Sai Charan)

### POST /chat — Proposed, with Jeevan's changes applied
Request: `{"message": "student text"}`
Student ID removed (token identifies the student). Progress removed (server reads it).
Response:
```json
{
  "reply": "tutor text",
  "question": {"id": "q1", "title": "...", "text": "..."},
  "playground": {"question_id": "q1", "starter_code": "..."},
  "note": {"module_name": "Module 1", "topic": "...", "content": "markdown"}
}
```
`question`, `playground` and `note` can each be `null`.

### POST /submit — Proposed
Request: `{"question_id": "q1", "code": "..."}`
Question ID is required (agreed by Jeevan's draft, confirm with Person 2).
Response: `{"passed": true, "output": "...", "error": null, "tutor_reply": "..."}`
The `passed` value is decided by ai/ code, never by the AI.

### POST /quiz/answer — Decided (one endpoint)
Owner: Person 2 (ai/). Used by the level-check quiz.
Request: `{"question_id": "q1", "answer": "..."}`
Response: `{"correct": true, "level": "medium", "next_question": {"id": "q2", "text": "...", "options": ["..."]}}`
`level` is decided by ai/ code: starts at medium, up after 2 correct in a row, down after a miss. The final level sets the module final exam's difficulty. The quiz never passes a module.

## Quizzes and exams (Person 2: Sai Charan) — Confirmed

### POST /quiz/start — Proposed
Request: `{"module_id": "module-1"}`. Returns the first standard-check question with no answer shown.

### POST /quiz/answer — see above. After the 15th question, returns the score.

### POST /exam/start — Proposed
Request: `{"module_id": "module-1"}`. Uses the two standard-check results to pick difficulty. Returns 30 questions with no answers shown.

### POST /exam/submit — Proposed
Request: `{"exam_id": "...", "answers": [{"question_id": "...", "answer": "..."}]}`
Response: `score`, `band` (`can_improve`, `good`, or `excellent`), `retry_available_at` (now for 15 to 24; 24 hours later for 0 to 14).
If `band` is `excellent`: `next_module_unlocked: true`, and the server calls POST /internal/progress.

### POST /checkpoint/start and POST /checkpoint/submit — Same pattern as the module exam. Not in version 1; add when Module 5 is built.

## Decisions log
1. Decided: ai/ calculates progress from passed final exams and checkpoints, out of 10 modules.
2. Decided: Google login and password reset are out of version 1.
3. Decided: quiz is one endpoint, POST /quiz/answer. Key is checked, then saved.
4. Decided: notes content comes from ai/ main points, designed by Praharshini.
5. Decided: ai/ sends a note only when the student asks to save it.
6. Decided: note fields are module_name, topic, and content (Markdown), as confirmed in the notes section.
7. Decided: the internal key lookup is named get_decrypted_key.
8. Decided: standard status codes (200, 400, 401, 500).
9. Decided: ai/ sends progress to accounts with POST /internal/progress. Progress never goes down; the bar increases only when a module is finished.
10. Decided: backend in Python (FastAPI), frontend in React, database PostgreSQL. See the Stack section in the root CLAUDE.md.
11. Decided: 2 standard-check quizzes per module (15 questions, understanding target 12, informational only) set the final exam difficulty. Module final exam: 30 questions, +1 per correct answer. 0 to 14 can improve with no retry for 24 hours. 15 to 24 good, retry straight away. 25 to 30 excellent, next module unlocks and the bar moves up. See CLAUDE.md.
12. Decided: the quiz and exam endpoints listed above.
13. Decided: checkpoint exams (after Module 5 and later) are 30 questions. Score 25 to 30 continues to the next module. Score 24 or below waits 24 hours before retry. No straight retry. Version 1 has no checkpoint. See CLAUDE.md.
