# ZTWIM Upstream E2E Test Structure

This document describes the e2e test layout in [openshift/zero-trust-workload-identity-manager](https://github.com/openshift/zero-trust-workload-identity-manager) so the plugin can suggest and generate compatible e2e tests for new PRs.

## Repository and Paths

- **Repo**: <https://github.com/openshift/zero-trust-workload-identity-manager>
- **E2E root**: `test/e2e/`
- **Utils**: `test/e2e/utils/` (constants, helpers)

## File Layout

| Path | Purpose |
| ---- | ------- |
| `test/e2e/e2e_suite_test.go` | Suite setup: kubeconfig, scheme (operator API + operator-framework), clients (controller-runtime, clientset, apiext, config). `TestE2E(t)` entrypoint with Ginkgo. |
| `test/e2e/e2e_test.go` | Main e2e specs: `Describe("Zero Trust Workload Identity Manager", Ordered, …)` with nested `Context` and `It`. |
| `test/e2e/utils/constants.go` | Namespace, deployment/StatefulSet/DaemonSet names, label selectors, ConfigMap names, timeouts. |
| `test/e2e/utils/utils.go` | Helpers: `GetClusterBaseDomain`, `FindOperatorSubscription`, `WaitForCRDEstablished`, `WaitForDeploymentAvailable`, `WaitForSpireServerConditions`, etc. |

## Conventions

- **Package**: `e2e`.
- **Framework**: Ginkgo v2 (`Describe`, `Context`, `It`, `BeforeAll`, `BeforeEach`, `By`, `DeferCleanup`, `Eventually`).
- **API**: `operatorv1alpha1` from `github.com/openshift/zero-trust-workload-identity-manager/api/v1alpha1`.
- **Clients** (set in suite): `k8sClient` (controller-runtime), `clientset` (kubernetes.Interface), `apiextClient`, `configClient`. Use `testCtx` (timeout from `context.WithTimeout`) in tests.
- **Constants**: Use `utils.OperatorNamespace`, `utils.SpireServerStatefulSetName`, `utils.DefaultTimeout`, etc., from `test/e2e/utils`.

## Test Structure (operator and operands)

1. **BeforeAll**: Get cluster base domain, set `appDomain`, `jwtIssuer`, `clusterName`, find Subscription and OperatorCondition name.
2. **BeforeEach**: Create `testCtx` with timeout; `DeferCleanup(cancel)`.
3. **Context("Installation")**: Operator install, CRDs established, ZTWIM/SpireServer/SpireAgent/SpiffeCSIDriver/SpireOIDCDiscoveryProvider creation and Ready conditions, operand aggregation on ZTWIM, operator recovery from pod deletion.
4. **Context("OperatorCondition")**: Upgradeable True when healthy; False when SPIRE Server (or multiple operands) down; recovery to True.
5. **Context("Common configurations")**: Subscription log level; SpireServer resources, nodeSelector/tolerations, affinity, log level; similar for other operands where applicable.

## Important Scenarios to Cover (highly recommended)

- Operator installed and all managed CRDs Established.
- Operator recovers from force pod deletion (new pod Running, deployment Available).
- ZeroTrustWorkloadIdentityManager created with global config (trust domain, cluster name, bundle ConfigMap).
- SpireServer created and conditions (e.g. StatefulSetAvailable, Ready) True; StatefulSet Ready.
- SpireAgent created and conditions True; DaemonSet Available.
- SpiffeCSIDriver created and conditions True; DaemonSet Available.
- SpireOIDCDiscoveryProvider created and conditions True; Deployment Available.
- ZeroTrustWorkloadIdentityManager aggregates status: 4 operands (SpireServer, SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider), each Ready.
- OperatorCondition Upgradeable: True when all operands ready; False when an operand pod is deleted; True again after recovery.
- CR-driven configuration: SpireServer resources, nodeSelector, tolerations, affinity, log level; operator log level via Subscription env.

## Generated file naming

For a PR number `prno`, the plugin can generate a **pick-and-choose** Go file:

- **Filename**: `test/e2e/<prno>_test_e2e.go` (e.g. `123_test_e2e.go`).
- **Content**: Same package `e2e`, same imports as existing e2e tests, and one or more `Describe`/`Context`/`It` blocks. Each block should be clearly commented so the user can copy only the tests they need into `e2e_test.go` or keep as a separate `_test.go` file in the same package.
