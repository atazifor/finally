# FinAlly Project - the Finance Ally

All project documentation is in the `planning` directory.

The key document is PLAN.md included in full below; the market data component has been completed and is summarized in the file `planning/MARKET_DATA_SUMMARY.md` with more details in the `planning/archive` folder. Consult these docs only when required. The remainder of the platform is still to be developed.

## Code review by Codex

Every commit is reviewed by Codex (a GPT model) through the git pre-commit hook in `.githooks/pre-commit` (enabled with `git config core.hooksPath .githooks`). High-severity findings block the commit.

- Run `git commit` with a 10-minute Bash timeout. The review takes a few minutes.
- If the commit is blocked, fix the reported high-severity findings and commit again. Address medium findings, or explain to the user why not.
- Never bypass the hook (`--no-verify`, `-n`, or changing `core.hooksPath`). Only the user may decide to skip a review.
- To review a whole branch or an old commit, use the `codex-reviewer` agent.

@planning/PLAN.md