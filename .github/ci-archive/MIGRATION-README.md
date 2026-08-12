# Jenkins to GitHub Actions Migration Report

This document summarizes the migration of this repository's Jenkins pipelines to GitHub Actions workflows.

## Summary

| Original Jenkinsfile | New GitHub Actions Workflow | Pipeline Type |
| --- | --- | --- |
| `Jenkinsfile` | `.github/workflows/process.yml` | Declarative |
| `androidbuild/Jenkinsfile` | `.github/workflows/android-build.yml` | Declarative |
| `matrixbuilds/Jenkinsfile` | `.github/workflows/matrix-build.yml` | Declarative (matrix) |
| `laravelgithubapi/Jenkinsfile` | `.github/workflows/laravel-github-api.yml` | Declarative |
| `netlifydeployment/Jenkinsfile` | `.github/workflows/netlify-deployment.yml` | Declarative |

All original Jenkinsfiles have been archived to `.github/ci-archive/` for reference and removed from their original locations.

All new workflows were validated with [`actionlint`](https://github.com/rhysd/actionlint) (v1.7.12) and pass with no errors.

---

## 1. `Jenkinsfile` → `.github/workflows/process.yml`

**Trigger:** `workflow_dispatch` and `push` to `main` (Jenkins job had no explicit trigger; assumed manual/CI trigger).

**Conversion notes:**
- `agent { label 'master' }` → `runs-on: ubuntu-latest`.
- The `sh` step decoding a Base64 file was preserved as a `run` step. Because the original script referenced the literal strings `INPUT_FILE_PATH` and `INPUT_FILE` (not their Groovy variables, due to how the triple-quoted `sh` block was written), the migrated workflow makes this explicit by using a `INPUT_FILE_PATH_BASE64` secret as the Base64 source and the `INPUT_FILE` environment variable for the destination path.
- `archiveArtifacts artifacts: 'input.csv', allowEmptyArchive: true` → `actions/upload-artifact` with `if-no-files-found: ignore`.

**Required secrets:**
- `INPUT_FILE_PATH_BASE64` — Base64-encoded content that should be decoded into `input.csv`.

---

## 2. `androidbuild/Jenkinsfile` → `.github/workflows/android-build.yml`

**Trigger:** `workflow_dispatch` and `push` to any branch (mirrors branch-based flavor logic in the original pipeline).

**Conversion notes:**
- `agent { label 'android' }` → `runs-on: ubuntu-latest`.
- `environment { ANDROID_HOME, GRADLE_OPTS }` → workflow-level `env`.
- `checkout scm` → `actions/checkout`.
- The branch-name flavor extraction logic (`readProperties`) was inlined into a single `Setup` step using Bash, exposing `build_flavor` and `build_flavor_cap` as step outputs consumed by later steps (replacing Groovy `env.BUILD_FLAVOR`).
- `publishTestResults` → `dorny/test-reporter` (JUnit XML reporting).
- `publishHTML` (lint report) → `actions/upload-artifact` (GitHub Actions has no first-class HTML publishing action equivalent to the Jenkins HTML Publisher plugin; the HTML report is uploaded as a build artifact instead).
- `archiveArtifacts` for APKs → `actions/upload-artifact`.
- `when { expression { env.BUILD_FLAVOR != 'debug' } }` → `if: steps.setup.outputs.build_flavor != 'debug'`.
- `post { always { cleanWs() } }` → not required; GitHub Actions runners are ephemeral, so no explicit workspace cleanup step is needed.

**Required secrets/variables:** None (no credentials used in the original pipeline beyond environment configuration).

---

## 3. `matrixbuilds/Jenkinsfile` → `.github/workflows/matrix-build.yml`

**Trigger:** `workflow_dispatch` and `push` to `master`.

**Conversion notes:**
- The `matrix { axes { ... } excludes { ... } }` block → a native GitHub Actions `strategy.matrix` with `platform` and `java_version`, plus a matching `exclude` entry for `macos`/`jdk-8`.
- Platform-specific `agent { label "${PLATFORM}" }` → an `include` mapping each `platform` to the appropriate GitHub-hosted `os` runner (`ubuntu-latest`, `windows-latest`, `macos-latest`).
- `tools { jdk ..., maven ... }` → `actions/setup-java` and `stCarolas/setup-maven` (maven 3.8.1 pinned).
- Windows-conditional `bat`/`sh` steps → separate steps gated by `if: matrix.platform == 'windows'` (using `shell: cmd`) vs `if: matrix.platform != 'windows'`.
- `publishTestResults` → `dorny/test-reporter`.
- `archiveArtifacts ... classifier: "${PLATFORM}-${JAVA_VERSION}"` → `actions/upload-artifact` with a matrix-scoped artifact name.
- The `Integration` stage (single agent, depends on all matrix builds) → a separate `integration` job with `needs: build-test-package`, downloading all artifacts via `actions/download-artifact`.
- The `Docker Build` stage (`when { branch 'master' }`, nested loops over platforms/java versions) → a separate `docker-build` job with `if: github.ref == 'refs/heads/master'` and its own `strategy.matrix` (linux only, jdk-11/17), replacing the Groovy `for` loops.
- `post { always { echo ... } }` → omitted; GitHub Actions surfaces job status directly in the UI.

**Required secrets/variables:** None currently required; if the `docker-build` job needs to push images, add registry credentials as secrets (e.g., `REGISTRY_USERNAME`/`REGISTRY_PASSWORD`) and a `docker/login-action` step.

---

## 4. `laravelgithubapi/Jenkinsfile` → `.github/workflows/laravel-github-api.yml`

**Trigger:** `pull_request` (matches the original pipeline's PR-driven build/status/close behavior).

**Conversion notes:**
- `agent any` → `runs-on: ubuntu-latest`.
- A MySQL `services` container replaces the Jenkins host's local MySQL install used via `mysql -u ... -p...`.
- `withCredentials([usernamePassword(credentialsId: MYSQL_CREDENTIALS_ID, ...)])` → `secrets.MYSQL_USERNAME` / `secrets.MYSQL_PASSWORD` environment variables.
- `withCredentials([usernamePassword(credentialsId: GITHUB_CREDENTIALS_ID, ...)])` → `secrets.GH_STATUS_TOKEN`, used as a bearer/basic auth token for the GitHub Status and Pulls APIs.
- The `post { always { ... } }` block (commit status update + PR auto-close on failure + DB cleanup) → three separate steps using `if: always()` / `if: failure()`, using `${{ github.repository }}`, `${{ github.event.pull_request.head.sha }}`, and `${{ github.event.pull_request.number }}` in place of Groovy `scm`/`env.CHANGE_ID` lookups (removing the need for the `CHANGE_ID` curl/jq fallback, since `github.event.pull_request.number` is always available on `pull_request` events).
- SQL scratch files write to `/tmp` instead of the Jenkins-specific `/var/lib/jenkins/automation` path.

**Required secrets:**
- `MYSQL_USERNAME` / `MYSQL_PASSWORD` — MySQL service credentials (also used as the MySQL root password for the service container).
- `GH_STATUS_TOKEN` — GitHub token with permission to set commit statuses and update pull requests (a fine-grained PAT or `GITHUB_TOKEN` with appropriate permissions).

---

## 5. `netlifydeployment/Jenkinsfile` → `.github/workflows/netlify-deployment.yml`

**Trigger:** `workflow_dispatch` and `push` to `main`.

**Conversion notes:**
- `agent any` → `runs-on: ubuntu-latest`, split into a `build` job and a `deploy` job (`needs: build`) so the build artifact can be shared without re-running `npm install`/`npm run build`.
- `sh 'npm install'` / `sh 'npm run build'` → equivalent `run` steps, after `actions/setup-node` (Node 20).
- `environment { NETLIFY_SITE_ID, NETLIFY_AUTH_TOKEN = credentials('netlify-token') }` → job-level `env` with `NETLIFY_AUTH_TOKEN` sourced from `secrets.NETLIFY_AUTH_TOKEN`.
- `sh 'npm install -g netlify-cli'` / `sh 'netlify deploy ...'` → equivalent `run` steps in the `deploy` job.

**Required secrets:**
- `NETLIFY_AUTH_TOKEN` — Netlify personal access token (previously bound via the Jenkins `netlify-token` credential).

---

## Actions Used (pinned to commit SHA)

| Action | Version | SHA |
| --- | --- | --- |
| `actions/checkout` | v5.0.0 | `08c6903cd8c0fde910a37f88322edcfb5dd907a8` |
| `actions/upload-artifact` | v4.6.2 | `ea165f8d65b6e75b540449e92b4886f43607fa02` |
| `actions/download-artifact` | v5.0.0 | `634f93cb2916e3fdff6788551b99b062d0335ce0` |
| `actions/setup-java` | v4.7.1 | `c5195efecf7bdfc987ee8bae7a71cb8b11521c00` |
| `actions/setup-node` | v6.0.0 | `2028fbc5c25fe9cf00d9f06a71cc4710d4507903` |
| `dorny/test-reporter` | v3.0.0 | `a43b3a5f7366b97d083190328d2c652e1a8b6aa2` |
| `stCarolas/setup-maven` | v5 | `d6af6abeda15e98926a57b5aa970a96bb37f97d1` |

## Validation

- All workflows were checked with `actionlint` (v1.7.12) and reported no errors.

## Follow-up / Manual Steps Required

1. Add the required repository secrets listed above (`INPUT_FILE_PATH_BASE64`, `MYSQL_USERNAME`, `MYSQL_PASSWORD`, `GH_STATUS_TOKEN`, `NETLIFY_AUTH_TOKEN`) under **Settings → Secrets and variables → Actions**.
2. Review branch/trigger assumptions above (original Jenkinsfiles had no `triggers {}` block in most cases) and adjust `on:` sections to match your desired CI triggers.
3. Confirm Android SDK/Gradle and platform-specific tooling versions match your project's actual requirements (the GitHub-hosted runners include common SDKs, but custom versions may need explicit setup actions).
