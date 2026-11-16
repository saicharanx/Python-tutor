# API contract (shared)

Status: AGREED. Every item is confirmed by the team for version 1. Change this file only after all three members agree.

## Conventions
- Every protected endpoint needs `Authorization: Bearer <session_token>`. The server finds the student from the token. The browser never sends a student ID.
- Errors return `{"message": "..."}`. Status codes: 200 success, 400 bad input, 401 not logged in, 500 server error.
- Keys are never sent back to the browser after saving.
- Internal endpoints (marked "Called only by ai/") use the header `X-Service-Secret`. The value comes from `SERVICE_SECRET` in each service's `.env`. They do not use the student's Bearer token.

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
Rules: progress never goes down. The progress bar increases only when a module is finished. Quiz, exam, and checkpoint results are sent separately through POST /internal/results.

### POST /notes
Owner: Person 3 (accounts/). Called only by ai/. Not reachable from the browser.
Header: `X-Service-Secret: <shared secret from .env>`.
Request: `{"student_id": "...", "module_name": "Module 1", "topic": "Variables", "main_points": ["...", "..."]}`
Accounts turns `main_points` into clean Markdown and stores it as `content`.
Success: `{"status": "saved"}`
Sent only when the student asks to save an explanation as a note.

### POST /internal/results
Owner: Person 3 (accounts/). Called only by ai/. Not reachable from the browser.
Header: `X-Service-Secret: <shared secret from .env>`.
Purpose: store every quiz, exam, and checkpoint attempt, so results views and the 24-hour retry check have data.
Request: `{"student_id": "...", "attempt_type": "quiz | exam | checkpoint", "module_id": "module-1", "score": 12, "total": 15, "band": "good", "level": "medium", "attempted_at": "2026-11-20T10:00:00Z"}`
`band` is only for exams and checkpoints. `level` is only for quizzes.
Success: `{"status": "saved"}`

If a save fails, ai/ retries it. ai/ keeps no copy of results. Accounts is the only store.

### GET /internal/results
Owner: Person 3 (accounts/). Called only by ai/. Not reachable from the browser.
Header: `X-Service-Secret: <shared secret from .env>`.
Purpose: ai/ reads a student's results back. It uses the two standard-check results to set the exam's difficulty, and the latest exam attempt to enforce the 24-hour retry wait.
Request: `?student_id=...&module_id=module-1` (module_id is optional)
Response: `{"results": [{"attempt_type": "quiz", "module_id": "module-1", "score": 12, "total": 15, "band": null, "level": "medium", "attempted_at": "..."}]}`, newest first.

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
Owner: Person 2. Request: `{"module_id": "module-1"}`. The server works out which quiz is next for that module: quiz 1 first, then quiz 2. Returns `{"quiz_number": 1, "question": {"id": "q1", "text": "...", "options": ["..."]}}` with no answer shown.
If both quizzes for the module are already finished, returns 400 with `{"message": "Both quizzes finished for this module"}`.
A student cannot restart or skip a quiz. If the student has a quiz in progress, this returns the current question, so a student who leaves mid-quiz resumes at the same question. Quiz state is kept in memory by ai/. The "resumes at the same question" promise holds only while the AI server stays running. If the server restarts or sleeps (common on free hosts), an unfinished quiz is lost and the student's next call to POST /quiz/start begins that quiz again with fresh questions. Only finished quizzes are saved to accounts (POST /internal/results).

### POST /quiz/answer
Owner: Person 2. Request: `{"question_id": "q1", "answer": "..."}`
Response: `{"correct": true, "level": "medium", "next_question": {"id": "q2", "text": "...", "options": ["..."]}}`
After the 15th question, the response is instead: `{"finished": true, "score": 12, "total": 15, "final_level": "medium"}`. ai/ then sends this attempt to POST /internal/results with `attempt_type: "quiz"`.
`level` is decided by ai/ code: starts at medium, up after 2 correct in a row, down after a miss. Each quiz's final level feeds the module exam's difficulty (decision 13). The quiz never passes a module.

### POST /exam/start
Owner: Person 2. Request: `{"module_id": "module-1"}`. Refuses with 400 and `{"message": "Finish both standard-check quizzes first"}` unless both quizzes for that module are finished. Uses the final levels of the two standard-check quizzes. The exam questions come from every level between those two levels, inclusive. For example, quizzes ending at medium and hard give a mix of medium and hard questions. Returns 30 questions with no answers shown.

### POST /exam/submit
Owner: Person 2. Request: `{"exam_id": "...", "answers": [{"question_id": "...", "answer": "..."}]}`
Exams are kept in memory by ai/, like quizzes. If the server restarts or sleeps during an exam, the exam is lost. A submit for a lost exam returns 410 with `{"message": "Your exam was interrupted. Start the exam again."}`. A lost exam is not saved, so it has no 24-hour wait, and the student can start it again right away.
Response: `score`, `band` (`can_improve` for 0 to 14, `good` for 15 to 24, `excellent` for 25 to 30), `retry_available_at` (now for 15 to 24; 24 hours later for 0 to 14; `null` for excellent). ai/ then sends this attempt to POST /internal/results.
If `band` is `excellent`: `next_module_unlocked: true`, and the server calls POST /internal/progress.

### POST /checkpoint/start and POST /checkpoint/submit
Same pattern as the module exam. Not in version 1. Add when Module 5 is built.

## Decisions log
1. ai/ calculates progress from passed final exams and checkpoints, out of 10 modules.
2. Google login and password reset are out of version 1.
3. The quiz uses two endpoints: POST /quiz/start to get the first question, and POST /quiz/answer for each answer. The key is checked, then saved.
4. Notes content comes from ai/ main points, designed by Praharshini.
5. ai/ sends a note only when the student asks to save it.
6. Note fields are module_name, topic, and content (Markdown).
7. The internal key lookup is named get_decrypted_key.
8. Standard status codes: 200, 400, 401, 500.
9. ai/ sends progress to accounts with POST /internal/progress. Progress never goes down; the bar increases only when a module is finished.
10. Backend in Python (FastAPI), frontend in React, database PostgreSQL. See the Stack section in the root CLAUDE.md.
11. 2 standard-check quizzes per module (15 questions, understanding target 12, informational only) set the final exam difficulty. Module final exam: 30 questions, +1 per correct answer. 0 to 14 can improve with no retry for 24 hours. 15 to 24 good, retry straight away. 25 to 30 excellent, next module unlocks and the bar moves up.
12. The quiz and exam endpoints are listed above.
13. Module exam difficulty comes from the levels between the two standard-check quiz results, inclusive. Accounts stores all results. ai/ reads them back through GET /internal/results, so the tutor can adjust how it explains things to each student's level.
14. Quizzes are part of the learning path. Once a quiz starts it cannot be restarted or skipped by the student. A student who leaves mid-quiz resumes at the same question while the AI server stays running. If the server restarts or sleeps, an unfinished quiz restarts with fresh questions, and only finished quizzes are saved. The module exam cannot start until both quizzes for that module are finished (400 otherwise).
15. Exams are kept in memory like quizzes. If the server restarts or sleeps during an exam, the exam is lost and not saved. The student starts it again with no 24-hour wait.
16. Checkpoint exams (after Module 5 and later) are 30 questions. Score 25 to 30 continues to the next module. Score 24 or below waits 24 hours before retry. No straight retry. Version 1 has no checkpoint.
