# code-review

A reusable GitHub Actions workflow, [`.github/workflows/claude-code-review.yml`](.github/workflows/claude-code-review.yml), that has Claude review your repository's pull requests. It runs your repository's own checks, reads the diff, posts each finding as an inline comment and a summary comment, and does it all as your own GitHub App. You call it from a short workflow file, a stub, in your repository.

- [What the workflow does](#what-the-workflow-does)
- [How it keeps secrets away from PR code](#how-it-keeps-secrets-away-from-pr-code)
- [Set it up](#set-it-up)
  - [Create the GitHub App](#create-the-github-app)
  - [Add the stub](#add-the-stub)
  - [Self-hosted example](#self-hosted-example)
  - [Let /iwol:setup write the stub](#let-iwolsetup-write-the-stub)
- [Inputs](#inputs)
- [Secrets](#secrets)
- [Approving clean reviews](#approving-clean-reviews)
- [Versioning](#versioning)
- [Adding and removing a secret](#adding-and-removing-a-secret)
- [Archive](#archive)

The comments in the workflow file are the authority on how each step works. This README covers what you need to use it.

## What the workflow does

The workflow is triggered only by `workflow_call`, so it does nothing until a stub in your repository calls it.

**When it reviews.**

- **By itself.** When a pull request opens, or each time it's marked ready for review, it reviews the PR if the PR is not a draft, its branch is in the same repository, and it was not opened by a bot. Such a review replaces a review still running for the PR.
- **On request.** It reviews when someone with write or admin access on the repository comments on the PR with a line starting `/review`. A request reaches any PR whose branch is in the repository, including the ones the automatic review skips: drafts, PRs opened by Dependabot or another bot, PRs into a branch other than the one your stub names, and closed or merged PRs. It's the usual way to get a Dependabot PR reviewed. The workflow accepts a comment from the repository's owner, a member of the organisation that owns it, or a collaborator on it (a comment whose `author_association` is `OWNER`, `MEMBER` or `COLLABORATOR`), then checks the commenter's repository permission itself. An owner, member or collaborator without write or admin access gets a reply starting `Claude review request failed:` from the workflow's first job, and nothing else runs. A `/review` from anyone else is ignored, with no reply. The line may name a model, an effort and a credential, in any order: `/review`, `/review sonnet`, `/review high`, `/review opus max`, `/review max opus` and `/review api-key opus` all work. If it cannot parse the request it replies with a comment starting `Claude review request failed:` and does not review.
- **Not with a mention.** Only a `/review` line requests a review. If you set `review-app-slug`, a comment that mentions `@<slug>` from any owner, organisation member or collaborator, with or without write access, also reaches the workflow, so a thank-you to the bot gets a 🎉 reaction. A comment with a `/review` line is reviewed whether or not it also mentions or thanks the bot, so a thank-you with a `/review` line gets both. Without `review-app-slug`, nothing gets the 🎉.
- **Never for forks.** A pull request whose branch lives in a fork is never reviewed, automatically or on request.

**What a review does.**

1. It reacts 👀 to whatever started the review: the PR description for the automatic review, or the request comment. The 👀 stays after the review finishes.
2. It runs your repository's own checks (the `checks` input) and gives the reviewer their results, so it reads real lint, type and test failures instead of guessing at them.
3. The reviewer reads the diff and posts one inline comment per finding, on the line it concerns.
4. The workflow posts a summary comment, assembled from the reviewer's structured output rather than written freehand:

   ```text
   ### 2 issues found

   - <one line per finding>
   - <one line per finding>

   Partial review: <only when the reviewer did not cover every changed file>

   <the reviewer's short note>

   <one line per check result>

   Reviewed with <model> at <effort> effort
   Reviewed head: <sha>
   ```

   A review with no findings has the heading `### No issues found! 🚀` instead.

**Models, effort and credentials.** A review uses `default-model`, `default-effort` and `default-credential` unless the request names others. For the automatic review, the request is the PR description's last line starting `/review`, read the same way as a comment's (`/review haiku low`, `/review opus max`), and anything it leaves out uses the defaults. A description line the workflow can't parse is ignored, with only a warning in the run log, and the review uses the defaults. A model may be an alias (`opus`, `sonnet`, `haiku`, `fable` or `best`) or a full model ID. An alias resolves to the newest model of that name known to the Claude Code inside the pinned [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action). Dependabot keeps that pin current in this repository, which is how an alias moves on to a newer model without you changing anything.

A request may also name the Claude credential the review runs on: `oauth` for `CLAUDE_CODE_OAUTH_TOKEN`, or `api-key` for `ANTHROPIC_API_KEY`, as in `/review api-key` or `/review oauth opus max`. Your stub must pass that secret: a comment asking for one it doesn't pass gets a reply starting `Claude review request failed:`, and a PR description that does so has its credential ignored, with a warning in the run log, while its model and effort still apply. The review is handed only the credential it runs on, so the other is never used, not even by the action's inline-comment classifier, which calls the API with the key whenever it has one. To offer the choice, set both secrets and pass both in your stub.

## How it keeps secrets away from PR code

A pull request's code is whatever its author wrote, so the workflow never runs it anywhere a secret can be reached.

- **The checks job runs the PR's code with nothing to steal.** Your `checks` steps (install scripts, linters, compilers, tests) run in a job that has no secrets and only a read-only `GITHUB_TOKEN`, which it does not persist into the checkout. It saves the check results as an artifact.
- **The review job runs no PR code.** It starts on a fresh machine (on a GitHub-hosted runner, or on a self-hosted one only when each job gets a new VM or container that is thrown away after the job and cannot reach its host or the machines other jobs run on; see [Self-hosted example](#self-hosted-example)), downloads only the check results, and moves git's directory out of the checkout, which keeps the app token the action writes into `.git/config` out of the reviewer's reach, since the reviewer can read anything in the checkout. It reads the PR as text and never executes it. Only the jobs that run no PR code (the request, review and report-failure jobs) hold secrets, and the Claude credential reaches only the review job.
- **Your stub passes secrets by name.** The stub lists each secret the workflow needs, never `secrets: inherit`, so the workflow never sees your other repository or organisation secrets.
- **It posts as your app.** The review comments, reactions and approvals come from your own GitHub App, `<app slug>[bot]`, not from `claude[bot]`, and the app has only the permissions you gave it.

## Set it up

You need a GitHub App for the review to post as, a stub workflow that calls this workflow, a Node version file in your repository (`.nvmrc` unless you set `node-version-file`, plus a `packageManager` field in `package.json` if you use pnpm), and three values stored in your repository: the app's client ID, its private key, and a Claude credential.

### Create the GitHub App

1. Create a GitHub App, under your user or organisation's **Settings → Developer settings → GitHub Apps → New GitHub App**. Its name gives it a slug, its login without `[bot]`, and reviews will be posted as `<app slug>[bot]`. You can leave the webhook inactive.
2. Under **Repository permissions**, set **Pull requests** and **Issues** to **Read and write**. The review needs them to post comments and reactions, and to submit and dismiss approvals.
3. Create the app, note its **Client ID**, and generate a **private key** (a `.pem` file).
4. Install the app on the repository you want reviewed.
5. Store the client ID and private key in that repository:

   ```sh
   gh variable set REVIEW_APP_CLIENT_ID --body "<the app's client ID>"
   gh secret set REVIEW_APP_PRIVATE_KEY < path/to/private-key.pem
   ```

6. Store one or both of the two Claude credentials (see [Secrets](#secrets)). With both, a `/review` line can choose which one a review runs on:

   ```sh
   gh secret set CLAUDE_CODE_OAUTH_TOKEN   # a token from `claude setup-token`
   # and/or
   gh secret set ANTHROPIC_API_KEY
   ```

If you want a thank-you to the app to get a 🎉, pass the app's slug as `review-app-slug`. Reviews are requested with `/review` either way.

### Add the stub

Save this as `.github/workflows/review.yml` in your repository and replace the `checks` lines with your own:

```yaml
name: Code Review

on:
  pull_request:
    types: [opened, ready_for_review]
    branches: [main]
  # A comment line starting "/review [model] [effort] [oauth|api-key]" asks for a fresh review
  issue_comment:
    types: [created]

jobs:
  review:
    # GitHub shows the called workflow's checks as "Code Review / Claude / <job>"
    name: Claude
    # The workflow posts through the review app's own token
    permissions:
      contents: read
    uses: it-works-on-local-dev/code-review/.github/workflows/claude-code-review.yml@v1
    # Only the secrets the workflow declares, never secrets: inherit
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      # and/or an API key; with both, /review oauth or /review api-key picks one:
      # ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      REVIEW_APP_PRIVATE_KEY: ${{ secrets.REVIEW_APP_PRIVATE_KEY }}
    with:
      review-app-client-id: ${{ vars.REVIEW_APP_CLIENT_ID }}
      checks: |
        run_step setup install npm ci
        run_step check lint npm run lint
        run_step check typecheck npx tsc --noEmit
        run_step check test npm test
```

The job's name, `Claude`, is why GitHub lists the workflow's checks as `Code Review / Claude / <job>`. Open a pull request to see the first review.

### Self-hosted example

The stub above runs on GitHub-hosted `ubuntu-latest`. To run the checks and review jobs on self-hosted runners, add a job to your stub that works out which runner to use and pass its result as `runs-on`. This is how this repository's own organisation does it, with a runner picker in its `it-works-on-local-dev/runners` repository that chooses between its self-hosted runners and GitHub-hosted ones.

Self-hosted runners must give each job a freshly provisioned VM or container that is thrown away after the job and cannot reach its host or the machines other jobs run on. That means no mounted host Docker socket, no privileged mode, and no host namespaces or mounts, which rules out Actions Runner Controller runner pods in dind mode; otherwise, use a new VM for each job, for example from an autoscaler that creates a new VM for each just-in-time runner. Registering runners with `--ephemeral` or just-in-time on reused hardware is not enough. The checks job runs the PR's code, and the review job, sent to the same runners, holds the Claude credential and the app token, so on a reused machine, or from a container that can reach its host, code started by the checks job could outlive it and reach the review job's tokens.

```yaml
jobs:
  runner:
    name: Runner
    # The workflow's request job's filter, so the picker starts only when a review might follow
    if: >-
      (github.event_name == 'pull_request' &&
        github.event.pull_request.draft == false &&
        github.event.pull_request.head.repo.full_name == github.repository &&
        github.event.pull_request.user.type != 'Bot') ||
      (github.event_name == 'issue_comment' &&
        github.event.issue.pull_request &&
        (contains(github.event.comment.body, '/review') ||
          contains(github.event.comment.body, '@my-review-app')) &&
        contains(fromJSON('["OWNER", "MEMBER", "COLLABORATOR"]'), github.event.comment.author_association))
    permissions: {}
    uses: it-works-on-local-dev/runners/.github/workflows/pick-runner.yml@main
    secrets:
      RUNNER_STATUS_TOKEN: ${{ secrets.RUNNER_STATUS_TOKEN }}

  review:
    name: Claude
    needs: runner
    permissions:
      contents: read
    uses: it-works-on-local-dev/code-review/.github/workflows/claude-code-review.yml@v1
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      REVIEW_APP_PRIVATE_KEY: ${{ secrets.REVIEW_APP_PRIVATE_KEY }}
    with:
      review-app-client-id: ${{ vars.REVIEW_APP_CLIENT_ID }}
      review-app-slug: my-review-app
      # The picker's linux output is already JSON: a label string or an array of labels
      runs-on: ${{ needs.runner.outputs.linux }}
      checks: |
        run_step setup install npm ci
        run_step check lint npm run lint
```

The `name`, `on` and the rest of the file are as in the hosted stub. The `runner` job's `if:` copies the workflow's own filter for the events it acts on, with the example's `my-review-app` slug in place of `review-app-slug`, so the picker doesn't start for every comment on every issue and PR. When the picker is skipped, the review job, which needs it, is skipped too, as intended. If the picker fails, the review is skipped with no reply to the `/review`. With this shape the picker's checks show as `Code Review / Runner / …` beside `Code Review / Claude / …`. Any job of your own that outputs a JSON label string or array works the same way, and `RUNNER_STATUS_TOKEN` belongs to the picker, not to this workflow.

### Let /iwol:setup write the stub

The `iwol` Claude Code plugin, from [it-works-on-local-dev/skills](https://github.com/it-works-on-local-dev/skills), has a setup skill, `/iwol:setup`. It asks which reviewer your repository uses and writes this stub at `@v1` with your checks, secrets and inputs filled in. You can also write the stub by hand as above.

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `package-manager` | no | `npm` | `npm` or `pnpm`. pnpm is set up before Node, at the version named by `package.json`'s `packageManager` field, which must be set; without it the checks job stops before any check runs. |
| `node-version-file` | no | `.nvmrc` | The file Node's version is read from. It must exist in your repository: the checks job always sets Node up from it, and stops before any check runs without it. |
| `exclude-paths` | no | none | Newline-separated paths left out of every diff and never read by the reviewer, for generated output such as `dist/` or `src/generated/`. |
| `checks` | yes | | Shell run in the checks job after Node is set up, one step per line, written `run_step <setup\|check> <name> <command...>`. A `setup` step adds a summary line only when it fails; a `check` step always adds a pass or FAIL line. A name may use only letters, digits, `.`, `_` and `-`. |
| `review-app-client-id` | yes | | The review app's client ID, normally the `REVIEW_APP_CLIENT_ID` repository variable. |
| `review-app-slug` | no | empty | The app's login without `[bot]`. With it set, a comment mentioning `@<slug>` also reaches the workflow, so a thank-you to the bot gets a 🎉. Only a `/review` line starts a review, with or without it. |
| `runs-on` | no | `"ubuntu-latest"` | A JSON label string or JSON array of labels for the machine the checks and review jobs run on. In YAML, quote the JSON: `runs-on: '"ubuntu-24.04"'` or `runs-on: '["self-hosted", "linux"]'`. An expression whose output is already JSON, such as the self-hosted example's `${{ needs.runner.outputs.linux }}`, is passed as it is. |
| `default-model` | no | `opus` | The model used when a request names none: an alias or a full model ID. |
| `default-effort` | no | `xhigh` | The effort used when a request names none. |
| `default-credential` | no | `auto` | The Claude credential used when a request names none: `oauth` (`CLAUDE_CODE_OAUTH_TOKEN`), `api-key` (`ANTHROPIC_API_KEY`), or `auto`, which takes whichever of the two your stub passes, and the API key when it passes both, as the action itself would. If it names a secret your stub doesn't pass, every review that names no credential fails. |
| `approve` | no | `false` | Approve the reviewed head when a review is clean, and dismiss the bot's earlier approvals when one is not. See [Approving clean reviews](#approving-clean-reviews). |

## Secrets

Pass each secret by name in the stub's `secrets:` block.

| Secret | Required | Meaning |
| --- | --- | --- |
| `CLAUDE_CODE_OAUTH_TOKEN` | at least one of the two | A Claude Code OAuth token, for example from `claude setup-token`. |
| `ANTHROPIC_API_KEY` | at least one of the two | An Anthropic API key. |
| `REVIEW_APP_PRIVATE_KEY` | yes | The review app's private key. |

`CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY` are each optional to GitHub, but the workflow needs one of them: with neither, it fails early with `Set CLAUDE_CODE_OAUTH_TOKEN or ANTHROPIC_API_KEY`. Pass both to let each review request choose between them, as described under [What the workflow does](#what-the-workflow-does).

## Approving clean reviews

With `approve: true`, a clean review also submits a GitHub approval on the head it reviewed, and a later review that is not clean dismisses the bot's earlier approvals. A review is clean when:

- it has no findings,
- it covered every changed file,
- no check failed or is missing,
- every review thread the bot started is resolved, and
- the PR is still open and not a draft at the reviewed head.

**Never let this approval count toward required reviews, on any plan.** Whoever writes a pull request can obtain it: the reviewer reads the PR's own diff, and the checks are the PR's own scripts, so the author controls everything the verdict rests on. The author can even pick the automatic review's model, effort and credential, with a `/review` line in the PR description. Treat the approval as a visible signal only. Where your branch protection or rulesets require approvals, also require one from a person or team, through Code Owners or a ruleset's required reviewers. Do not remove the app's permissions instead: it needs them to post the review.

## Versioning

Releases are tagged `vX.Y.Z` by [release-please](https://github.com/googleapis/release-please) from conventional-commit PR titles, and each release's changes are listed in [CHANGELOG.md](CHANGELOG.md).

- **`@v1` moves.** The `v1` tag moves to each compatible release, so a stub at `@v1` receives fixes and features without changing.
- **Breaking changes get a new major tag.** A breaking change is released under a `v2` tag, which no `@v1` stub follows. Moving to it is a change to your stub.
- **You can pin.** A stub may call a full tag such as `@v1.2.3` instead, and move it when you choose.

This organisation's own repositories call the workflow at `@main`, so they see a merged change before it is released.

## Adding and removing a secret

This section is for changing the workflow itself. Stubs pass secrets by name, which has two consequences: a secret added to the workflow reaches no caller until that caller's stub passes it, and GitHub will not start the workflow for a stub that passes a secret the workflow does not declare.

**Adding a secret.** Declare it as optional (`required: false`), so stubs that do not pass it yet keep working. Then add it to the stub in every place one is written down, in step with the workflow:

- the stub in the workflow's header comment,
- this repository's [`.github/workflows/review.yml`](.github/workflows/review.yml),
- the tables and stubs in this README, and
- the stub template in the setup skill of [it-works-on-local-dev/skills](https://github.com/it-works-on-local-dev/skills), which must change in step with this repository.

**Removing or renaming a secret** is the reverse, and riskier. A stub that still passes the old name stops working the moment `v1` moves to the release that dropped it. Either keep the old name declared as optional until every stub has stopped passing it, or make the removal a major release. A rename is an addition followed, later, by a removal.

## Archive

The workflow's history before this repository, including its 31 pull requests, is in the private [it-works-on-local-dev/tooling](https://github.com/it-works-on-local-dev/tooling) repository.
