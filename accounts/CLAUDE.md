# accounts/ (Person 3: Praharshini)

Read the root CLAUDE.md first.

## You own
- Login for students
- Encrypted storage of each student's API key. The encryption key comes from an environment variable (name: GROQ_KEY_ENCRYPTION_KEY). Its value lives only in your local .env file.
- Key check: a real test request to the provider. Never a format-only check.
- Database design (tables for students, keys, progress, notes)
- Notes design: receive the AI's concept notes from ai/ and decide how they are stored and returned (GET /notes). Notes are read-only for students.
- Endpoints you own: POST /auth/register, POST /auth/login, POST /auth/logout, POST /api-key/check, POST /api-key/save, GET /progress, GET /notes
- Internal endpoints you own, called only by ai/ with the X-Service-Secret header: POST /internal/progress, POST /notes, POST /internal/results, and GET /internal/results
- You are the only store for quiz, exam and checkpoint results. ai/ reads them back from you.
- Saving progress per student account, as received from ai/
- Results views for the team (for example, quiz results)

## Rules for this folder
- Never return a stored key to any client, including the student who owns it.
- Never write a key to logs, error messages, or the database in plain text.
- Progress is written here as received from ai/. Do not compute it here.
- Do not change files in frontend/ or ai/.
