# edx-berkeley

This org hosts the infrastructure, tooling, and course content for the Data 8x edX program at UC Berkeley. The courses are self-paced Jupyter-based courses graded via an otter-grader service running on GKE.

---

## Repositories

### Infrastructure

| Repository | Visibility | Purpose |
|---|---|---|
| [edx-hub](https://github.com/edx-berkeley/edx-hub) | Private | Deployment repo for the `edx` GKE cluster. Contains Helm values, hubploy config, SOPS-encrypted secrets, and GitHub Actions workflows that deploy JupyterHub and otter-service to staging/prod. Default branch: `prod`. |
| [edx-user-image](https://github.com/edx-berkeley/edx-user-image) | Public | Docker image for JupyterHub single-user servers. CI builds and tests on PRs; pushing a `X.Y.Z` tag releases to Google Artifact Registry and opens an auto-PR in edx-hub. |
| [otter-service](https://github.com/edx-berkeley/otter-service) | Public | Tornado-based grading service. Receives submissions from edX/LTI, runs otter-grader in Docker, and posts grades back. Deployed to `otter-prod` / `otter-staging` namespaces on the `edx` cluster. |
| [edx-berkeley.github.io](https://github.com/edx-berkeley/edx-berkeley.github.io) | Private | Next.js site served via GitHub Pages as the org's public web presence. |

### Course content

| Repository | Visibility | Purpose |
|---|---|---|
| [xDevs](https://github.com/edx-berkeley/xDevs) | Private | Shared instructor dev repo for 88B, 88C, 88E, 8X. Raw source notebooks are processed with `otter-assign` to produce student releases, solutions, and autograder zips. |
| [88B-dev](https://github.com/edx-berkeley/88B-dev) | Private | Instructor notebooks and configs for Data 88B. |
| [88C-dev](https://github.com/edx-berkeley/88C-dev) | Private | Instructor notebooks and configs for Data 88C. |
| [88E-online_dev](https://github.com/edx-berkeley/88E-online_dev) | Private | Instructor notebooks and configs for Data 88E (online). |
| [88B-student](https://github.com/edx-berkeley/88B-student) | Public | Student-facing lab notebooks for Data 88B. |
| [88C-student](https://github.com/edx-berkeley/88C-student) | Public | Student-facing lab notebooks for Data 88C. |
| [88E-student](https://github.com/edx-berkeley/88E-student) | Public | Student-facing lab notebooks for Data 88E. |
| [8X-student](https://github.com/edx-berkeley/8X-student) | Public | Student-facing lab notebooks for Data 8X. |
| [88E-online](https://github.com/edx-berkeley/88E-online) | Private | Published online course materials for Data 88E. |

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
| [data8-materials](https://github.com/edx-berkeley/data8-materials) | Internal | Archived Data 8 course materials. |
| [data8x](https://github.com/edx-berkeley/data8x) | Private | Archived Data 8x course website (Jekyll). |
| [data8x-materials18](https://github.com/edx-berkeley/data8x-materials18) | Private | Archived 2018 course materials. |
| [data8x-materials19](https://github.com/edx-berkeley/data8x-materials19) | Private | Archived 2019 course materials. |

---

## GitHub Apps

Four GitHub Apps are installed on this org. All are used exclusively by GitHub Actions workflows — no personal access tokens.

| App slug | App ID | Purpose | Used by |
|---|---|---|---|
| `edx-image-builder` | 3380935 | Writes to edx-hub: opens and updates the auto-PR that bumps the `edx-user-image` tag in the deployment config | [edx-user-image](https://github.com/edx-berkeley/edx-user-image) (`build-push-create-pr.yaml`) |
| `course-content-reader` | 3484933 | Reads autograder repos at runtime to fetch the correct autograder zip for a given submission | [otter-service](https://github.com/edx-berkeley/otter-service) (runtime, via `OTTER_AUTOGRADERS_*` env vars) |
| `edx-notebook-distributor` | 3537861 | Writes student-facing and solutions notebooks to the `-student` and `-dev` repos after `otter-assign` processing | [xDevs](https://github.com/edx-berkeley/xDevs) (notebook distribution workflow) |
| `edx-hub-read-ci` | 3570619 | Reads edx-hub (private) to resolve the currently deployed `edx-user-image` tag during notebook CI | [xDevs](https://github.com/edx-berkeley/xDevs) (`notebook-ci.yml`, referenced as `NOTEBOOK_CI_APP_ID`) |

---

## Repository Variables and Secrets

There are no org-level Actions variables or secrets. All variables and secrets are scoped to individual repos.

### edx-hub

**Variables**

| Name | Description |
|---|---|
| `STAGING_ENABLED` | Set to `true` to allow staging deploys; `false` gates staging off while prod deploys continue |

**Secrets**

| Name | Description |
|---|---|
| `GCP_SA_KEY` | GCP service account JSON key for `edx-hub-github-actions@data8x-scratch.iam.gserviceaccount.com` — used by both deploy workflows for GKE auth, SOPS/KMS decryption, and Artifact Registry access |

---

### edx-user-image

**Variables**

| Name | Description |
|---|---|
| `IMAGE` | Full image path in Google Artifact Registry (e.g., `data8x-scratch/user-images/edx-user-image`) |
| `HUB` | Deployment name used to locate and update the image tag in edx-hub |
| `EDX_IMAGE_BUILDER_APP_ID` | App ID for the `edx-image-builder` GitHub App (opens PRs in edx-hub) |
| `IMAGE_BUILDER_BOT_EMAIL` | Git author email for automated commits to edx-hub |
| `IMAGE_BUILDER_BOT_NAME` | Git author name for automated commits to edx-hub |
| `OTTER_AUTOGRADERS_APP_ID` | App ID for the `course-content-reader` GitHub App (reads autograder repos) |
| `OTTER_AUTOGRADERS_INSTALLATION_ID` | Installation ID for the `course-content-reader` GitHub App |

**Secrets**

| Name | Description |
|---|---|
| `GAR_SECRET_KEY_EDX` | GCP service account JSON key with push access to Google Artifact Registry |
| `PRIVATE_KEY_SECRET` | Private key for the `edx-image-builder` GitHub App |
| `OTTER_GH_APP_PRIVATE_KEY` | Private key for the `course-content-reader` GitHub App |
| `SLACK_WEBHOOK_URL` | Incoming webhook URL for the `#edx-hub-ci` Slack channel |

---

### otter-service

**Variables**

| Name | Description |
|---|---|
| `OTTER_AUTOGRADERS_APP_ID` | App ID for the `course-content-reader` GitHub App |
| `OTTER_AUTOGRADERS_INSTALLATION_ID` | Installation ID for the `course-content-reader` GitHub App |

**Secrets**

| Name | Description |
|---|---|
| `OTTER_AUTOGRADERS_PRIVATE_KEY` | Private key for the `course-content-reader` GitHub App |
| `SLACK_WEBHOOK_URL` | Incoming webhook URL for the `#edx-hub-ci` Slack channel |

---

### xDevs

**Variables**

| Name | Description |
|---|---|
| `NOTEBOOK_CI_APP_ID` | App ID for the `edx-hub-read-ci` GitHub App (reads edx-hub to resolve the deployed image tag) |

**Secrets**

| Name | Description |
|---|---|
| `NOTEBOOK_CI_PRIVATE_KEY` | Private key for the `edx-hub-read-ci` GitHub App |
| `GCP_SA_KEY` | GCP service account JSON key with pull access to Google Artifact Registry |
| `GCP_PROJECT` | GCP project ID (`data8x-scratch`) |
| `SLACK_WEBHOOK_URL` | Incoming webhook URL for the `#edx-hub-ci` Slack channel |

---

## Infrastructure Overview

All services run on the `edx` GKE cluster in GCP project `data8x-scratch`, region `us-central1`.

| Namespace | Service | Node pool |
|---|---|---|
| `edx-staging` | JupyterHub (staging) | `edx-pool` (e2-standard-4) |
| `edx-prod` | JupyterHub (prod) | `edx-pool` (e2-standard-4) |
| `otter-staging` | otter-service (staging) | `otter-pool` (n1-highmem-4) |
| `otter-prod` | otter-service (prod) | `otter-pool` (n1-highmem-4) |
| `keda` | KEDA autoscaler | — |

Container images are stored in Google Artifact Registry (`us-central1-docker.pkg.dev/data8x-scratch/user-images/`). SOPS with Cloud KMS is used to encrypt secrets committed to edx-hub.
