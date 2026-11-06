# Changelog

## [2.0.0](https://github.com/it-works-on-local-dev/code-review/compare/v1.2.1...v2.0.0) (2026-11-06)


### ⚠ BREAKING CHANGES

* add engine selection, per-engine inputs and engine-neutral names to the request contract ([#23](https://github.com/it-works-on-local-dev/code-review/issues/23))
* rename the reusable workflow to code-review.yml ([#21](https://github.com/it-works-on-local-dev/code-review/issues/21))

### Features

* add engine selection, per-engine inputs and engine-neutral names to the request contract ([#23](https://github.com/it-works-on-local-dev/code-review/issues/23)) ([6ac1267](https://github.com/it-works-on-local-dev/code-review/commit/6ac12671823a73acb0242b2915922904b5298b26))
* add the Codex engine step with its two credential routes ([#29](https://github.com/it-works-on-local-dev/code-review/issues/29)) ([6f3c2ac](https://github.com/it-works-on-local-dev/code-review/commit/6f3c2acc218bef13455487ebf1d9203377c79282))
* post Codex findings as inline comments and scrub their bodies ([#31](https://github.com/it-works-on-local-dev/code-review/issues/31)) ([269070b](https://github.com/it-works-on-local-dev/code-review/commit/269070b1301fd19257d5e7609783dda1e8e4d8b1))
* rename the reusable workflow to code-review.yml ([#21](https://github.com/it-works-on-local-dev/code-review/issues/21)) ([fd800aa](https://github.com/it-works-on-local-dev/code-review/commit/fd800aacb6ac2984fef892f5781a4a0a2f9d57b9))
* scrub review output for credential-shaped strings before anything is posted ([#27](https://github.com/it-works-on-local-dev/code-review/issues/27)) ([8c522f3](https://github.com/it-works-on-local-dev/code-review/commit/8c522f35d18e410d371c6c29cddd8ec31093b718))

## [1.2.1](https://github.com/it-works-on-local-dev/code-review/compare/v1.2.0...v1.2.1) (2026-11-04)


### Bug Fixes

* **deps:** bump anthropics/claude-code-action from 1.0.262 to 1.0.266 ([#20](https://github.com/it-works-on-local-dev/code-review/issues/20)) ([3e77474](https://github.com/it-works-on-local-dev/code-review/commit/3e774743c69583cc02a992e12990f26be1d674f5))
* **deps:** ship claude-code-action 1.0.262 and release each Dependabot bump ([#18](https://github.com/it-works-on-local-dev/code-review/issues/18)) ([4573ba5](https://github.com/it-works-on-local-dev/code-review/commit/4573ba548aa1c4735834af693ca0fbefbba41b13))

## [1.2.0](https://github.com/it-works-on-local-dev/code-review/compare/v1.1.0...v1.2.0) (2026-10-24)


### Features

* say why a Claude review failed ([#15](https://github.com/it-works-on-local-dev/code-review/issues/15)) ([041ecb1](https://github.com/it-works-on-local-dev/code-review/commit/041ecb124d4d4736b39db14699210a88cc638373))


### Bug Fixes

* ask for the Contents permission when creating the review app ([#13](https://github.com/it-works-on-local-dev/code-review/issues/13)) ([06e1f7d](https://github.com/it-works-on-local-dev/code-review/commit/06e1f7df7d4ddfd11d436f921837b1cd97140532))

## [1.1.0](https://github.com/it-works-on-local-dev/code-review/compare/v1.0.0...v1.1.0) (2026-10-22)


### Features

* let a review request pick the Claude credential ([#9](https://github.com/it-works-on-local-dev/code-review/issues/9)) ([d34400b](https://github.com/it-works-on-local-dev/code-review/commit/d34400bece9794cb904fb58296ce680ae55aa91e))

## 1.0.0 (2026-10-22)


### Features

* add release-please automation that tags releases and moves the v1 tag ([#5](https://github.com/it-works-on-local-dev/code-review/issues/5)) ([334db9c](https://github.com/it-works-on-local-dev/code-review/commit/334db9cf28a55bcc1fda1d9ca6a06851021211e3))
* parametrise the review workflow with runs-on, review-app-slug and either Claude credential ([#3](https://github.com/it-works-on-local-dev/code-review/issues/3)) ([7a4438c](https://github.com/it-works-on-local-dev/code-review/commit/7a4438c7708ade04b3c7311eace8e765a844f53e))
