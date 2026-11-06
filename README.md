# code-review

A reusable GitHub Actions workflow, [`.github/workflows/code-review.yml`](.github/workflows/code-review.yml), that has Claude or OpenAI Codex review your repository's pull requests. It runs your repository's own checks, reads the diff, posts each finding as an inline comment and a summary comment, and does it all as your own GitHub App. You call it from a short workflow file, a stub, in your repository.

- [What the workflow does](#what-the-workflow-does)
- [How it keeps secrets away from PR code](#how-it-keeps-secrets-away-from-pr-code)
- [Checking the output before it's posted](#checking-the-output-before-its-posted)
- [Reviewing with Codex](#reviewing-with-codex)
  - [Codex models and efforts](#codex-models-and-efforts)
  - [Codex credentials](#codex-credentials)
  - [What OpenAI permits](#what-openai-permits)
  - [What Codex can and can't do in a review](#what-codex-can-and-cant-do-in-a-review)
  - [Runners for Codex](#runners-for-codex)
  - [What hasn't been tested](#what-hasnt-been-tested)
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

The comments in the workflow file are the authority on how each step works. This README covers what you need to use it.

## What the workflow does

The workflow is triggered only by `workflow_call`, so it does nothing until a stub in your repository calls it.

**When it reviews.**

- **By itself.** When a pull request opens, or each time it's marked ready for review, it reviews the PR if the PR is not a draft, its branch is in the same repository, and it was not opened by a bot. Such a review replaces a review still running for the PR.
- **On request.** It reviews when someone with write or admin access on the repository comments on the PR with a line starting `/review`. A request reaches any PR whose branch is in the repository, including the ones the automatic review skips: drafts, PRs opened by Dependabot or another bot, PRs into a branch other than the one your stub names, and closed or merged PRs. It's the usual way to get a Dependabot PR reviewed. The workflow accepts a comment from the repository's owner, a member of the organisation that owns it, or a collaborator on it (a comment whose `author_association` is `OWNER`, `MEMBER` or `COLLABORATOR`), then checks the commenter's repository permission itself. An owner, member or collaborator without write or admin access gets a reply starting `Review request failed:` from the workflow's first job, and nothing else runs. A `/review` from anyone else is ignored, with no reply. The line is `/review [claude|codex] [model] [effort] [sub|api]`: an engine, a model, an effort and a credential, each optional and in any order. `/review`, `/review sonnet`, `/review high`, `/review opus max`, `/review max opus`, and `/review api opus` all work. The Codex forms are `/review codex` and `/review gpt-6.1-sol ultra` (see [Reviewing with Codex](#reviewing-with-codex)). If it cannot parse the request it replies with a comment starting `Review request failed:` and does not review.
- **Not with a mention.** Only a `/review` line requests a review. If you set `review-app-slug`, a comment that mentions `@<slug>` from any owner, organisation member or collaborator, with or without write access, also reaches the workflow, so a thank-you to the bot gets a 🎉 reaction. A comment with a `/review` line is reviewed whether or not it also mentions or thanks the bot, so a thank-you with a `/review` line gets both. Without `review-app-slug`, nothing gets the 🎉.
- **Never for forks.** A pull request whose branch lives in a fork is never reviewed, automatically or on request.

**What a review does.**

1. It reacts 👀 to whatever started the review: the PR description for the automatic review, or the request comment. The 👀 stays after the review finishes.
2. It runs your repository's own checks (the `checks` input) and gives the reviewer their results, so it reads real lint, type and test failures instead of guessing at them.
3. The reviewer reads the diff and returns its findings as structured output. The workflow checks every comment body (see [Checking the output before it's posted](#checking-the-output-before-its-posted)) and, only when they're clean, posts one inline comment per finding, on the line it concerns, on the reviewed head. A finding on a line GitHub won't accept for an inline comment isn't lost: it stays in the summary.
4. The workflow posts a summary comment, assembled from the reviewer's structured output rather than written freehand:

   ```text
   ### 2 issues found

   - <one line per finding>
   - <one line per finding>

   Partial review: <only when the reviewer did not cover every changed file>

   <the reviewer's short note>

   <one line per check result>

   Reviewed by Claude with <model> at <effort> effort
   Reviewed head: <sha>
   ```

   A review with no findings has the heading `### No issues found! 🚀` instead. A Codex review's footer reads `Reviewed by Codex with ...`.

**When a review fails.** An automatic review that fails leaves a red `Code Review / Reviewer / Claude` or `Code Review / Reviewer / Codex` check (named after the engine), with no comment (unless its summary is withheld, below), and one that fails before the review starts, such as a missing credential, leaves a red `Code Review / Reviewer / request` check, with the review check skipped, so a successful re-run leaves nothing stale behind. A requested review that fails, times out or is cancelled gets a reply starting `Review didn't finish`. When a Claude review's own run ended in an error, such as `Credit balance is too low` or a usage limit, which a retry won't fix, the run shows that error as a `Claude review failed` annotation and in its summary, and the reply quotes it: `Claude stopped with an error: <error>`. The error is cut to one line of at most 200 characters, with any long token-like word redacted. A Codex review's reply doesn't quote Codex's error, because the Codex action reports none; it says `Codex's run failed; its error is in the run log`, and on a review that ran on `sub` it adds a hint to run `codex login` and reset `CODEX_AUTH_JSON` (see [What OpenAI permits](#what-openai-permits)). Without one, the reply says the review may have hit the 60-minute limit or been replaced by a newer one. When the review summary contains a credential-shaped string, the summary is withheld, the review log isn't uploaded, and the workflow posts one notice starting `Review output withheld:` with a link to the run; a requested review gets that instead of `Review didn't finish`, and an automatic one gets it along with the red check named after the engine (`Code Review / Reviewer / Claude` or `Code Review / Reviewer / Codex`), and the notice stays on the PR after a successful re-run. No inline comment is posted in that case either.

**The review log.** Each review run uploads its review log, unless the summary is withheld or a credential-shaped string turns up in the log itself (the summary still posts in that case if it is clean), as an artifact named `code-review-pr<PR>-<sha>-attempt<n>`: the PR number, the reviewed head (the sha on the summary's `Reviewed head:` line) and the run attempt. For a Claude review the artifact holds the execution log (every tool call, then a result block listing the permission denials and the total cost); for a Codex review it holds `codex-final-message.json`, Codex's structured final message, instead. Either way it also holds the check output, and the diffs and PR conversation the reviewer read, so a suspected reviewer miss can be investigated. It is kept for 14 days.

**Engine, model, effort and credential.** A request may name any of the four, in any order, each optional.

- **Engine.** `claude` or `codex`. With no engine word, a model word implies its engine: `gpt-` models mean Codex, and `opus`, `sonnet`, `haiku`, `fable`, `best` and `claude-` models mean Claude. With neither, the review uses the `default-reviewer` input, `claude` unless you set `codex`. An engine word that contradicts the model, two engine words, or two models fail the request.
- **Model.** For Claude, an alias (`opus`, `sonnet`, `haiku`, `fable` or `best`) or a full model ID. An alias resolves to the newest model of that name known to the Claude Code inside the pinned [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action). Dependabot keeps that pin current in this repository, which is how an alias moves on to a newer model without you changing anything. For Codex, a `gpt-` model; see [Codex models and efforts](#codex-models-and-efforts).
- **Effort.** Claude takes `low`, `medium`, `high`, `xhigh` or `max`. Codex takes those and `ultra`; `ultra` on a Claude review fails the request.
- **Credential.** `sub` or `api`, resolved per engine: `sub` is `CLAUDE_CODE_OAUTH_TOKEN` or `CODEX_AUTH_JSON`, and `api` is `ANTHROPIC_API_KEY` or `OPENAI_API_KEY`. Your stub must pass that secret. The old words `oauth` and `api-key` fail the request with a hint to `sub` and `api`.

Anything a request leaves out uses the engine's own defaults: `default-claude-model`, `default-claude-effort` and `default-claude-credential` for Claude, and the three `default-codex-*` inputs for Codex. Each engine reads only its own. A credential of `auto` takes whichever of the engine's two secrets your stub passes, and the API key when it passes both. To offer the choice, set both and pass both in your stub.

The `default-reviewer` must have a credential in your stub, or the run fails early. A request that names an engine whose credential your stub doesn't pass gets a reply naming that engine's two secrets, and a comment naming a credential your stub doesn't pass is refused the same way. The review is handed only the credential it runs on, so the other is never used, not even by the action's inline-comment classifier, which calls the API with the key whenever it has one.

For the automatic review, the request is the PR description's last line starting `/review`, read the same way as a comment's (`/review haiku low`, `/review codex max`; if your stub passes no secret for the named engine, that line is ignored with a warning and the defaults apply). A comment's failed request gets a `Review request failed:` reply; the description's line is lenient: a line the workflow can't parse is ignored, with only a warning in the run log, and the review uses the defaults. A credential in the description that your stub doesn't pass is ignored with a warning too, and the line's engine, model and effort still apply (the review then uses that engine's default credential, and fails if your stub doesn't pass its secret).

## How it keeps secrets away from PR code

A pull request's code is whatever its author wrote, so the workflow never runs it anywhere a secret can be reached.

- **The checks job runs the PR's code with nothing to steal.** Your `checks` steps (install scripts, linters, compilers, tests) run in a job that has no secrets and only a read-only `GITHUB_TOKEN`, which it does not persist into the checkout. It saves the check results as an artifact.
- **The review job runs no PR code.** It starts on a fresh machine (on a GitHub-hosted runner, or on a self-hosted one only when each job gets a new VM or container that is thrown away after the job and cannot reach its host or the machines other jobs run on; see [Self-hosted example](#self-hosted-example)), downloads only the check results, and moves git's directory out of the checkout, which keeps the app token the action writes into `.git/config` out of the reviewer's reach, since the reviewer can read anything in the checkout. It reads the PR as text and never executes it. Only the jobs that run no PR code (the request, review and report-failure jobs) hold secrets, and the engine's credential reaches only the review job, and only the one the review runs on.
- **Codex never holds a credential it can print, or a GitHub token.** The Codex process gets no GitHub token, and no Claude secret reaches its step. On `api` the key is held by a local proxy the Codex action starts, so the Codex process never sees it. On `sub` the stored login is written to a directory of its own under the runner's temp directory, its tokens are masked in the log, and the directory is removed when the Codex step is over, finished or not. Codex's commands run in a read-only sandbox with no network, and on `sub` that sandbox also denies them the Codex home, so a PR that steers Codex into printing `auth.json` gets nothing. See [Codex credentials](#codex-credentials) and [What Codex can and can't do in a review](#what-codex-can-and-cant-do-in-a-review).
- **Nothing is posted before it's checked.** The summary and every inline comment body are scanned for credential-shaped strings first, for both engines; see [Checking the output before it's posted](#checking-the-output-before-its-posted).
- **Your stub passes secrets by name.** The stub lists each secret the workflow needs, never `secrets: inherit`, so the workflow never sees your other repository or organisation secrets.
- **It posts as your app.** The review comments, reactions and approvals come from your own GitHub App, `<app slug>[bot]`, not from `claude[bot]`, and the app has only the permissions you gave it.

## Checking the output before it's posted

For both engines, before anything from a review is posted, the workflow checks the summary and every inline comment body for token-shaped strings (a JWT, or a key with a known prefix such as `sk-`, `sk-ant-`, `sk-proj-`, `ghp_` or `github_pat_`). On a match nothing is posted, no inline comment and no summary, and no artifact is kept. One notice on the PR says the output was withheld and links to the run; its text never includes the matched string. See [When a review fails](#what-the-workflow-does) for what that does to the red check and the reply. The review log is checked the same way, but a match there only stops the upload and leaves a clean summary alone.

## Reviewing with Codex

The Codex engine reviews the same way Claude does, as far as you can see: it runs after your checks, reads the diff, and the workflow posts its findings as inline comments and a summary, as your app. Ask for it with `/review codex`, a `gpt-` model, or `default-reviewer: codex`; the request grammar, the `default-codex-*` inputs and the secrets are documented under [What the workflow does](#what-the-workflow-does), [Inputs](#inputs) and [Secrets](#secrets), and aren't repeated here.

### Codex models and efforts

Codex takes these models, and the workflow accepts any `gpt-` slug:

| Model | Note |
| --- | --- |
| `gpt-6.1-sol` | The default, and the top priority in Codex's catalogue at the time of writing. |
| `gpt-6-astra` | |
| `gpt-6-sol` | |
| `gpt-6-luna` | |
| `gpt-5.6-sol` | |
| `gpt-5.6-terra` | |
| `gpt-5.6-luna` | |
| `gpt-5.5` | |

Codex has no alias layer like Claude's `opus`, so `default-codex-model` is a concrete slug, and moving to a newer model is a change you make by hand. The Codex CLI itself is pinned by version inside the workflow. A weekly workflow in this repository, [`.github/workflows/bump-codex.yml`](.github/workflows/bump-codex.yml), opens a PR titled `fix(deps): bump the Codex CLI from X to Y` when npm has a newer release, so a merged bump cuts a patch release, and adds a line to the PR body when the catalogue's top-priority model differs from the default.

Codex's efforts are `low`, `medium`, `high`, `xhigh`, `max` and `ultra`. `ultra` is Codex-only: on a Claude review it fails the request.

### Codex credentials

Pass one or both of the Codex engine's two secrets; with both, `/review sub` or `/review api` chooses.

- **`OPENAI_API_KEY` (`api`) is the recommended route.** Store an OpenAI API key with `gh secret set OPENAI_API_KEY`. The Codex action runs a local proxy that holds the key, and the Codex process never sees it.
- **`CODEX_AUTH_JSON` (`sub`) is the stored-login route**, for a ChatGPT subscription. Log in with the Codex CLI's credentials store set to a file, so the login lands in a file rather than your system keychain: set `cli_auth_credentials_store = "file"` in `~/.codex/config.toml`, then run `codex login`. Then store the `auth.json` from the Codex home directory (`~/.codex` unless you changed it), compacted:

  ```sh
  jq -c . ~/.codex/auth.json | gh secret set CODEX_AUTH_JSON
  ```

  Store it compacted, because GitHub masks the secret's every line in the log, and a pretty-printed file makes it mask every `{` and `}` in the run's output.

For a `sub` review the workflow writes the secret into a directory under the runner's temp directory for that run only, masks its tokens in the log, and removes the directory afterwards, whether the review finished or not. Codex may refresh the login while it runs, but nothing writes a refreshed login back to the secret.

### What OpenAI permits

As OpenAI documents it:

- API keys are the route for automation.
- The stored-login file is an advanced route for trusted private automation, and never for public or open-source repositories. Use `CODEX_AUTH_JSON` in a private repository only.
- One `auth.json` serves one serialised job stream: don't share it between concurrent jobs or machines.
- Codex refreshes the token bundle after about 8 days. This workflow keeps no refreshed file, so expect the stored login to go stale: when it does, run `codex login` again and reset the secret. A `Review didn't finish` reply for a review that ran on `sub` says so.
- The sanctioned subscription route for automation is an Enterprise access token, which this workflow doesn't support.

### What Codex can and can't do in a review

- **It reads, through shell commands, in a read-only sandbox.** It reads the checkout, the diffs, the base branch's review rules and the check results with commands such as `cat`, `sed` and `grep`. It can't change a file or reach the network. On `sub`, its commands run under a profile, `read-only-without-login`, that also denies them the Codex home.
- **It gets no GitHub token.** It can't comment, react or read anything on GitHub itself; the diffs and the PR's conversation are written to files for it.
- **It loads no instructions from the checkout.** A PR's `AGENTS.md`, `AGENTS.override.md`, skills and `.codex/` directory can't steer it. The base branch's review rules are the only rules it reads.
- **The workflow posts its findings.** Codex returns its findings as structured output; the workflow posts one inline comment per finding, on the reviewed head, once the output has passed the check above. A finding on a line GitHub won't accept for an inline comment stays in the summary. Claude's findings are posted the same way.
- **If its run fails**, the reply says `Codex's run failed; its error is in the run log`; Codex's own error isn't quoted.

### Runners for Codex

- The Codex action's default `drop-sudo` takes `sudo` away for the rest of the review job, so nothing after the Codex step can use it.
- A Codex review on the default `runs-on` of `"ubuntu-latest"` runs its review job on `ubuntu-24.04`, because the Codex action can't yet drop `sudo` on the Ubuntu 26.04 that `ubuntu-latest` now means ([openai/codex-action#188](https://github.com/openai/codex-action/issues/188)). Checks still run on `ubuntu-latest`.
- A self-hosted runner for a Codex review needs `setfacl` and unprivileged user namespaces, in addition to the isolation described in [Self-hosted example](#self-hosted-example).

### What hasn't been tested

At the time of writing, the `api` route has had no real run yet, while the `sub` route has. The first review with `OPENAI_API_KEY` may turn something up.

## Set it up

You need a GitHub App for the review to post as, a stub workflow that calls this workflow, a Node version file in your repository (`.nvmrc` unless you set `node-version-file`, plus a `packageManager` field in `package.json` if you use pnpm), and three values stored in your repository: the app's client ID, its private key, and a credential for the engine you review with.

### Create the GitHub App

1. Create a GitHub App, under your user or organisation's **Settings → Developer settings → GitHub Apps → New GitHub App**. Its name gives it a slug, its login without `[bot]`, and reviews will be posted as `<app slug>[bot]`. You can leave the webhook inactive.
2. Under **Repository permissions**, set **Contents** to **Read-only**, and **Pull requests** and **Issues** to **Read and write**. The review fetches your base branch with the app's token, which a private repository refuses without Contents, and needs the other two to post comments and reactions, and to submit and dismiss approvals.
3. Create the app, note its **Client ID**, and generate a **private key** (a `.pem` file).
4. Install the app on the repository you want reviewed.
5. Store the client ID and private key in that repository:

   ```sh
   gh variable set REVIEW_APP_CLIENT_ID --body "<the app's client ID>"
   gh secret set REVIEW_APP_PRIVATE_KEY < path/to/private-key.pem
   ```

6. Store one or both of an engine's two credentials (see [Secrets](#secrets)). With both, a `/review` line can choose which one a review runs on with `sub` or `api`. For Claude:

   ```sh
   gh secret set CLAUDE_CODE_OAUTH_TOKEN   # a token from `claude setup-token`
   # and/or
   gh secret set ANTHROPIC_API_KEY
   ```

   For Codex, set `OPENAI_API_KEY` and/or `CODEX_AUTH_JSON` instead; see [Codex credentials](#codex-credentials) for how to get and store each.

If you want a thank-you to the app to get a 🎉, pass the app's slug as `review-app-slug`. Reviews are requested with `/review` either way.

### Add the stub

Save this as `.github/workflows/code-review.yml` in your repository and replace the `checks` lines with your own:

```yaml
name: Code Review

on:
  pull_request:
    types: [opened, ready_for_review]
    branches: [main]
  # A comment line starting "/review [claude|codex] [model] [effort] [sub|api]" asks for a review
  issue_comment:
    types: [created]

jobs:
  review:
    # GitHub shows the called workflow's checks as "Code Review / Reviewer / <job>"
    name: Reviewer
    # The workflow posts through the review app's own token
    permissions:
      contents: read
    uses: it-works-on-local-dev/code-review/.github/workflows/code-review.yml@v2
    # Only the secrets the workflow declares, never secrets: inherit
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      # and/or an API key; with both, /review sub or /review api picks one:
      # ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      # For Codex reviews, OPENAI_API_KEY (api) and/or, in a private repository only, CODEX_AUTH_JSON (sub):
      # OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      REVIEW_APP_PRIVATE_KEY: ${{ secrets.REVIEW_APP_PRIVATE_KEY }}
    with:
      review-app-client-id: ${{ vars.REVIEW_APP_CLIENT_ID }}
      checks: |
        run_step setup install npm ci
        run_step check lint npm run lint
        run_step check typecheck npx tsc --noEmit
        run_step check test npm test
```

The job's name, `Reviewer`, is why GitHub lists the workflow's checks as `Code Review / Reviewer / request`, `Code Review / Reviewer / checks` and `Code Review / Reviewer / Claude` or `Code Review / Reviewer / Codex`, the last named after the engine that reviews. A name that depends on the engine can't be a required status check, and nothing should require it. Open a pull request to see the first review.

The `v2` tag exists once 2.0.0 is released, so a stub at `@v2` can't run before then. Repositories using the earlier workflow stay on `claude-code-review.yml@v1`, which keeps working (see [Versioning](#versioning)).

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
    name: Reviewer
    needs: runner
    permissions:
      contents: read
    uses: it-works-on-local-dev/code-review/.github/workflows/code-review.yml@v2
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      # OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
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

The `name`, `on` and the rest of the file are as in the hosted stub. The `runner` job's `if:` copies the workflow's own filter for the events it acts on, with the example's `my-review-app` slug in place of `review-app-slug`, so the picker doesn't start for every comment on every issue and PR. When the picker is skipped, the review job, which needs it, is skipped too, as intended. If the picker fails, the review is skipped with no reply to the `/review`. With this shape the picker's checks show as `Code Review / Runner / …` beside `Code Review / Reviewer / …`. Any job of your own that outputs a JSON label string or array works the same way, and `RUNNER_STATUS_TOKEN` belongs to the picker, not to this workflow.

### Let /iwol:setup write the stub

The `iwol` Claude Code plugin, from [it-works-on-local-dev/skills](https://github.com/it-works-on-local-dev/skills), has a setup skill, `/iwol:setup`. It asks which reviewer your repository uses and writes this stub as `code-review.yml` at `@v2` with your checks, secrets and inputs filled in. You can also write the stub by hand as above.

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
| `default-reviewer` | no | `claude` | The engine, `claude` or `codex`, used when a request names neither an engine nor a model. The default engine's credentials must be passed in your stub, or every run fails before the checks start. |
| `default-claude-model` | no | `opus` | The model a Claude review uses when its request names none: an alias or a full model ID. |
| `default-claude-effort` | no | `xhigh` | The effort a Claude review uses when its request names none: `low`, `medium`, `high`, `xhigh` or `max`. |
| `default-claude-credential` | no | `auto` | The credential a Claude review uses when its request names none: `sub` (`CLAUDE_CODE_OAUTH_TOKEN`), `api` (`ANTHROPIC_API_KEY`), or `auto`, which takes whichever of the two your stub passes, and the API key when it passes both. With `sub` or `api`, your stub must pass that secret, or every Claude review that names no credential fails. |
| `default-codex-model` | no | `gpt-6.1-sol` | The model a Codex review uses when its request names none: a `gpt-` model, bumped by hand (see [Codex models and efforts](#codex-models-and-efforts)). The value is checked on every run, even for Claude reviews, so anything that isn't a valid `gpt-` model fails every run. |
| `default-codex-effort` | no | `xhigh` | The effort a Codex review uses when its request names none: `low`, `medium`, `high`, `xhigh`, `max` or `ultra`. Checked on every run, so it must be valid even for Claude reviews. |
| `default-codex-credential` | no | `auto` | The credential a Codex review uses when its request names none: `sub` (`CODEX_AUTH_JSON`), `api` (`OPENAI_API_KEY`), or `auto`, which takes whichever of the two your stub passes, and the API key when it passes both. With `sub` or `api`, your stub must pass that secret, or every Codex review that names no credential fails. Checked on every run, so it must be valid even for Claude reviews. |
| `approve` | no | `false` | Approve the reviewed head when a review is clean, and dismiss the bot's earlier approvals when one is not. See [Approving clean reviews](#approving-clean-reviews). |

## Secrets

Pass each secret by name in the stub's `secrets:` block.

| Secret | Required | Meaning |
| --- | --- | --- |
| `CLAUDE_CODE_OAUTH_TOKEN` | one of the engine's two | A Claude Code OAuth token, for example from `claude setup-token`: the Claude engine's `sub` credential. |
| `ANTHROPIC_API_KEY` | one of the engine's two | An Anthropic API key: the Claude engine's `api` credential. |
| `OPENAI_API_KEY` | one of the engine's two | One of the Codex engine's two credentials, its `api` credential and the recommended one. |
| `CODEX_AUTH_JSON` | one of the engine's two | One of the Codex engine's two credentials, its `sub` credential: the `auth.json` of a stored ChatGPT login. For private repositories only. See [Codex credentials](#codex-credentials). |
| `REVIEW_APP_PRIVATE_KEY` | yes | The review app's private key. |

The four engine credentials are each optional to GitHub, but the `default-reviewer` engine needs at least one of its two: without one, the run fails early. Another engine needs one only when a request names it, and a request naming an engine with no credential passed gets a reply naming that engine's two secrets. Pass both of an engine's secrets to let each review request choose between them with `sub` or `api`, as described under [What the workflow does](#what-the-workflow-does).

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

- **`@v1` is frozen at 1.2.1, the last 1.x release.** The workflow was `claude-code-review.yml` there, with Claude as its only engine. Public `@v1` stubs keep working, without Codex.
- **`@v2` moves.** From 2.0.0, the `v2` tag moves to each compatible release, so a stub at `@v2` receives fixes and features without changing.
- **Breaking changes get a new major tag.** Moving from `v1` to `v2` is a change to your stub, as is any later major. Migrating from `@v1` to `@v2` means:
  - the new path, `.github/workflows/code-review.yml@v2`, in place of `claude-code-review.yml@v1`;
  - the renamed inputs: `default-model`, `default-effort` and `default-credential` become `default-claude-model`, `default-claude-effort` and `default-claude-credential`;
  - the new words: `sub` and `api` in place of `oauth` and `api-key`, in the inputs and in `/review` lines, plus the optional engine word;
  - the job name `Reviewer`, so the checks read `Code Review / Reviewer / <job>`.
- **You can pin.** A stub may call a full tag such as `@v1.2.3` instead, and move it when you choose.

This organisation's own repositories call the workflow at `@main`, so they see a merged change before it is released.

## Adding and removing a secret

This section is for changing the workflow itself. Stubs pass secrets by name, which has two consequences: a secret added to the workflow reaches no caller until that caller's stub passes it, and GitHub will not start the workflow for a stub that passes a secret the workflow does not declare.

**Adding a secret.** Declare it as optional (`required: false`), so stubs that do not pass it yet keep working. Then add it to the stub in every place one is written down, in step with the workflow:

- the stub in the workflow's header comment,
- this repository's [`.github/workflows/review.yml`](.github/workflows/review.yml),
- the stub file the setup skill writes as `code-review.yml`,
- the tables and stubs in this README, and
- the stub template in the setup skill of [it-works-on-local-dev/skills](https://github.com/it-works-on-local-dev/skills), which must change in step with this repository.

**Removing or renaming a secret** is the reverse, and riskier. A stub that still passes the old name stops working the moment `v2` moves to the release that dropped it. Either keep the old name declared as optional until every stub has stopped passing it, or make the removal a major release. A rename is an addition followed, later, by a removal.
