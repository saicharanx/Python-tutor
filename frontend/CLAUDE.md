# frontend/ (Person 1: Jeevan)

Read the root CLAUDE.md first.

## You own
- Look and ease of use for the whole app: sign-up/log-in page, key entry screen
- Layout: notes panel (left), chat with playground card (middle), progress bar (top)
- The playground card inside the chat, and the submit button
- Showing the tutor's replies
- The "Save this as a note?" prompt. Show it only when a /chat response has a non-null `note`. Yes sends POST /chat with `save_note: true` and an empty `message`. No sends nothing. Show the server's reply after Yes.
- Quiz and exam screens: questions, options, the quiz's end-of-quiz score, the exam score and band, and the 24-hour retry wait
- Showing notes read-only, in the format GET /notes returns (from Praharshini). Students cannot edit, delete, or add notes.

## You call only
- POST /auth/register, POST /auth/login, POST /auth/logout (Person 3)
- POST /api-key/check and POST /api-key/save (Person 3)
- POST /chat and POST /submit (Person 2)
- POST /quiz/start, POST /quiz/answer, POST /exam/start, POST /exam/submit (Person 2)
- GET /progress and GET /notes (Person 3)

Do not call the AI provider directly from the browser.

## Rules for this folder
- The API key goes from the input field straight to the server. Do not store it in localStorage, cookies, or logs.
- The key format check is a hint for the student only. The server does the real check.
- Show the progress bar from the value GET /progress returns. Sai Charan's ai/ code calculates it. Do not compute progress in the browser.
- Do not change files in ai/ or accounts/.
