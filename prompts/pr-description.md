---
name: PR Description Draft
description: Fill-in prompt for drafting a pull request description from a diff and its intent.
version: 1
---

Write a pull request description for the following change.

Context:
- What problem this solves: {{problem}}
- What changed (summary of the diff): {{summary}}
- Anything a reviewer should test manually: {{test_notes}}

Requirements:
- Lead with why the change was made, not a restatement of the diff.
- Keep it under 200 words unless the change genuinely needs more.
- List a short test plan as checkboxes.
- No filler phrases, no puffery, plain language throughout.
