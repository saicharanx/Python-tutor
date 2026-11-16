# Python Tutor: project context (read this first)

## What we are building
A web app for our college that teaches Python. A chat tutor explains a concept, opens a code playground inside the chat, asks a question, checks the student's submitted code, and explains the result. Students bring their own free AI API key.

## Stack (decided)
- Backend: Python with FastAPI
- Frontend: React
- Database: PostgreSQL. Tables: students, sessions, api_keys, progress, notes, quiz_results. Designed for worldwide use.
- Hosting: free tier only

## Current scope (version 1)
- Syllabus Modules 1 to 3 only:
  - Module 1: basics, input/print, variables, operators
  - Module 2: if / elif / else
  - Module 3: while and for loops
- Modules 4 to 10 come later. Do not build them now.
- Users: our college classmates only. Not public.
- Hosting: free tier.

## Version 1 features (only these)
- Sign-up and log-in page (email and password)
- After log-in, the student is asked for their Groq API key. It is checked with a real request before chat opens. Only one AI provider (Groq) in version 1.
- Chat, with the code playground card inside it
- Standard-check quizzes: 2 per module, 15 questions each. The understanding target is 12 out of 15. This is only to check whether the student understands the topic. There is no pass or fail. During the quiz, the level goes up after 2 correct in a row and down after a miss. The final level sets the difficulty of that module's final exam.
- Final exam per module: 30 questions. +1 mark per correct answer, no negative marks.
  - 0 to 14 marks: can improve. No retry for 24 hours.
  - 15 to 24 marks: good, can improve. Retry straight away.
  - 25 to 30 marks: excellent. The student continues: the next module unlocks and the progress bar moves up.
- Checkpoint exams (for example, after Module 5, covering Modules 1 to 5): 30 questions, +1 per correct answer. No retry straight away. Score 25 to 30: the student continues to the next module. Score 24 or below: wait 24 hours before retrying. Not in version 1, which ends at Module 3.
- Progress bar (modules passed out of 10). It increases only when a final exam or checkpoint is passed. It never goes down.
- Notes panel, read-only (the AI writes concept notes only, never exercise answers; students cannot edit them)

Not in version 1: Google login, password reset, Modules 4 to 10, Markdown flowcharts in notes.

## Layout
- Left: notes panel.
- Middle: chat, with the playground card inside it.
- Top: progress bar.

## Who owns what
- Jeevan (Person 1): frontend/. Makes the whole app look good and easy to understand: sign-up/log-in page, key entry screen, chat, playground card, progress bar placement, notes panel.
- Sai Charan (Person 2): ai/. Tutor rules, Groq calls, code checker, level-check quiz, progress bar logic, syllabus data. Sends the AI's notes to Praharshini.
- Praharshini (Person 3): accounts/. Log-in, encrypted API keys and key check, database, notes design and storage (she shapes the AI's notes, then GET /notes returns them read-only to the frontend), progress and results storage, results views.

Only edit your own folder. contracts/api.md is shared. Change it only after the other two agree.

## Endpoints (full detail in contracts/api.md)
- Person 2 (ai/): POST /chat, POST /submit, POST /quiz/start, POST /quiz/answer, POST /exam/start, POST /exam/submit
- Person 3 (accounts/): POST /auth/register, POST /auth/login, POST /auth/logout, POST /api-key/check, POST /api-key/save, GET /progress, GET /notes, and the internal POST /internal/progress and POST /notes (called only by ai/)

## Rules (non-negotiable)
1. Never give the full solution to an exercise. Guide with hints.
2. Never send the answer key to the AI model. Answer keys stay on the server.
3. Never store a raw API key. Store it encrypted. Never log it. Never return it to the browser.
4. Check a key with a real request to the provider. A format check (starts with gsk_, length) is only a first hint in the UI, never proof that a key works.
5. Progress is decided by ai/ code, never by the AI. The AI only explains the result.
6. Never commit .env files or real keys. Test with fake keys only.
7. Keep changes small. Explain code before adding it. Do not add dependencies without asking.

## Known limits (do not hide these)
- The checker runs student code with `python -I` and a 3-second timeout. This is not a secure sandbox. Do not put it on a public server until it is isolated.
- The AI provider is under test (Groq). Keep the model name in one config file in ai/.

## Working rules
- One branch per person. Open a pull request when a piece works. One review before merge.
- If a request breaks a rule above, stop and say which rule.
- Notes and progress are saved per student account, not in the browser.
