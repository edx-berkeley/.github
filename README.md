# edx-berkeley

This org hosts the infrastructure, tooling, and course content for the UCB Data Science edX program. The courses are self-paced Jupyter-based courses graded via an otter-grader service running on GKE.

---

## Repositories

### Infrastructure

| Repository | Visibility | Purpose |
|---|---|---|
| [edx-hub](https://github.com/edx-berkeley/edx-hub) | Private | Deployment repo for the `edx` GKE cluster. Contains Helm values, hubploy config, SOPS-encrypted secrets (deploy-edx only), and GitHub Actions workflows that deploy JupyterHub and otter-service to staging/prod. Default branch: `prod`. |
| [edx-user-image](https://github.com/edx-berkeley/edx-user-image) | Public | Docker image for JupyterHub single-user servers. CI builds and tests on PRs; pushing a `X.Y.Z` tag releases to Google Artifact Registry and opens an auto-PR in edx-hub. |
| [otter-service](https://github.com/edx-berkeley/otter-service) | Public | Tornado-based grading service. Receives submissions from edX/LTI, runs otter-grader in Docker, and posts grades back. Deployed to `otter-prod` / `otter-staging` namespaces on the `edx` cluster. |
| [edx-berkeley.github.io](https://github.com/edx-berkeley/edx-berkeley.github.io) | Private | Next.js site served via GitHub Pages as the org's public web presence. |

### Course content

| Repository | Visibility | Purpose |
|---|---|---|
| [xDevs](https://github.com/edx-berkeley/xDevs) | Private | Shared instructor dev repo for 88B, 88C, 88E, 8X. Raw source notebooks are processed with `otter-assign` to produce student releases, solutions, and autograder zips. |
| [88B-student](https://github.com/edx-berkeley/88B-student) | Public | Student-facing lab notebooks for Data 88B. |
| [88C-student](https://github.com/edx-berkeley/88C-student) | Public | Student-facing lab notebooks for Data 88C. |
| [88E-student](https://github.com/edx-berkeley/88E-student) | Public | Student-facing lab notebooks for Data 88E. |
| [8X-student](https://github.com/edx-berkeley/8X-student) | Public | Student-facing lab notebooks for Data 8X. |

### Autograders

| Repository | Visibility | Purpose |
|---|---|---|
| [88B-autograders](https://github.com/edx-berkeley/88B-autograders) | Private | Otter autograder zips for Data 88B labs. Read at runtime by otter-service via the `course-content-reader` GitHub App. |
| [88C-autograders](https://github.com/edx-berkeley/88C-autograders) | Private | Otter autograder zips for Data 88C labs. |
| [88E-autograders](https://github.com/edx-berkeley/88E-autograders) | Private | Otter autograder zips for Data 88E labs. |
| [8X-autograders](https://github.com/edx-berkeley/8X-autograders) | Internal | Otter autograder zips for Data 8X labs. |

### Utilities

| Repository | Visibility | Purpose |
|---|---|---|
| [otter-submit](https://github.com/edx-berkeley/otter-submit) | Public | Student-side submission helper that packages a notebook and POSTs it to otter-service. |
| [edx-support](https://github.com/edx-berkeley/edx-support) | Internal | Support tooling (course roster management, LTI utilities). |
| [edx-usage](https://github.com/edx-berkeley/edx-usage) | Public | Scripts for pulling and reporting edX course usage data. |

---

## CI process

End-to-end flow from a notebook edit through to a JupyterHub deploy. Each arrow is automated unless marked **(manual)**. Every step posts to `#edx-hub-ci` Slack on success and failure.

### 1. Notebook content (xDevs → student/autograder repos)

```
   Open PR with raw notebook edits in xDevs/{88b,88c,88e,8x}/raw_notebooks/**
        │
        ▼  notebook-pipeline.yml (PR target)
   Runs otter-assign inside the deployed edx-user-image, then runs
   grader.check + otter grade against the regenerated student/solution.
   Commits the regenerated artifacts back to the PR head branch.
        │
        ▼  (manual)  Merge PR to xDevs:main
        │
        ▼  (manual)  Dispatch deploy-notebooks workflow with apply_changes=true
   Opens PRs in 88B/88C/88E-{student,autograders} with the new artifacts.
        │
        ▼  (manual)  Merge each downstream PR
   Students see the updated notebooks; otter-service fetches the new
   autograder zip on next submission.
```

### 2. JupyterHub image (edx-user-image → edx-hub → cluster)

```
   Open PR with Dockerfile / environment changes
        │
        ▼  grader-check.yml (PR target)
   Builds the PR image, fetches all course notebooks via the
   course-content-reader App, runs grader.check across student
   (must fail) and solution (must pass) notebooks.
        │
        ▼  (manual)  Merge PR to edx-user-image:main
        │
        ▼  (manual)  Push a version tag (X.Y.Z) to upstream
        │
        ▼  build-push-create-pr.yaml (tag push)
   Builds and pushes the image to Google Artifact Registry.
   Opens an auto-PR in edx-hub:staging bumping `deployments/edx/config/common.yaml`.
        │
        ▼  (manual)  Merge auto-PR to edx-hub:staging
        │
        ▼  deploy-edx.yaml (push to staging, paths-filtered)
   Runs `hubploy deploy` against the staging cluster.
        │
        ▼  (manual)  Open + merge PR from edx-hub:staging → edx-hub:prod
        │
        ▼  deploy-edx.yaml (push to prod, paths-filtered)
   Same deploy against the prod cluster.
```

### 3. otter-service (otter-service → edx-hub → cluster)

```
   Open PR with otter-service code changes
        │
        ▼  python-app.yml + docker-grade-check.yml (PR target)
   Lints, runs pytest with WIF-authed GCP, builds the Docker image,
   and runs the docker grade harness end-to-end.
        │
        ▼  (manual)  Merge PR to otter-service:main
        │
        ▼  (manual)  Bump __version__ + CHANGELOG, push X.Y.Z tag
        │
        ▼  release.yml (tag push)
   Builds the Docker image, runs the FULL grading test against all
   courses (gates publish). On success: creates GitHub Release, publishes
   to PyPI, builds + pushes image via Cloud Build, opens auto-PR in
   edx-hub:staging bumping `otter-service/values.yaml`.
        │
        ▼  (manual)  Merge auto-PR to edx-hub:staging
        │
        ▼  deploy-otter.yaml (push to staging, paths-filtered)
   Runs `helm upgrade --install` against the staging cluster, then a
   smoke-test job exercising the full KEDA-routed grading pipeline.
        │
        ▼  (manual)  Open + merge PR from edx-hub:staging → edx-hub:prod
        │
        ▼  deploy-otter.yaml (push to prod, paths-filtered)
   Same deploy + smoke test against prod.
```

### What runs on doc-only PRs

All four repos' CI workflows ignore PRs that only modify `README.md`, `CHANGELOG.md`, `LICENSE`, `docs/**`, or `.github/**`. Doc-only PRs trigger zero workflow runs.

---

## GitHub Apps

Four GitHub Apps are installed on this org. All are used exclusively by GitHub Actions workflows — no personal access tokens.

| App slug | App ID | Purpose | Used by |
|---|---|---|---|
| `edx-image-builder` | 3380935 | Writes to edx-hub: opens and updates the auto-PR that bumps the `edx-user-image` tag in the deployment config. Also used by otter-service to open its release auto-PR. | edx-user-image (`build-push-create-pr.yaml`), otter-service (`release.yml` update-edx-hub job) |
| `course-content-reader` | 3484933 | Reads autograder and student/solution repos at runtime to fetch the correct autograder zip for a given submission. | otter-service (runtime + CI), edx-user-image (`grader-check.yml`), xDevs (`notebook-pipeline.yml`) |
| `edx-notebook-distributor` | 3537861 | Reads PR-head repos (private forks of xDevs) and writes regenerated artifacts; also opens PRs to the `{88B,88C,88E}-{student,autograders}` repos for downstream propagation. | xDevs (`notebook-pipeline.yml`, `deploy-notebooks.yml`) |
| `edx-hub-read-ci` | 3570619 | Reads edx-hub (private) to resolve the currently deployed `edx-user-image` tag during notebook CI. | xDevs (`notebook-pipeline.yml`, referenced as `EDX_HUB_READ_CI_APP_ID`) |

---

## Repository Variables and Secrets

Credentials shared across multiple repos live at the **org level** and are scoped to the repos that need them via the `SELECTED` visibility. Repo-level variables/secrets are reserved for genuinely repo-specific things (image paths, bot metadata, repo-level GAR creds).

### Org-level (visible to selected repos)

**Variables**

| Name | Value/Purpose |
|---|---|
| `GCP_PROJECT` | `data8x-scratch` |
| `EDX_IMAGE_BUILDER_APP_ID` | App ID for the `edx-image-builder` GitHub App |
| `COURSE_CONTENT_READER_APP_ID` | App ID for `course-content-reader` |
| `COURSE_CONTENT_READER_INSTALLATION_ID` | Installation ID for `course-content-reader` |
| `EDX_NOTEBOOK_DISTRIBUTOR_APP_ID` | App ID for `edx-notebook-distributor` |
| `EDX_NOTEBOOK_DISTRIBUTOR_INSTALLATION_ID` | Installation ID for `edx-notebook-distributor` |
| `EDX_HUB_READ_CI_APP_ID` | App ID for `edx-hub-read-ci` |
| `EDX_HUB_READ_CI_INSTALLATION_ID` | Installation ID for `edx-hub-read-ci` |
| `DOCKERHUB_USERNAME` | Docker Hub username `edxcdss` for the Berkeley SPA `edx-ds@berkeley.edu`. Auth for image pulls in CI (currently used by `edx-user-image/.github/workflows/grader-check.yml` to dodge Docker Hub's 100-pulls/6h anonymous limit). SPA password SOPS-encrypted at [`edx-hub/deployments/edx/secrets/service-accounts.yaml`](https://github.com/edx-berkeley/edx-hub/blob/staging/deployments/edx/secrets/service-accounts.yaml). |
| `EDX_SERVICE_ACCOUNT_EMAIL` | edX.org login email (`edx-ds@berkeley.edu` — same Berkeley SPA as `DOCKERHUB_USERNAME` above). Used by `edx-support` and `edx-usage` nightly CSV-fetch workflows. Password SOPS-encrypted at the same `service-accounts.yaml` file. |

**Secrets**

| Name | Purpose |
|---|---|
| `GCP_SA_KEY` | GCP service account JSON key for `edx-hub-github-actions@data8x-scratch.iam.gserviceaccount.com` — used by edx-hub deploy workflows (GKE auth, SOPS/KMS decryption in deploy-edx only, Artifact Registry pull) and xDevs notebook-pipeline (pull deployed image). |
| `EDX_IMAGE_BUILDER_PRIVATE_KEY` | Private key for `edx-image-builder` |
| `COURSE_CONTENT_READER_PRIVATE_KEY` | Private key for `course-content-reader` |
| `EDX_NOTEBOOK_DISTRIBUTOR_PRIVATE_KEY` | Private key for `edx-notebook-distributor` |
| `EDX_HUB_READ_CI_PRIVATE_KEY` | Private key for `edx-hub-read-ci` |
| `OTTER_LTI_CONSUMER_KEY` | LTI consumer key — passed by edx-hub `deploy-otter.yaml` into the otter-srv pod for posting grades back to edX |
| `OTTER_LTI_CONSUMER_SECRET` | LTI consumer secret — same path as above |
| `JH_API_TOKEN_PROD` | JupyterHub API token (prod) — used by otter-srv for user lookups |
| `JH_API_TOKEN_STAGING` | JupyterHub API token (staging) |
| `SLACK_WEBHOOK_URL` | Incoming webhook URL for the `#edx-hub-ci` Slack channel |
| `DOCKERHUB_TOKEN` | Docker Hub Personal Access Token for the SPA referenced by `DOCKERHUB_USERNAME` above. Rotated 2026-06-09 alongside the SPA cutover. |
| `EDX_SERVICE_ACCOUNT_PASSWORD` | edX.org login password for the same Berkeley SPA referenced by `EDX_SERVICE_ACCOUNT_EMAIL` above. SOPS-decryptable copy at `edx-hub/deployments/edx/secrets/service-accounts.yaml` (single source of truth for the SPA password). |

### Repo-level overrides

Only kept where the value is genuinely repo-specific:

| Repo | Variables | Secrets |
|---|---|---|
| **edx-hub** | `STAGING_ENABLED` (`true`/`false` gate for staging deploys) | — |
| **edx-user-image** | `IMAGE` (GAR image path), `HUB` (deployment name), `IMAGE_BUILDER_BOT_EMAIL`, `IMAGE_BUILDER_BOT_NAME` | `GAR_SECRET_KEY_EDX` (GAR push key, separate from the org-level pull key) |
| **otter-service** | — | — |
| **xDevs** | — | — |

---

## Infrastructure Overview

All services run on the `edx` GKE cluster in GCP project `data8x-scratch`, zone `us-central1-b`.

| Namespace | Service | Node pool |
|---|---|---|
| `edx-staging` | JupyterHub (staging) | `edx-pool` (e2-standard-4) |
| `edx-prod` | JupyterHub (prod) | `edx-pool` (e2-standard-4) |
| `otter-staging` | otter-service (staging) | `otter-pool` (n1-highmem-4) |
| `otter-prod` | otter-service (prod) | `otter-pool` (n1-highmem-4) |
| `keda` | KEDA autoscaler | — |

Container images are stored in Google Artifact Registry (`us-central1-docker.pkg.dev/data8x-scratch/user-images/`). SOPS with Cloud KMS is used to encrypt secrets in `deployments/edx/secrets/` (deploy-edx only); otter-service runtime credentials moved to org-level GitHub secrets (no SOPS).
