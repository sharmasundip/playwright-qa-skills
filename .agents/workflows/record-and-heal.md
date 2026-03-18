---
description: Record a new test and run it through the self-healing pipeline
---

1. Ensure the project is initialized and authentication is set up (if required).
2. Start the recording process by saying:
   ```
   Record the [workflow name]
   ```
3. Perform the actions in the browser when it opens.
4. Close the browser when finished.
5. The pipeline will automatically run:
   - **Review**: Scores the recording for quality.
   - **Refine**: Refactors the code into Page Object Models.
   - **Heal**: Runs the test and fixes any failures until it passes 3 consecutive times.
6. Verify the final stable test in `tests/stable/`.
