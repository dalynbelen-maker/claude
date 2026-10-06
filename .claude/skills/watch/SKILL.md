---
name: watch
description: Follow a pull request until it is green and mergeable, watching CI, reviews and comments, and pushing fixes for failures. Use when asked to watch, babysit, monitor or drive a PR to green, or on /watch [PR number or URL].
---

# /watch: drive a PR to green

Follow one pull request until it is mergeable: CI green on the latest commit, no merge conflict, and no review thread waiting on you. Fix what you can, report what you can't.

## Pick the PR

- An argument (`/watch 42`, `/watch <url>`) names the PR.
- No argument: use the PR for the current branch (`gh pr view --json number,url`). If there is none, ask which PR to watch and stop.
- Check out the PR head locally (`gh pr checkout <n>`) and note whether you opened it. On a PR someone else opened, only make small, clearly in-scope fixes; ask the user before anything larger.

## Each round

Look at the whole PR on its current head, then act on every open item in this order.

```bash
gh pr view <n> --json state,mergeable,mergeStateStatus,headRefName,baseRefName,reviewDecision
gh pr checks <n>
gh api repos/{owner}/{repo}/pulls/<n>/comments   # inline review comments
gh pr view <n> --comments                          # top-level comments and reviews
```

1. **Merged or closed**: report it and stop watching.
2. **Merge conflict**: merge the base branch into the PR branch (`git fetch origin <base> && git merge origin/<base>`), resolve, regenerate lockfiles or generated files with the repo's own tooling, run the checks below, push. Never rebase, amend or force-push someone else's branch.
3. **CI red**:
   - Read the failing job's log (`gh run view <run-id> --log-failed`).
   - Reproduce the failure locally when you can, find the root cause, fix it, show the same check passing, push.
   - If the same check is red on the base branch too, it isn't this PR's fault. Say so once on the PR, and port an existing fix if there is one.
   - "Flaky" is not a root cause. Re-run a job at most once, and only when it died before any test ran (checkout, install, runner lost). A second failure is real.
4. **Review comments**:
   - Small, local asks (nits, renames, an added test, a one-function refactor) and bot findings: verify, fix, push, then reply on the thread and resolve it.
   - Larger asks (multi-file refactors, API or schema changes, open-ended design feedback): reply with a proposal and ask the user before changing code.
   - A question about intent: answer it from the diff.
5. **Everything green and mergeable**: tell the user the PR is waiting on reviewers and stop the loop, unless they asked you to keep watching for new reviews.

## Before every push

- Run the repo's own fast checks: lint, format, typecheck, and unit tests for the packages you changed.
- For a CI fix, show the original failure reproduced and then passing.
- Re-read your diff as a reviewer would. Keep it to what the failure or comment needs.
- Push only when everything is clean. One validated push beats three speculative ones.

## Waiting between rounds

- While checks are running, use `gh pr checks <n> --watch --fail-fast` instead of polling by hand.
- When nothing is running and you're waiting on people, re-check on a slow interval (for example `/loop 10m /watch <n>`), not a tight one.
- Stop when the user says stop, or when the PR is merged or closed.

## Never

- Skip, disable or quarantine a test to get green.
- Push an empty commit, or close and reopen the PR, to re-trigger CI.
- Rewrite history on a branch you didn't create.
- Approve or merge the PR yourself.

## Reporting

Keep the user posted only when something changes: a fix pushed, a blocker you can't resolve, a question only they can answer, or the PR going green. One or two lines each, with links. Don't narrate every poll.
