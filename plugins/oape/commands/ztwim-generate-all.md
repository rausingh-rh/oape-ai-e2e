---
description: Generate all ZTWIM PR test artifacts in one run (test scenarios, execution steps, e2e Go code)
argument-hint: "<pr-url> [--output <path>] [--env <cluster-type>]"
---

## Name
oape:ztwim-generate-all

## Synopsis
```
/oape:ztwim-generate-all <pr-url> [--output <path>] [--env <cluster-type>]
```

## Description

**Single command** that analyzes a ZTWIM operator Pull Request and generates **all three outputs** in one run:

1. **test-cases.md** — Test scenarios (what to test, PR-specific focus, verification, cleanup).
2. **execution-steps.md** — Step-by-step procedure with executable `oc` commands (prerequisites, install, stack, verify, PR-specific steps, cleanup).
3. **\<prno\>_test_e2e.go** — Go e2e test code (Ginkgo Describe/Context/It) for the ZTWIM repo `test/e2e/`, with operator/operand scenarios and PR-specific tests.

All files are written into **one output directory**: `<output-dir>/ztwim_pr_<number>/`. Default `<output-dir>` is `output` (create if missing). Use `--output <path>` to set a different base.

- **Repository**: openshift/zero-trust-workload-identity-manager only.
- **Install/Stack**: Use plugin fixtures (`plugins/oape/ztwim-test-generator/fixtures/operator-install.yaml`, `plugins/oape/ztwim-test-generator/fixtures/ztwim-stack.yaml`). Do not discover from repo.
- **E2E**: Follow upstream test/e2e structure; see plugin [docs/e2e-structure.md](../ztwim-test-generator/docs/e2e-structure.md) and [fixtures/e2e-important-scenarios.md](../ztwim-test-generator/fixtures/e2e-important-scenarios.md).

## Implementation

### Step 1: Validate PR and Analyze Changes (Once)

1. **Validate PR URL** is for ZTWIM: `https://github.com/openshift/zero-trust-workload-identity-manager/pull/<number>`. If not, inform the user this command is for ZTWIM PRs only.
2. **Use browser tools**: Navigate to the PR URL, then to "Files changed" (append `/files`). Use **browser_snapshot** to read PR description and changed files. Do **not** use `gh` CLI.
3. Extract PR number and (optionally) a short description from the title. Map changed files to:
   - Test focus (API types, CRD, controller, RBAC, samples, e2e).
   - E2E focus (operator/operand scenarios, PR-specific It blocks).

### Step 2: Output Directory

- **Path**: `<output-dir>/ztwim_pr_<number>/` (e.g. `output/ztwim_pr_72/`).
- **Default `<output-dir>`**: `output` (relative to workspace root). Create the directory if it does not exist.
- **With `--output <path>`**: Use `<path>` as the base; write into `<path>/ztwim_pr_<number>/`.
- **With `--env`**: Use only when generating execution-steps content (e.g. env-specific notes); optional.

### Step 3: Generate test-cases.md

Write **test-cases.md** into the output directory. Content must include:

- Operator info (ZTWIM), repository.
- Prerequisites (cluster, env vars: APP_DOMAIN, JWT_ISSUER_ENDPOINT, CLUSTER_NAME).
- Install: fixture path, `oc apply -f .../operator-install.yaml`, wait for CSV and deployment.
- Stack: fixture path, envsubst, `envsubst < .../ztwim-stack.yaml | oc apply -f -`.
- PR-specific test cases derived from Files changed (field tests, controller tests, validation).
- Verification (oc get CRs, oc wait, oc logs in zero-trust-workload-identity-manager).
- Cleanup order: SpireOIDCDiscoveryProvider → SpiffeCSIDriver → SpireAgent → SpireServer → ZeroTrustWorkloadIdentityManager → subscription → CSV → OperatorGroup → namespace.

### Step 4: Generate execution-steps.md

Write **execution-steps.md** into the output directory. Content must include:

- Prerequisites: `which oc`, `oc version`, `oc whoami`, `oc get nodes`, `oc get clusterversion`, packagemanifests check, then APP_DOMAIN/JWT_ISSUER_ENDPOINT/CLUSTER_NAME and echo.
- Install: `oc apply -f <path-to>/operator-install.yaml`, then oc wait for CSV and deployment, oc get pods.
- Stack: envsubst and oc apply for ztwim-stack.yaml.
- CR verification: oc get all ZTWIM CRs, oc wait for Ready.
- PR-specific execution steps (from Files changed).
- Cleanup: full oc delete sequence in fixed order.

Use plugin fixture paths; document `<path-to>` as `plugins/oape/ztwim-test-generator/fixtures`.

### Step 5: Generate \<prno\>_test_e2e.go and e2e-suggestions.md

**e2e-suggestions.md** (in the same output directory):

- Short list of which operator/operand e2e scenarios apply.
- Which are highly recommended for this PR.
- PR-specific It block suggestions.

**\<prno\>_test_e2e.go** (in the same output directory):

- Package `e2e`; same imports and style as upstream (see [docs/e2e-structure.md](../ztwim-test-generator/docs/e2e-structure.md), [fixtures/e2e-sample_test.go.example](../ztwim-test-generator/fixtures/e2e-sample_test.go.example)).
- Use `k8sClient`, `clientset`, `testCtx`, `utils.*`; no BeforeSuite/TestE2E.
- Describe/Context/It blocks with `By("…")`; comment each It (e.g. `// PR-suggested: ...`) for pick-and-choose.
- Include important scenarios (install, recovery, operand conditions, ZTWIM aggregation, OperatorCondition Upgradeable, CR-driven config) and PR-specific tests from Files changed.

### Step 6: Confirm Output

Tell the user the output directory path and list the three (or four) generated files: test-cases.md, execution-steps.md, \<prno\>_test_e2e.go, and optionally e2e-suggestions.md.

## Arguments

- **$1 (pr-url)**: ZTWIM operator GitHub PR URL — `https://github.com/openshift/zero-trust-workload-identity-manager/pull/<number>`.
- **--output**: Output base directory (optional). Default: `output`. All files go in `<output>/ztwim_pr_<number>/`.
- **--env**: Target environment for execution-steps (optional): `aws`, `gcp`, `azure`, `vsphere`, `baremetal`.

## Examples

```
/oape:ztwim-generate-all https://github.com/openshift/zero-trust-workload-identity-manager/pull/72
# Writes: output/ztwim_pr_72/test-cases.md
#         output/ztwim_pr_72/execution-steps.md
#         output/ztwim_pr_72/72_test_e2e.go
#         output/ztwim_pr_72/e2e-suggestions.md

/oape:ztwim-generate-all https://github.com/openshift/zero-trust-workload-identity-manager/pull/72 --output .work
# Writes: .work/ztwim_pr_72/test-cases.md, .work/ztwim_pr_72/execution-steps.md, .work/ztwim_pr_72/72_test_e2e.go, .work/ztwim_pr_72/e2e-suggestions.md
```

## Notes

- **ZTWIM only**: For openshift/zero-trust-workload-identity-manager PRs only.
- **Single PR analysis**: Browser is used once for the PR and Files changed; the same analysis drives all three outputs.
- **Fixtures**: Install and stack content come from `plugins/oape/ztwim-test-generator/fixtures`; do not discover from repo.
