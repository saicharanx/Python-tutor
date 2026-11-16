# API contract (shared)

Status: AGREED. All items below are confirmed by the team for version 1. Change this file only after all three members agree.

## Conventions
- Every protected endpoint needs `Authorization: Bearer <session_token>`. The server finds the student from the token. The browser never sends a student ID.
- Errors return `{"message": "..."}`. Status codes: 200 success, 400 bad input, 401 not logged in, 500 server error.
- Keys are never sent back to the browser after saving.

## Accounts (Person 3: Praharshini)

### POST /auth/register
Request: `{"email": "...", "password": "..."}`
Success: `{"message": "Account created successfully"}`
Duplicate: `{"message": "Email already registered"}`

### POST /auth/login
Request: `{"email": "...", "password": "..."}`
Success: `{"session_token": "...", "expires_in": 86400, "student": {"id": "...", "email": "..."}}`
Failure: `{"message": "Invalid email or password"}`
Session expires after 24 hours of inactivity.

### POST /auth/logout
Header: Bearer token. Success: `{"message": "Logged out successfully"}`

### POST /api-key/check
Request: `{"api_key": "..."}`
Pass: `{"status": "pass"}`
Fail: `{"status": "fail", "message": "Your API key is invalid"}`
Provider down or network error: `{"status": "fail", "message": "..."}`
The check is a real request to the provider. A format check is only a hint in the UI.

### POST /api-key/save
The page calls `/api-key/check` first. Only if it passes does it call this endpoint with the same key.
Request: `{"api_key": "..."}`. Response: `{"status": "saved"}`. The key is encrypted before storage and never returned.

### Internal: get_decrypted_key
Not a public endpoint. Used by ai/ code to call the provider. Must never be reachable from the browser.

### GET /progress
Requires Bearer token.
Success: `{"progress_percentage": 30, "modules_passed": 3, "total_modules": 10, "current_module": "Module 4"}`
The bar counts modules out of 10, the whole course. `progress_percentage` = `modules_passed` / `total_modules` × 100.
Sai Charan's ai/ code calculates progress, based only on passed module final exams and checkpoints. Quizzes never change progress. ai/ sends the result to accounts (POST /internal/progress). GET /progress returns the stored value.

### POST /internal/progress
Owner: Person 3 (accounts/). Called only by ai/. Not reachable from the browser.
Header: `X-Service-Secret: <shared secret from .env>`. Missing or wrong secret: 401.
Request: `{"student_id": "...", "modules_passed": 3, "total_modules": 10, "progress_percentage": 30, "current_module": "Module 4"}`
Success: `{"status": "saved"}`
Unknown student: 400 with `{"message": "Student not found"}`
Rules: progress never goes down. The progress bar increases only when a module is finished. Each quiz result is kept in quiz_results.

### POST /notes
Owner: Person 3 (accounts/). Called only by ai/. Not reachable from the browser.
Header: `X-Service-Secret: <shared secret from .env>`.
Request: `{"student_id": "...", "module_name": "Module 1", "topic": "Variables", "main_points": ["...", "..."]}`
Accounts turns `main_points` into clean Markdown and stores it as `content`.
Success: `{"status": "saved"}`
Sent only when the student asks to save an explanation as a note.

### GET /notes
Owner: Person 3. Requires the student's Bearer token. Read-only for students.
Response: a list of notes, newest first, each with `module_name`, `topic`, and `content` (Markdown).
Example: `{"notes": [{"module_name": "Module 1", "topic": "Variables", "content": "## Variables\nA variable stores a value..."}]}`
Limits: content up to 10,000 characters; up to 100 saved notes per student.
Praharshini designs the final note format. Jeevan displays it read-only. Notes must be clear enough for a student to revise from without extra help.

## AI tutor and exams (Person 2: Sai Charan)

### POST /chat
Request: `{"message": "student text"}`
The student ID is not sent. The token identifies the student. Progress is not sent. The server reads it.
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

### POST /submit
Request: `{"question_id": "q1", "code": "..."}`. The question ID is required.
Response: `{"passed": true, "output": "...", "error": null, "tutor_reply": "..."}`
The `passed` value is decided by ai/ code, never by the AI.

### POST /quiz/start
Owner: Person 2. Request: `{"module_id": "module-1"}`. Returns the first standard-check question with no answer shown.

### POST /quiz/answer
Owner: Person 2. Request: `{"question_id": "q1", "answer": "..."}`
Response: `{"correct": true, "level": "medium", "next_question": {"id": "q2", "text": "...", "options": ["..."]}}`
After the 15th question, the response returns the score instead of a next question.
`level` is decided by ai/ code: starts at medium, up after 2 correct in a row, down after a miss. The final level sets the module final exam's difficulty. The quiz never passes a module.

### POST /exam/start
Owner: Person 2. Request: `{"module_id": "module-1"}`. Uses the two standard-check results to pick difficulty. Returns 30 questions with no answers shown.

### POST /exam/submit
Owner: Person 2. Request: `{"exam_id": "...", "answers": [{"question_id": "...", "answer": "..."}]}`
Response: `score`, `band` (`can_improve`, `good`, or `excellent`), `retry_available_at` (now for 15 to 24; 24 hours later for 0 to 14).
If `band` is `excellent`: `next_module_unlocked: true`, and the server calls POST /internal/progress.

### POST /checkpoint/start and POST /checkpoint/submit
Same pattern as the module exam. Not in version 1. Add when Module 5 is built.

## Decisions log
1. ai/ calculates progress from passed final exams and checkpoints, out of 10 modules.
2. Google login and password reset are out of version 1.
3. Quiz is one endpoint, POST /quiz/answer. Key is checked, then saved.
4. Notes content comes from ai/ main points, designed by Praharshini.
5. ai/ sends a note only when the student asks to save it.
6. Note fields are module_name, topic, and content (Markdown).
7. The internal key lookup is named get_decrypted_key.
8. Standard status codes: 200, 400, 401, 500.
9. ai/ sends progress to accounts with POST /internal/progress. Progress never goes down; the bar increases only when a module is finished.
10. Backend in Python (FastAPI), frontend in React, database PostgreSQL. See the Stack section in the root CLAUDE.md.
11. 2 standard-check quizzes per module (15 questions, understanding target 12, informational only) set the final exam difficulty. Module final exam: 30 questions, +1 per correct answer. 0 to 14 can improve with no retry for 24 hours. 15 to 24 good, retry straight away. 25 to 30 excellent, next module unlocks and the bar moves up.
12. The quiz and exam endpoints are listed above.
13. Checkpoint exams (after Module 5 and later) are 30 questions. Score 25 to 30 continues to the next module. Score 24 or below waits 24 hours before retry. No straight retry. Version 1 has no checkpoint.
