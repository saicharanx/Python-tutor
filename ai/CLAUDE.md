# ai/ (Person 2: Sai Charan)

Read the root CLAUDE.md first.

## You own
- Tutor system prompt and reply rules (short replies, hints, no full solutions)
- Calls to the AI provider (Groq under test), with the model name in one config file
- The checker: syntax check with ast.parse, then a run with `python -I` and a 3-second timeout
- Syllabus data for Modules 1 to 3
- The level-check quiz: starts at medium, up after 2 correct in a row, down after a miss
- The progress rule: a module counts as passed only when its quiz level is passed, counted once
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
