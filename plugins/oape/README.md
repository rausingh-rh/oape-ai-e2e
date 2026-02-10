# oape Plugin

AI-driven OpenShift operator development tools, following OpenShift and Kubernetes API conventions.

## Commands

### `/oape:api-generate`

Reads an OpenShift enhancement proposal PR, extracts the required API changes, and generates compliant Go type definitions in the correct paths of the current OpenShift operator repository.

**Usage:**
```shell
/oape:api-generate https://github.com/openshift/enhancements/pull/1234
```

**What it does:**
1. **Prechecks** -- Validates the PR URL, required tools (`gh`, `go`, `git`), GitHub authentication, repository type (must be an OpenShift operator repo with `openshift/api` dependency), and PR accessibility. Fails immediately if any precheck fails.
2. **Knowledge Refresh** -- Fetches and internalizes the latest OpenShift and Kubernetes API conventions before generating any code.
3. **Enhancement Analysis** -- Reads the enhancement proposal to extract API group, version, kinds, fields, validation requirements, feature gate info, and whether it is a configuration or workload API.
4. **Code Generation** -- Generates or modifies Go type definitions following conventions derived from the authoritative documents and patterns from the existing codebase.
5. **FeatureGate Registration** -- Adds FeatureGate to `features.go` when applicable.

## Prerequisites

- **gh** (GitHub CLI) -- installed and authenticated
- **go** -- Go toolchain
- **git** -- Git
- Must be run from within an OpenShift operator repository

## Conventions Enforced

- [OpenShift API Conventions](https://github.com/openshift/enhancements/blob/master/dev-guide/api-conventions.md)
- [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
