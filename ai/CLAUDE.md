# ai/ (Person 2: Sai Charan)

Read the root CLAUDE.md first.

## You own
- Tutor system prompt and reply rules (short replies, hints, no full solutions)
- Calls to the AI provider (Groq, the only provider in version 1), with the model name in one config file
- The checker: syntax check with ast.parse, then a run with `python -I` and a 3-second timeout
- Syllabus data for Modules 1 to 3
- Module exam difficulty: questions come from every level between the two standard-check quiz results, inclusive.
- Read each student's results from accounts/ (GET /internal/results). Keep no copy of results. The tutor uses them to adjust its explanations to the student's level, without giving answers.
- If saving a result to accounts/ fails, retry the save. Never decide retries or difficulty from a local copy.
- Quizzes cannot be restarted or skipped. Keep the quiz state on the server, so a student who leaves mid-quiz resumes at the same question. A module exam cannot start until both quizzes for that module are finished.
- Standard-check quizzes: 2 per module, 15 questions each. Understanding target 12 out of 15, informational only, no pass or fail. During the quiz, the level goes up after 2 correct in a row and down after a miss. The two quizzes' final levels set the difficulty range of the module's final exam (see the contract, decision 13).
- Module final exams: 30 questions, +1 per correct answer, no negative marks. 0 to 14: can improve, no retry for 24 hours. 15 to 24: good, retry straight away. 25 to 30: excellent, the next module unlocks.
- Checkpoint exams (after Module 5 and later, not in version 1): 30 questions. 25 to 30 continues. 24 or below waits 24 hours. No straight retry.
- Progress never goes down. The progress bar increases only when a final exam or checkpoint is passed. Send progress to accounts/ with POST /internal/progress.
- Progress calculation (the number accounts/ stores and GET /progress returns)
- Sending progress to accounts/ (POST /internal/progress) and the AI's concept notes (POST /notes). Praharshini designs how notes are stored and shown.
- Endpoints you own: POST /chat, POST /submit, POST /quiz/start, POST /quiz/answer, POST /exam/start, POST /exam/submit

## Rules for this folder
- Answer keys are read on the server and never sent to the AI model.
- The no-full-solution rule must be enforced in code, not only in the prompt. Tests showed both test models leaked near-complete code when asked for a full solution.
- Keep tutor replies short. Target under 120 words.
- Explain keywords and concepts from the syllabus. Do not invent new modules.
- Do not change files in frontend/ or accounts/. Call accounts/ only through contracts/api.md.

## Known limit
The checker is not a sandbox. Say so in any reply that touches it.
