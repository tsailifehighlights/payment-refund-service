# Postman API Onboarding – Implementation Guide

This document describes how the onboarding workflows were built, what is universal vs per-service, what the customer's ops team must configure, run instructions, validation, trade-offs, and AI usage. It supports the two onboarded services in this repo: **payment-refund-service** and **loan-origination-service**.

---

## 1. Sequential steps and what you need to do

Follow these in order so the implementation can proceed and run successfully.

| Step | What you do | Why / what happens next |
|------|-----------------|--------------------------|
| **1. Secrets on GitHub** | In the repo (e.g. `tsailifehighlights/cse-exercise`): **Settings → Secrets and variables → Actions**. Add: `POSTMAN_API_KEY`, `POSTMAN_ACCESS_TOKEN`. Optionally add `GH_FALLBACK_TOKEN` if workflow-file commits fail. | The workflows need these to call the Postman API and to push generated files. Without them, the action will fail or skip steps. |
| **2. Push this repo** | Commit and push so the repo on GitHub has: `.github/workflows/` at **repo root** with `postman-onboarding-payment-refund.yml` and `postman-onboarding-loan-origination.yml`, plus `payment-refund-service/`, `loan-origination-service/`, `claims-processing-service/` (each with `specs/`). | GitHub only discovers workflows from the repo root `.github/workflows/`. The `spec-url` in each workflow points at the **raw** URL of the spec in this repo. |
| **3. Confirm default branch** | Check the repo’s default branch (e.g. `main` or `master`). If it’s not `main`, edit the `spec-url` in each workflow file to use the correct branch name. | The runner fetches the spec from the raw URL; wrong branch returns 404. |
| **4. First run (payment-refund)** | In GitHub: **Actions → Postman onboarding (payment-refund)** → **Run workflow**. Wait for completion. | Bootstrap creates workspace and collections; repo-sync exports to the repo and may add `.github/workflows/ci.yml`. |
| **5. First run (loan-origination)** | **Actions → Postman onboarding (loan-origination)** → **Run workflow**. Wait for completion. | Same as above for the second service. |
| **6. Validate** | In Postman: confirm workspaces, Spec Hub entries, collections, environments. In repo: confirm `postman/` and `.postman/` (and optional `ci.yml`) in each service folder. | Ensures end-to-end behavior and that artifacts are present. |
| **7. Rerun (optional)** | Trigger one workflow again. Confirm it updates (e.g. spec/collections) and does not create duplicate workspaces. | Validates idempotent behavior. |

**What the agent cannot do for you:** Create or change GitHub secrets, push to your GitHub repo, or trigger workflows. You must perform steps 1, 2, 4, 5, and 7 yourself; the agent can only prepare the files and this guide.

---

## 2. How the workflow was built

- **Source:** The [postman-api-onboarding-action](https://github.com/postman-cs/postman-api-onboarding-action) README and `action.yml` were used as the single source of truth; no pre-made template repo was used.
- **Design:** One workflow file per service (`.github/workflows/postman-onboarding-payment-refund.yml` and `.github/workflows/postman-onboarding-loan-origination.yml` at repo root, with `working-directory` set to each service folder). Each workflow checks out the repo and runs the composite action once with that service’s inputs.
- **Trigger:** `workflow_dispatch` only, so runs are manual and predictable for the exercise. You can add a `push` trigger on `specs/**` later for automatic reruns when specs change.
- **Spec URL:** The action needs a URL that the runner can fetch. The spec is in this repo, so the workflow uses the **raw GitHub URL** for that file (e.g. `https://raw.githubusercontent.com/tsailifehighlights/cse-exercise/main/payment-refund-service/specs/payment-refund-api-openapi.yaml`). Replace `tsailifehighlights/cse-exercise` and `main` if your repo or default branch differs.

---

## 3. Decisions

- **One repo, multiple service folders:** This repo (`cse-exercise`) contains `payment-refund-service/`, `loan-origination-service/`, and `claims-processing-service/`. Workflows live at repo root (`.github/workflows/`); each uses `working-directory` so the action writes under the correct service folder. Specs live under each service's `specs/`. In production you might use one repo per service.
- **Single composite step:** The workflow does not call `postman-bootstrap-action` and `postman-repo-sync-action` separately; it uses `postman-api-onboarding-action@v0`, which runs both in order and passes bootstrap outputs into repo-sync.
- **Environments and URLs:** Environment slugs and runtime URLs in the workflows are taken from each OpenAPI spec’s `servers` block. Placeholder URLs (e.g. `https://api.payments.example.com/v2`) are fine for this exercise; no AWS or real backends are required.

---

## 4. Universal vs per-service

| Aspect | Universal (same for all services) | Per-service (changes per service) |
|--------|-----------------------------------|-----------------------------------|
| Workflow structure | One job: checkout → run composite action | — |
| Action | `postman-cs/postman-api-onboarding-action@v0` | — |
| Secrets | `POSTMAN_API_KEY`, `POSTMAN_ACCESS_TOKEN`, `GITHUB_TOKEN` | — |
| Permissions | `actions: write`, `contents: write`, `variables: write` | — |
| **project-name** | — | `payment-refund-api` vs `loan-origination-api` |
| **spec-url** | — | Path to this repo’s spec file (per service) |
| **domain** | — | `payments` vs `lending` |
| **domain-code** | — | `PMT` vs `LN` |
| **environments-json** | — | Payment: `["prod","uat","qa","dev"]`; Loan: `["prod","staging","dev"]` |
| **env-runtime-urls-json** | — | Map of env slug → base URL from each spec’s `servers` |

---

## 5. What the customer’s ops/platform team must configure

- **Secrets:** They must create and maintain `POSTMAN_API_KEY` and `POSTMAN_ACCESS_TOKEN` (and optionally `GH_FALLBACK_TOKEN`) in their GitHub org/repo. The access token expires and must be refreshed (e.g. via Postman CLI `postman login` and re-adding the secret).
- **Repo and branch:** Default branch must be known and reflected in `spec-url`; the spec file must exist at that path so the runner can fetch it.
- **Permissions:** The workflow needs `actions: write`, `contents: write`, and `variables: write`. If the action writes `.github/workflows/ci.yml`, the token used for that commit may need elevated scope (hence `GH_FALLBACK_TOKEN`).
- **Out of scope for this exercise:** AWS, GitLab, internal API gateways, or real runtime URLs. In production, ops would set real `env-runtime-urls-json` and, if applicable, `system-env-map-json` and governance mappings.

---

## 6. Run instructions

1. Ensure secrets are set (see §1).
2. Ensure the repo and branch in `spec-url` match your GitHub repo and default branch.
3. Go to **Actions** in the repo, select **Postman onboarding (payment-refund)** or **Postman onboarding (loan-origination)**, and click **Run workflow**.
4. First run: creates workspace, spec in Spec Hub, collections, environments, and exports into the repo (and may add `ci.yml`). Later runs: update spec/collections and sync repo again.

---

## 7. Validation

- **In Postman:** New workspace for the project; spec visible in Spec Hub; baseline, smoke, and contract collections present; environments created/updated.
- **In repo:** Under each service folder: `postman/` and `.postman/` with collection and environment JSON; optionally `.github/workflows/ci.yml`; workflow run shows outputs such as `workspace-id`, `commit-sha`, `repo-sync-summary-json`.

---

## 8. Trade-offs and issues

- **Access token expiry:** Without a valid `POSTMAN_ACCESS_TOKEN`, Bifrost workspace linking and governance steps are skipped; the rest of the action still runs. Refresh the token and update the secret when needed.
- **Workflow-file commits:** If the action fails to commit `.github/workflows/ci.yml`, try adding `GH_FALLBACK_TOKEN` with a token that has workflow write scope.
- **Branch in spec-url:** If the repo uses `master` (or another default branch), update `spec-url` in both workflow files accordingly.
- **Rerun behavior:** Re-running the workflow updates the existing workspace and spec/collections; it does not create a second workspace when the same `project-name` and workspace are used.

---

## 9. AI usage

- **AI-generated:** Workflow YAML structure and input tables; initial draft of this implementation guide (sections 2–9); wording for “Sequential steps and what you need to do.”
- **Human-validated or written:** Final choice of `project-name`, `domain`, `domain-code`, and environment slugs; exact `spec-url` format and branch; secret names and permissions; run/validation checklist; any fixes after a real run (e.g. permission or token issues).
