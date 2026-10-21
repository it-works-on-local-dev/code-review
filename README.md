# code-review

A reusable GitHub Actions workflow, `.github/workflows/claude-code-review.yml`, that has Claude review a repository's pull requests: it runs the repository's own checks, reads the diff, posts findings as inline comments and a summary as the repository's review GitHub App, and reviews again on a `/review` comment. Other repositories call it from a short stub. Imported from the private `it-works-on-local-dev/tooling` repository; this README is a placeholder until the public documentation lands.

Throwaway change to confirm the review workflow and CI run on a pull request here.
