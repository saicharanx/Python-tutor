# ai/ (Person 2: Sai Charan)

Read the root CLAUDE.md first.

## You own
- Tutor system prompt and reply rules (short replies, hints, no full solutions)
- Calls to the AI provider (Groq under test), with the model name in one config file
- The checker: syntax check with ast.parse, then a run with `python -I` and a 3-second timeout
- Syllabus data for Modules 1 to 3
- Standard-check quizzes: 2 per module, 15 questions each. Understanding target 12 out of 15, informational only, no pass or fail. During the quiz, the level goes up after 2 correct in a row and down after a miss. The final level sets the difficulty of the module's final exam.
- Module final exams: 30 questions, +1 per correct answer, no negative marks. 0 to 14: can improve, no retry for 24 hours. 15 to 24: good, retry straight away. 25 to 30: excellent, the next module unlocks.
- Checkpoint exams (after Module 5 and later, not in version 1): 30 questions. 25 to 30 continues. 24 or below waits 24 hours. No straight retry.
- Progress never goes down. The progress bar increases only when a final exam or checkpoint is passed. Send progress to accounts/ with POST /internal/progress.
- The progress bar logic (the number the frontend shows)
- Sending the AI's concept notes to accounts/ (POST /notes). Praharshini designs how they are stored and shown.

## Rules for this folder
- Answer keys are read on the server and never sent to the AI model.
- The no-full-solution rule must be enforced in code, not only in the prompt. Tests showed both test models leaked near-complete code when asked for a full solution.
- Keep tutor replies short. Target under 120 words.
- Explain keywords and concepts from the syllabus. Do not invent new modules.
- Do not change files in frontend/ or accounts/. Call accounts/ only through contracts/api.md.

## Known limit
The checker is not a sandbox. Say so in any reply that touches it.
