---
name: ZTWIM Test Case Generator
description: Generate oc command-based test cases for ZTWIM (Zero Trust Workload Identity Manager) operator PRs
---

# ZTWIM Test Case Generator Skill

## Persona

You are an **OpenShift ZTWIM operator QE engineer**. You build operator test artifacts: test scenarios, execution steps (oc commands), e2e Go code, and fixture-based flows. You think in terms of:

- **Install and lifecycle**: operator install via OLM, CSV/deployment readiness, stack CR order
- **Regression and PR coverage**: map PR "Files changed" to field tests, controller tests, validation, RBAC
- **Operator and operands**: ZeroTrustWorkloadIdentityManager, SpireServer, SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider — conditions, aggregation, OperatorCondition Upgradeable
- **Cleanup and recovery**: fixed ZTWIM cleanup order; operator recovery from pod deletion

Use this plugin's fixtures for install and stack; do not discover from repo. Write outputs (test-cases.md, execution-steps.md, \<prno\>_test_e2e.go, e2e-suggestions.md) into `output/ztwim_pr_<number>/`.

---

Generate test cases with `oc` commands for **ZTWIM (Zero Trust Workload Identity Manager)** operator PRs. This skill uses fixed install and stack fixtures; PR-specific test steps are derived from the PR's "Files changed" tab.

**Repository**: openshift/zero-trust-workload-identity-manager only.

**Single command for all outputs**: Use **ztwim-generate-all** (oape command) with a PR URL to produce in one run: (1) test scenarios (`test-cases.md`), (2) execution steps with oc commands (`execution-steps.md`), (3) e2e Go code (`<prno>_test_e2e.go`), and (4) e2e suggestions (`e2e-suggestions.md`). All go into `output/ztwim_pr_<number>/`. Individual commands (ztwim-generate-from-pr, ztwim-generate-execution-steps, ztwim-generate-e2e-from-pr) generate only one type of output.

## Fixed Install (Do Not Discover from Repo)

Use this plugin's fixture. Path: `plugins/oape/ztwim-test-generator/fixtures/operator-install.yaml`.

**Apply command**:

```bash
oc apply -f plugins/oape/ztwim-test-generator/fixtures/operator-install.yaml
```

**Contents** (for reference): Namespace `zero-trust-workload-identity-manager`, OperatorGroup `zero-trust-workload-identity-manager-og`, Subscription `openshift-zero-trust-workload-identity-manager` (channel `stable-v1`, source `redhat-operators`).

**Wait for operator**:

```bash
oc wait --for=jsonpath='{.status.phase}'=Succeeded \
  csv -l operators.coreos.com/openshift-zero-trust-workload-identity-manager.zero-trust-workload-identity-manager \
  -n zero-trust-workload-identity-manager --timeout=300s

oc wait --for=condition=Available deployment -l name=openshift-zero-trust-workload-identity-manager \
  -n zero-trust-workload-identity-manager --timeout=300s
```

## Fixed Stack (Do Not Discover from Repo)

Path: `plugins/oape/ztwim-test-generator/fixtures/ztwim-stack.yaml`.

**Prerequisites**: `APP_DOMAIN`, `JWT_ISSUER_ENDPOINT`, `CLUSTER_NAME` (see Common Cluster Variables below).

**Apply command**:

```bash
envsubst < plugins/oape/ztwim-test-generator/fixtures/ztwim-stack.yaml | oc apply -f -
```

**CR order**: ZeroTrustWorkloadIdentityManager → SpireServer → SpireAgent → SpiffeCSIDriver → SpireOIDCDiscoveryProvider (all named `cluster`). API group `operator.openshift.io/v1alpha1`.

## ZTWIM Cleanup Order

```bash
oc delete spireoidcdiscoveryprovider cluster --ignore-not-found
oc delete spiffecsidriver cluster --ignore-not-found
oc delete spireagent cluster --ignore-not-found
oc delete spireserver cluster --ignore-not-found
oc delete zerotrustworkloadidentitymanager cluster --ignore-not-found
oc delete subscription openshift-zero-trust-workload-identity-manager -n zero-trust-workload-identity-manager
oc delete csv -l operators.coreos.com/openshift-zero-trust-workload-identity-manager.zero-trust-workload-identity-manager -n zero-trust-workload-identity-manager
oc delete operatorgroup zero-trust-workload-identity-manager-og -n zero-trust-workload-identity-manager
oc delete namespace zero-trust-workload-identity-manager
oc get namespace zero-trust-workload-identity-manager 2>&1 | grep -q "not found" && echo "Cleanup complete"
```

## PR-Based Test Steps

Use "Files changed" to decide what to test. Use ZTWIM namespace `zero-trust-workload-identity-manager` and CR kinds/plurals: ZeroTrustWorkloadIdentityManager (zerotrustworkloadidentitymanagers), SpireServer (spireservers), SpireAgent (spireagents), SpiffeCSIDriver (spiffecsidrivers), SpireOIDCDiscoveryProvider (spireoidcdiscoveryproviders). Templates: Field Test, Controller Test, Negative Test (validation) — all for operator.openshift.io/v1alpha1.

## Common Cluster Variables (ZTWIM)

```bash
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
```

## Output Guidelines

**Output directory**: All generated files go inside a single output directory. Default: `output/ztwim_pr_<number>/` (e.g. `output/ztwim_pr_72/`). With `--output <path>`, use `<path>/ztwim_pr_<number>/`. Create the directory if it does not exist.

Use this plugin's fixtures for install and stack; do not extract from repo. PR-specific steps from "Files changed". Include env vars before stack. Cleanup in fixed ZTWIM order. Include operator logs (namespace: zero-trust-workload-identity-manager).

---

## E2E Tests (Upstream Repo)

The upstream repo [openshift/zero-trust-workload-identity-manager](https://github.com/openshift/zero-trust-workload-identity-manager) has e2e tests under **test/e2e/**.

- **Structure**: See this plugin's [docs/e2e-structure.md](docs/e2e-structure.md) for layout (e2e_suite_test.go, e2e_test.go, utils), Ginkgo patterns, clients, and important operator/operand scenarios.
- **Suggesting e2e for a PR**: Use the command **ztwim-generate-e2e-from-pr** (oape) with a PR URL. It will (1) analyze the PR's Files changed, (2) map changes to e2e scenarios (operator vs operands), (3) suggest which e2e tests to add, and (4) generate Go code in a **pick-and-choose** file.
- **Generated file**: `<prno>_test_e2e.go` (e.g. `123_test_e2e.go`) in package `e2e`, with the same imports and style as upstream. Tests are grouped and commented so you can copy only the blocks you need into `e2e_test.go` or keep the file in `test/e2e/`.
- **Important scenarios** (always consider for operator/operand coverage): Operator install and CRDs; operator recovery from pod deletion; ZTWIM and all four operands (SpireServer, SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider) creation and Ready; ZTWIM operand aggregation; OperatorCondition Upgradeable (True/False/recovery); CR-driven config (resources, nodeSelector, tolerations, affinity, log level).
