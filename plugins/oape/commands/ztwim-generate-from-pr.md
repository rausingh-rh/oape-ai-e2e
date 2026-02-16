---
description: Generate test cases with oc commands for ZTWIM operator PRs
argument-hint: "<pr-url> [--output <path>]"
---

## Name
oape:ztwim-generate-from-pr

## Synopsis
```
/oape:ztwim-generate-from-pr <pr-url> [--output <path>]
```

## Description

Analyzes a **ZTWIM (Zero Trust Workload Identity Manager)** operator Pull Request and generates test cases with executable `oc` commands. Uses fixed install and stack fixtures for ZTWIM and derives PR-specific test steps from the "Files changed" tab.

- **Repository**: openshift/zero-trust-workload-identity-manager only
- **Install**: Plugin fixture `plugins/oape/ztwim-test-generator/fixtures/operator-install.yaml` (OLM: namespace, OperatorGroup, Subscription)
- **Stack**: Plugin fixture `plugins/oape/ztwim-test-generator/fixtures/ztwim-stack.yaml` (5 CRs, applied via envsubst)
- **PR-specific tests**: Generated from the PR's Files changed (API types, controllers, CRDs, samples)

## Implementation

### Step 1: Validate PR and Analyze Changes

**IMPORTANT: Use browser tools to analyze the PR. Do NOT use `gh` CLI.**

1. **Validate PR URL** is for the ZTWIM repo: `https://github.com/openshift/zero-trust-workload-identity-manager/pull/<number>`. If a different repo is given, inform the user that this command is for ZTWIM PRs only.
2. **Use browser_navigate** to open the PR URL.
3. **Use browser_snapshot** to read PR description.
4. **Navigate to "Files changed"** (append `/files` to PR URL) and use **browser_snapshot** to read changed files.

Map file patterns to test focus:

| File Pattern | Category | Test Focus |
|--------------|----------|------------|
| `api/**/*_types.go` | API Types | New fields, validation on ZTWIM CRs |
| `config/crd/**/*.yaml` | CRD Changes | Schema updates |
| `*controller*.go`, `*reconcile*.go` | Controller | Reconciliation logic |
| `config/rbac/*.yaml` | RBAC | Permission changes |
| `config/samples/*.yaml` | Samples | Example CR usage |
| `test/e2e/**` | E2E Tests | Test patterns |

### Step 2: Use Fixed Install (Do Not Discover from Repo)

**Install** comes from the plugin fixture. Do not discover CSV/CRDs from the repo.

- **Fixture path**: `plugins/oape/ztwim-test-generator/fixtures/operator-install.yaml` (relative to workspace root).
- **Command** to document in output:
  ```bash
  oc apply -f <path-to>/operator-install.yaml
  ```
  Where `<path-to>` is the path to the plugin's `fixtures/` directory (e.g. `plugins/oape/ztwim-test-generator/fixtures`).

Contents: namespace `zero-trust-workload-identity-manager` (label `openshift.io/cluster-monitoring: "true"`), OperatorGroup `zero-trust-workload-identity-manager-og`, Subscription `openshift-zero-trust-workload-identity-manager` (channel `stable-v1`, source `redhat-operators`).

### Step 3: Use Fixed Stack (Do Not Discover from Repo)

**Stack** comes from the plugin fixture.

- **Fixture path**: `plugins/oape/ztwim-test-generator/fixtures/ztwim-stack.yaml`.
- **Prerequisites** (required env vars):
  ```bash
  export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
  export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
  export CLUSTER_NAME=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
  # Or set CLUSTER_NAME to a test value e.g. test01
  ```
- **Command** to document:
  ```bash
  envsubst < <path-to>/ztwim-stack.yaml | oc apply -f -
  ```

The stack contains 5 CRs in order: ZeroTrustWorkloadIdentityManager, SpireServer, SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider (all named `cluster`).

### Step 4: Generate PR-Specific Test Cases

Based on "Files changed", generate focused test steps for ZTWIM CRs. Use the ZTWIM CR kinds and namespace `zero-trust-workload-identity-manager`. Prefer patterns from the skill (field test, controller test) adapted to ZTWIM.

### Step 5: Verification and Cleanup

- **Verification**: Include standard ZTWIM checks (e.g. oc get zerotrustworkloadidentitymanager,spireserver,..., oc wait, oc logs in zero-trust-workload-identity-manager).
- **Cleanup order**: SpireOIDCDiscoveryProvider → SpiffeCSIDriver → SpireAgent → SpireServer → ZeroTrustWorkloadIdentityManager → subscription → CSV → OperatorGroup → namespace.

### Step 6: Output Generation

**All generated files go inside a single output directory.**

- **Output directory**: `<output-dir>/ztwim_pr_<number>/` (e.g. `output/ztwim_pr_72/`).
- **Default `<output-dir>`**: `output` (relative to workspace root). Create it if it does not exist.
- **With `--output <path>`**: Use `<path>` as the output base; write into `<path>/ztwim_pr_<number>/`.

Generate **test-cases.md** inside that directory. Content: Operator info (ZTWIM), Prerequisites, Install (fixture), Stack (prereqs + envsubst), PR-specific test cases, Verification, Cleanup.

## Arguments

- **$1 (pr-url)**: ZTWIM operator GitHub PR URL — `https://github.com/openshift/zero-trust-workload-identity-manager/pull/<number>`
- **--output**: Output base directory (optional). Default: `output`. Generated files go in `<output>/ztwim_pr_<number>/`.

## Examples

```
/oape:ztwim-generate-from-pr https://github.com/openshift/zero-trust-workload-identity-manager/pull/72
# Writes: output/ztwim_pr_72/test-cases.md

/oape:ztwim-generate-from-pr https://github.com/openshift/zero-trust-workload-identity-manager/pull/72 --output .work
# Writes: .work/ztwim_pr_72/test-cases.md
```

## Notes

- **ZTWIM only**: This command is for openshift/zero-trust-workload-identity-manager PRs only.
- **Use browser tools** for PR description and Files changed — not `gh` CLI.
- **Install and stack** are from `plugins/oape/ztwim-test-generator/fixtures`; do not discover from repo.
