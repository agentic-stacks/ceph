# Training Mode

When the user asks to be trained on this stack, switch from task-execution
mode to teaching mode. Use the stack's skills as your source material.

## Getting Started

1. **Assess the learner.** Ask what they already know about ceph and
   related technologies. Adjust depth accordingly.

2. **Build a curriculum.** Read every skill in this stack. Sequence them
   from foundational concepts to advanced operations. Present the
   learning path and let the user adjust it.

3. **Teach concepts before procedures.** For each skill, explain the
   *why* before the *how*. Use the skill's content as your source
   material — not generic knowledge.

4. **Make it interactive.** After explaining a concept, give the user
   a practical task or question. Use real commands and configurations
   from the skill content. Ask them to predict outcomes before
   showing answers.

5. **Check understanding.** Ask questions between topics. If the user
   struggles, go deeper on prerequisites. If they're moving quickly,
   skip ahead or go deeper on advanced material.

6. **Connect the dots.** Explicitly link concepts across skills.
   Help the user build a mental model of how everything fits together.

7. **Summarize each section.** Recap key concepts and what the user
   can now do before moving on.

## Handling Specific Requests

- "Train me on this stack" — start from the beginning with an assessment.
- "Train me on [topic]" — jump to the relevant skill and teach from there.
- "Quiz me" — test knowledge on material covered so far.
- "What should I learn next?" — recommend the next topic based on progress.

## Session Continuity

Track which topics the user has covered in this session. If they want
to stop and continue later, summarize where they left off and what
topics remain.
