---
description: Suggest e2e tests and generate Go code for ZTWIM operator PRs from upstream e2e structure
argument-hint: "<pr-url> [--output <path>]"
---

## Name
oape:ztwim-generate-e2e-from-pr

## Synopsis
```
/oape:ztwim-generate-e2e-from-pr <pr-url> [--output <path>]
```

## Description

Analyzes a **ZTWIM** operator Pull Request and (1) suggests which e2e tests to add based on the upstream [test/e2e](https://github.com/openshift/zero-trust-workload-identity-manager/tree/main/test/e2e) structure, and (2) generates **Go e2e test code** in a file **`<prno>_test_e2e.go`** (e.g. `123_test_e2e.go`) that you can copy from into the ZTWIM repo. Tests are written with **operator and operand context** and include **important scenarios** that should be highly checked.

- **Repository**: openshift/zero-trust-workload-identity-manager only.
- **Upstream e2e**: test/e2e (e2e_suite_test.go, e2e_test.go, utils). See plugin [docs/e2e-structure.md](../ztwim-test-generator/docs/e2e-structure.md).

## Implementation

### Step 1: Validate PR and Analyze Changes

1. **Validate PR URL** is for ZTWIM: `https://github.com/openshift/zero-trust-workload-identity-manager/pull/<number>`. If not, inform the user this command is for ZTWIM PRs only.
2. **Use browser tools**: Navigate to the PR, then to "Files changed" (append `/files` to PR URL). Use **browser_snapshot** to read changed files and paths.

Map file patterns to e2e focus:

| File Pattern | E2E Focus |
|--------------|-----------|
| `api/**/*_types.go` | New/updated CR fields; consider new or updated It specs for CR create/update and condition checks. |
| `*controller*.go`, `*reconcile*.go` | Reconciliation and operand lifecycle; OperatorCondition; recovery tests. |
| `config/crd/**` | Schema/validation; optional negative It (invalid CR). |
| `config/samples/*.yaml` | Example CR usage; align generated test CRs with samples. |
| `test/e2e/**` | Follow existing Describe/Context/It style; suggest similar or extended scenarios. |

### Step 2: Use Upstream E2E Structure

Read this plugin's [docs/e2e-structure.md](../ztwim-test-generator/docs/e2e-structure.md). Generated code must:

- **Package**: `e2e`.
- **Imports**: Same as upstream (e.g. `context`, `fmt`, Ginkgo, `operatorv1alpha1`, `test/e2e/utils`, `corev1`, `metav1`, `client`, etc.). Do not add unused imports.
- **Clients**: Use suite-level `k8sClient`, `clientset`, `apiextClient`, `configClient` and per-test `testCtx` (from `BeforeEach` with timeout). Assume these exist; do not redefine.
- **Helpers**: Use `utils.*` (e.g. `utils.OperatorNamespace`, `utils.WaitForSpireServerConditions`, `utils.WaitForDeploymentAvailable`). Match names from upstream constants.go and utils.go.
- **Style**: `Describe` / `Context` / `It`; `By("…")` for steps; `DeferCleanup` for teardown; `Eventually` with `WithTimeout`/`WithPolling` where appropriate. For a code-style reference, see plugin fixture [fixtures/e2e-sample_test.go.example](../ztwim-test-generator/fixtures/e2e-sample_test.go.example). For a scenario checklist, see [fixtures/e2e-important-scenarios.md](../ztwim-test-generator/fixtures/e2e-important-scenarios.md).

### Step 3: Suggest E2E Tests (Summary)

Produce a short **suggestion list** for the PR:

- Which **operator** scenarios apply (install, CRDs, pod recovery, log level via Subscription).
- Which **operand** scenarios apply (SpireServer, SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider: create, conditions, Ready).
- **ZTWIM** aggregation (operand status, Ready).
- **OperatorCondition** Upgradeable (True when healthy; False when operand down; recovery).
- **CR-driven config** (resources, nodeSelector, tolerations, affinity, log level) for any CR touched by the PR.
- Any **PR-specific** It blocks (new fields, new validation, changed behavior).

Mark which scenarios are **highly recommended** for this PR given the Files changed.

### Step 4: Generate `<prno>_test_e2e.go`**

Generate a single Go file named **`<prno>_test_e2e.go`** (e.g. `123_test_e2e.go`). The file must be **self-contained** for the package `e2e` and assume the suite (e2e_suite_test.go) and utils are present.

- **Header**: Standard Apache-2.0 copyright and package `e2e`.
- **Imports**: Only what is needed; match upstream style.
- **Structure**: One or more `Describe` or `Context` blocks. Each `It` should be **commented** with a short line (e.g. `// PR-suggested: operator recovery`) so the user can easily pick and choose which tests to copy into `e2e_test.go` or keep in this file.
- **Content**: Include both (a) **important scenarios** from [docs/e2e-structure.md](../ztwim-test-generator/docs/e2e-structure.md) that are relevant to the PR, and (b) **PR-specific** tests derived from Files changed. Prefer reusing existing utils and condition names; if the PR adds new API fields or conditions, generate plausible code and add a comment that the condition/field name may need to match the actual API.
- **No duplicate suite logic**: Do not define `BeforeSuite`, `TestE2E`, or client setup; only test blocks that run inside the existing suite.

### Step 5: Important Scenarios to Include (when relevant)

Always consider including or suggesting tests for:

1. Operator installed; all managed CRDs Established; operator Deployment Available.
2. Operator recovers from force pod deletion (new pod Running, deployment Available again).
3. ZeroTrustWorkloadIdentityManager created with trust domain, cluster name, bundle ConfigMap.
4. SpireServer / SpireAgent / SpiffeCSIDriver / SpireOIDCDiscoveryProvider created and respective conditions (e.g. StatefulSetAvailable, DaemonSetAvailable, DeploymentAvailable, Ready) True.
5. ZeroTrustWorkloadIdentityManager aggregates 4 operands; each operand Ready in status.
6. OperatorCondition Upgradeable: True when healthy; False when an operand pod is deleted; True again after recovery.
7. CR-driven configuration: SpireServer (or other operand) resources, nodeSelector, tolerations, affinity, log level; operator log level via Subscription env.

Generate at least 2–3 **highly recommended** It blocks that directly exercise code or CRs touched by the PR.

### Step 6: Output

**All generated files go inside a single output directory.**

- **Output directory**: `<output-dir>/ztwim_pr_<number>/` (e.g. `output/ztwim_pr_123/`).
- **Default `<output-dir>`**: `output` (relative to workspace root). Create it if it does not exist.
- **With `--output <path>`**: Use `<path>` as the output base; write into `<path>/ztwim_pr_<number>/`.

- **Suggestion list**: Write **e2e-suggestions.md** inside that directory (and optionally show in reply). Content: which operator/operand scenarios apply, highly recommended tests, PR-specific It blocks.
- **Go file**: Write **`<prno>_test_e2e.go`** inside that directory (e.g. `output/ztwim_pr_123/123_test_e2e.go`). Tell the user they can copy this file into the ZTWIM repo under `test/e2e/` or copy individual It blocks into `e2e_test.go`.

## Arguments

- **$1 (pr-url)**: ZTWIM operator GitHub PR URL — `https://github.com/openshift/zero-trust-workload-identity-manager/pull/<number>`.
- **--output**: Output base directory (optional). Default: `output`. Generated files go in `<output>/ztwim_pr_<number>/`.

## Examples

```
/oape:ztwim-generate-e2e-from-pr https://github.com/openshift/zero-trust-workload-identity-manager/pull/123
# Writes: output/ztwim_pr_123/123_test_e2e.go, output/ztwim_pr_123/e2e-suggestions.md

/oape:ztwim-generate-e2e-from-pr https://github.com/openshift/zero-trust-workload-identity-manager/pull/123 --output .work
# Writes: .work/ztwim_pr_123/123_test_e2e.go, .work/ztwim_pr_123/e2e-suggestions.md
```

## Notes

- **ZTWIM only**: For openshift/zero-trust-workload-identity-manager PRs only.
- **Browser**: Use browser tools for PR and Files changed; do not rely on gh CLI for file list.
- **Pick-and-choose**: The generated Go file is meant for choosing which tests to add; comments in the file should make it easy to copy only the needed blocks.
