# iwol config for this repository

## Checks

The orchestrator runs these itself; sub-agents never do.

- **Full check:** `actionlint`
- **Database:** none.
- **Finding every failure:** n/a, a single command.
- **Format:** none.
- **Format check:** none.
- **Post-wave refresh:** none.

## Generated files

None.

## Local services

None.

## Files to leave alone

Always untracked here. Never stage them: none.

## Review

- **Reviewer:** the Code Review workflow (`.github/workflows/review.yml`, calling this repo's reusable `claude-code-review.yml`), posting as `iwol-reviewer[bot]` (GraphQL login `iwol-reviewer`).
- **When it reviews:** by itself when a PR opens ready or is marked ready (not drafts, fork PRs or PRs opened by bots); after that only on request, so every later push needs a request.
- **Requesting a review:** comment a line starting `/review` on the PR, optionally `/review <model> <effort>` (default `opus`, an alias for the newest Opus model the workflow's Claude Code knows, at `xhigh`). A request that doesn't parse gets a bot reply starting "Claude review request failed:" and doesn't count.
- **In progress:** a running `Claude / review` check for an automatic review (GitHub's UI shows it as `Code Review / Claude / review`; match on `Claude / review`), or a 👀 on the request comment for a requested one. The 👀 stays, so only a later summary or failure reply ends a round; only bot comments newer than the request count. An automatic review also leaves a 👀 on the PR description, which stays from the first review on, so it never marks a review in progress.
- **Clean:** a summary comment from the bot that starts with the heading `### No issues found! 🚀`, then, each after a blank line, the reviewer's short note, one line per check the workflow's checks job ran, and `Reviewed with ... effort` followed by a last line `Reviewed head: <full sha>` (bare, no backticks) naming the PR's head. Older summaries, still on PRs reviewed before the workflow changed, start instead with the plain line `No issues found.` (no heading and no note, and a `Requested ... effort` footer); both count as clean. A summary naming an older head, or saying the review was partial, isn't clean: the workflow adds a line starting `Partial review:` when the reviewer didn't cover every changed file, and an older summary says so in the reviewer's own words. With `approve: true` in `review.yml`, a clean review whose checks all passed, run while every review thread the bot started was resolved, also submits a GitHub approval on the reviewed head, and a later review that isn't clean dismisses it; the summary stays the clean signal, and a PR can count as clean with no formal GitHub approval: when it's clean only because its findings were resolved, since the review that raised them gave none and dismissed any earlier one, when its summary lists a check as unavailable, since the workflow approves only when every check passed, or when its review ran while a review thread the bot started was still unresolved, since that review gave none even if the thread is resolved afterwards.
- **Findings:** inline comments from the bot, one per finding. The summary starts with an `### N issue(s) found` heading (`### 1 issue found`, `### 2 issues found`) in place of the clean heading, then lists each finding as one `- ` line; an older summary lists them with no heading, in place of "No issues found.". No priority badges; take priority from the wording. A summary line with no inline comment is still a finding.
- **Counting review rounds:** count the bot's summary comments, the ones with a `Reviewed head: ` line. GitHub records an empty `COMMENTED` review per inline comment, so GitHub's review count overcounts.
- **Resolved findings count as clean:** yes, once every thread from the head's summary has an outcome reply and is resolved, unless that summary says the review was partial.
- **Usage limit:** no message of its own. A failed, timed-out or cancelled review gets a bot comment starting "Claude review didn't finish" for a requested review, or a red `Claude / review` check for an automatic one. A red `Claude / request` or `Claude / runner / …` check with a skipped `Claude / review` also means the automatic review failed. A red `Claude / checks` stays out of it: the review still runs and reports the checks as unavailable. Each run uploads `claude-review-pr<PR>-<sha>-attempt<n>` (execution log, check output, the diffs it read) for 14 days. Retry an automatic review with `gh run rerun <run-id>` (all jobs, so the runner is picked again); retry a requested one with a new `/review`.
- **Review rules:** generic rules live in the workflow's prompt; this repo has no repo-specific rules.

## CI

- **Workflow:** `ci.yml`, gating job `check`.
- **Runs on:** hosted. CI uses GitHub-hosted runners only, so a queued job is only waiting for GitHub.
- **Recovering a stuck run:** none documented.
- **Waiting for CI:** after a push, a PR skill waits for the run on the pushed commit with this loop, run in the background. It prints `<run id> completed <conclusion>` when the run finishes, or each job's status after 20 minutes; a `null null null` there means no run started for the commit.

```bash
sha=$(git rev-parse HEAD)  # when merging: sha=$(gh pr view <PR> --json headRefOid --jq .headRefOid)
for attempt in $(seq 1 40); do
  run=$(gh run list --workflow ci.yml --commit "$sha" --json databaseId,status,conclusion --jq '.[0] | "\(.databaseId) \(.status) \(.conclusion)"')
  case "$run" in *" completed "*) echo "$run"; exit 0 ;; esac
  sleep 30
done
echo "not finished after 20 minutes: $run"
gh run view "${run%% *}" --json jobs --jq '.jobs[] | "\(.name): \(.status)"'
```

## Git and GitHub

- **Base branch:** `main`. The skills use it wherever they say `main`.
- **Worktrees:** `.claude/worktrees/`, which must be listed in the tracked `.gitignore` so fresh clones ignore it too.
- **Worktree setup:** none. A fresh worktree needs nothing: the checks use only tools installed on the machine.
- **Merge method:** squash. GitHub deletes the branch on merge, so the skills don't pass `--delete-branch`. Merge commits and rebase merges are disabled on GitHub.
- **PR title format:** conventional commit (`fix:` patch, `feat:` minor, `!` major), since release-please builds the changelog from PR titles. Keep `@word` out of PR titles: release-please links every `@word` in a title to that GitHub user's profile, which leaves a bogus link in the changelog.
- **Merge order:** none.
- **Auto-merge:** never.

## Releases

- **Release PR branch:** `release-please--branches--main`
- **Release workflow:** `release.yml`
- **Deploy workflow:** `release.yml`, whose only deploy step moves the `v<major>` tag once the release is created
- **Production version:** none
- **Migrations to check before a release:** none
- **Rollback:** none
