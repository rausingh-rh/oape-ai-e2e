---
description: Generate step-by-step test execution procedure for ZTWIM operator PRs
argument-hint: "<pr-url> [--output <path>] [--env <cluster-type>]"
---

## Name
oape:ztwim-generate-execution-steps

## Synopsis
```
/oape:ztwim-generate-execution-steps <pr-url> [--output <path>] [--env <cluster-type>]
```

## Description

Generates a complete test execution procedure for **ZTWIM (Zero Trust Workload Identity Manager)** operator PRs. Uses fixed install and stack fixtures; PR-specific steps are derived from the PR's Files changed. Repository: openshift/zero-trust-workload-identity-manager only.

## Implementation

### Step 1: Validate PR and Optionally Analyze Changes

1. **Validate PR URL** is for ZTWIM: `https://github.com/openshift/zero-trust-workload-identity-manager/pull/<number>`. If not, inform the user this command is for ZTWIM PRs only.
2. Optionally use **browser_navigate** and **browser_snapshot** to read PR description and "Files changed" (append `/files` to PR URL) to generate PR-specific execution steps.

### Step 2: Generate Prerequisites

```bash
which oc
oc version
oc whoami
oc get nodes
oc get clusterversion
oc get packagemanifests -n openshift-marketplace | grep -i zero-trust

# ZTWIM stack prerequisites
export APP_DOMAIN=apps.$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
export JWT_ISSUER_ENDPOINT=oidc-discovery.${APP_DOMAIN}
export CLUSTER_NAME=$(oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
# Or: export CLUSTER_NAME=test01

echo "APP_DOMAIN: $APP_DOMAIN"
echo "JWT_ISSUER_ENDPOINT: $JWT_ISSUER_ENDPOINT"
echo "CLUSTER_NAME: $CLUSTER_NAME"
```

### Step 3: Generate Operator Installation (Fixed Fixture)

Use this plugin's fixture. **Fixture**: `plugins/oape/ztwim-test-generator/fixtures/operator-install.yaml`

```bash
oc apply -f <path-to>/operator-install.yaml
```

```bash
oc wait --for=jsonpath='{.status.phase}'=Succeeded \
  csv -l operators.coreos.com/openshift-zero-trust-workload-identity-manager.zero-trust-workload-identity-manager \
  -n zero-trust-workload-identity-manager --timeout=300s

oc get pods -n zero-trust-workload-identity-manager
oc wait --for=condition=Available deployment -l name=openshift-zero-trust-workload-identity-manager \
  -n zero-trust-workload-identity-manager --timeout=300s
```

### Step 4: Generate Stack Deployment (Fixed Fixture)

**Fixture**: `plugins/oape/ztwim-test-generator/fixtures/ztwim-stack.yaml`

Ensure APP_DOMAIN, JWT_ISSUER_ENDPOINT, CLUSTER_NAME are set, then:

```bash
envsubst < <path-to>/ztwim-stack.yaml | oc apply -f -
```

### Step 5: Generate CR Verification

```bash
oc get zerotrustworkloadidentitymanager,spireserver,spireagent,spiffecsidriver,spireoidcdiscoveryprovider
oc wait --for=condition=Ready zerotrustworkloadidentitymanager/cluster --timeout=120s
```

### Step 6: Generate PR-Specific Steps

Based on Files changed: API/field changes, controller changes, RBAC changes. Use namespace `zero-trust-workload-identity-manager` and CR names/kinds from the stack.

### Step 7: Generate Cleanup

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

### Step 8: Output

**All generated files go inside a single output directory.**

- **Output directory**: `<output-dir>/ztwim_pr_<number>/` (e.g. `output/ztwim_pr_72/`).
- **Default `<output-dir>`**: `output` (relative to workspace root). Create it if it does not exist.
- **With `--output <path>`**: Use `<path>` as the output base; write into `<path>/ztwim_pr_<number>/`.

Write **execution-steps.md** inside that directory. Content: Prerequisites, Install (fixture), Stack (env vars + envsubst), Verify, PR-specific steps (if any), Cleanup.

## Arguments

- **$1 (pr-url)**: ZTWIM operator GitHub PR URL
- **--output**: Output base directory (optional). Default: `output`. Generated files go in `<output>/ztwim_pr_<number>/`.
- **--env**: Target environment (optional): `aws`, `gcp`, `azure`, `vsphere`, `baremetal`

## Examples

```
/oape:ztwim-generate-execution-steps https://github.com/openshift/zero-trust-workload-identity-manager/pull/72
# Writes: output/ztwim_pr_72/execution-steps.md

/oape:ztwim-generate-execution-steps https://github.com/openshift/zero-trust-workload-identity-manager/pull/72 --output .work
# Writes: .work/ztwim_pr_72/execution-steps.md
```

## Notes

- **ZTWIM only**: For openshift/zero-trust-workload-identity-manager PRs only.
- **Install and stack** come from `plugins/oape/ztwim-test-generator/fixtures`; do not extract from repo.
- **Cleanup order** is fixed: CRs (SpireOIDCDiscoveryProvider → … → ZeroTrustWorkloadIdentityManager), then subscription, CSV, OperatorGroup, namespace.
