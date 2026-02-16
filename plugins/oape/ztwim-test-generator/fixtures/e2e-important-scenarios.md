# Important E2E Scenarios (Operator and Operands)

Use this list when suggesting or generating e2e tests for ZTWIM PRs. These scenarios should be **highly checked** for operator and operand health.

## Operator

| # | Scenario | What to verify |
|---|----------|----------------|
| 1 | Operator install | All managed CRDs Established; operator Deployment Available in `zero-trust-workload-identity-manager` namespace. |
| 2 | Operator recovery | After force-deleting operator pod(s), new pod(s) Running and Deployment Available again. |
| 3 | Operator log level | Subscription env (e.g. OPERATOR_LOG_LEVEL) applied; Deployment gets rolling update and new env; verify in Deployment. |

## ZeroTrustWorkloadIdentityManager (manager CR)

| # | Scenario | What to verify |
|---|----------|----------------|
| 4 | ZTWIM created | Create ZeroTrustWorkloadIdentityManager with trust domain, cluster name, bundle ConfigMap; no error. |
| 5 | Operand aggregation | ZTWIM status.operands has 4 entries (SpireServer, SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider); each Ready and message ReasonReady. |

## Operands (SpireServer, SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider)

| # | Scenario | What to verify |
|---|----------|----------------|
| 6 | SpireServer install | Create SpireServer CR; conditions (e.g. StatefulSetAvailable, Ready) True; StatefulSet Ready. |
| 7 | SpireAgent install | Create SpireAgent CR; conditions True; DaemonSet Available. |
| 8 | SpiffeCSIDriver install | Create SpiffeCSIDriver CR; conditions True; DaemonSet Available. |
| 9 | SpireOIDCDiscoveryProvider install | Create SpireOIDCDiscoveryProvider CR; conditions True; Deployment Available. |

## OperatorCondition (Upgradeable)

| # | Scenario | What to verify |
|---|----------|----------------|
| 10 | Upgradeable when healthy | OperatorCondition Upgradeable status True, reason Ready. |
| 11 | Upgradeable False and recovery | Delete SPIRE Server (or other operand) pod; Upgradeable becomes False; after operand recovers, Upgradeable returns to True. |
| 12 | Multiple pod failures | Delete multiple operand pods (e.g. Agent + CSI); Upgradeable False; after recovery, True. |

## CR-driven configuration (operands)

| # | Scenario | What to verify |
|---|----------|----------------|
| 13 | SpireServer resources | Patch SpireServer with resources (limits/requests); StatefulSet rolling update; pods have expected resources. |
| 14 | SpireServer nodeSelector/tolerations | Patch SpireServer with nodeSelector and tolerations; pods scheduled on expected nodes; tolerations applied. |
| 15 | SpireServer affinity | Patch SpireServer with affinity (e.g. PodAntiAffinity); rolling update; pod rescheduled as expected. |
| 16 | SpireServer log level | Patch SpireServer log level; ConfigMap and/or StatefulSet updated; verify in ConfigMap or pod config. |

(Similar patterns apply for SpireAgent, SpiffeCSIDriver, SpireOIDCDiscoveryProvider when the PR touches those CRs.)

## PR-specific

- **API/CRD changes**: New or changed fields → add or adjust It that creates/updates CR and checks status or conditions.
- **Controller changes**: Reconciliation or condition logic → add It for recovery, Upgradeable, or operand lifecycle.
- **Validation**: CRD validation or webhook → add negative It (invalid CR, expect error or condition).

When generating `<prno>_test_e2e.go`, include at least the scenarios above that are relevant to the PR's Files changed, and add PR-specific It blocks as needed.
