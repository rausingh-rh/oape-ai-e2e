---
description: Generate OpenShift API type definitions from an enhancement proposal PR, following OpenShift and Kubernetes API conventions
argument-hint: <enhancement-pr-url>
---

## Name
oape:api-generate

## Synopsis
```shell
/oape:api-generate <https://github.com/openshift/enhancements/pull/NNNN>
```

## Description
The `oape:api-generate` command reads an OpenShift enhancement proposal PR, extracts the required API changes, and generates compliant Go type definitions in the correct paths of the current OpenShift operator repository.

It refreshes its knowledge of API conventions from the authoritative sources on every run, analyzes the enhancement proposal, and generates or modifies Go types that strictly follow both OpenShift and Kubernetes API conventions.

**You MUST follow ALL conventions strictly. If any precheck fails, you MUST stop immediately and report the failure.**

## Implementation

### Phase 0: Prechecks

All prechecks must pass before proceeding. If ANY precheck fails, STOP immediately and report the failure.

#### Precheck 1 — Validate Enhancement PR URL

The provided argument MUST be a valid GitHub PR URL pointing to the `openshift/enhancements` repository.

```bash
ENHANCEMENT_PR="$ARGUMENTS"

# Validate URL format
if [ -z "$ENHANCEMENT_PR" ]; then
  echo "PRECHECK FAILED: No enhancement PR URL provided."
  echo "Usage: /oape:api-generate <https://github.com/openshift/enhancements/pull/NNNN>"
  exit 1
fi

if ! echo "$ENHANCEMENT_PR" | grep -qE '^https://github\.com/openshift/enhancements/pull/[0-9]+/?$'; then
  echo "PRECHECK FAILED: Invalid enhancement PR URL."
  echo "Expected format: https://github.com/openshift/enhancements/pull/<number>"
  echo "Got: $ENHANCEMENT_PR"
  exit 1
fi

ENHANCEMENT_PR_NUMBER=$(echo "$ENHANCEMENT_PR" | grep -oE '[0-9]+$')
echo "Enhancement PR #$ENHANCEMENT_PR_NUMBER validated."
```

#### Precheck 2 — Verify Required Tools

```bash
MISSING_TOOLS=""

# Check gh CLI
if ! command -v gh &> /dev/null; then
  MISSING_TOOLS="$MISSING_TOOLS gh(GitHub CLI)"
fi

# Check Go
if ! command -v go &> /dev/null; then
  MISSING_TOOLS="$MISSING_TOOLS go"
fi

# Check git
if ! command -v git &> /dev/null; then
  MISSING_TOOLS="$MISSING_TOOLS git"
fi

if [ -n "$MISSING_TOOLS" ]; then
  echo "PRECHECK FAILED: Missing required tools:$MISSING_TOOLS"
  echo "Please install the missing tools and try again."
  exit 1
fi

# Check gh auth status
if ! gh auth status &> /dev/null 2>&1; then
  echo "PRECHECK FAILED: GitHub CLI is not authenticated."
  echo "Run 'gh auth login' to authenticate."
  exit 1
fi

echo "All required tools are available and authenticated."
```

#### Precheck 3 — Verify Current Repository is a Valid OpenShift Operator Repo

```bash
# Must be in a git repository
if ! git rev-parse --is-inside-work-tree &> /dev/null 2>&1; then
  echo "PRECHECK FAILED: Not inside a git repository."
  echo "This command must be run from within an OpenShift operator repository."
  exit 1
fi

REPO_ROOT=$(git rev-parse --show-toplevel)
echo "Repository root: $REPO_ROOT"

# Must have a go.mod file
if [ ! -f "$REPO_ROOT/go.mod" ]; then
  echo "PRECHECK FAILED: No go.mod found at repository root."
  echo "This command must be run from within a Go-based OpenShift operator repository."
  exit 1
fi

# Identify the Go module name
GO_MODULE=$(head -1 "$REPO_ROOT/go.mod" | awk '{print $2}')
echo "Go module: $GO_MODULE"

# Check if this repo vendors or imports openshift/api
if grep -q "github.com/openshift/api" "$REPO_ROOT/go.mod"; then
  echo "Confirmed: Repository depends on github.com/openshift/api"
elif echo "$GO_MODULE" | grep -q "github.com/openshift/api"; then
  echo "Confirmed: This IS the openshift/api repository."
else
  echo "PRECHECK FAILED: This repository does not appear to be an OpenShift operator repository."
  echo "go.mod does not reference github.com/openshift/api."
  echo "Module: $GO_MODULE"
  exit 1
fi
```

#### Precheck 4 — Verify Enhancement PR is Accessible

```bash
echo "Fetching enhancement PR #$ENHANCEMENT_PR_NUMBER details..."

PR_STATE=$(gh pr view "$ENHANCEMENT_PR_NUMBER" --repo openshift/enhancements --json state --jq '.state' 2>/dev/null)

if [ -z "$PR_STATE" ]; then
  echo "PRECHECK FAILED: Unable to access enhancement PR #$ENHANCEMENT_PR_NUMBER."
  echo "Ensure the PR exists and you have access to the openshift/enhancements repository."
  exit 1
fi

echo "Enhancement PR #$ENHANCEMENT_PR_NUMBER state: $PR_STATE"

# Get the PR title and body for context
PR_TITLE=$(gh pr view "$ENHANCEMENT_PR_NUMBER" --repo openshift/enhancements --json title --jq '.title')
echo "Enhancement title: $PR_TITLE"
```

#### Precheck 5 — Verify Clean Working Tree (Warning)

```bash
if ! git diff --quiet || ! git diff --cached --quiet; then
  echo "WARNING: Uncommitted changes detected in the working tree."
  echo "It is recommended to commit or stash changes before generating API types."
  echo "Proceeding anyway..."
  git status --short
else
  echo "Working tree is clean."
fi
```

**If ALL prechecks above passed, proceed to Phase 1.**
**If ANY precheck FAILED (exit 1), STOP. Do NOT proceed further. Report the failure to the user.**

---

### Phase 1: Refresh Knowledge — Fetch Latest API Conventions

You MUST fetch and read both of these documents in full BEFORE analyzing the enhancement proposal
or generating any code. Never rely on cached knowledge — the freshly fetched versions are the
single source of truth.

1. **OpenShift API Conventions**: `https://raw.githubusercontent.com/openshift/enhancements/master/dev-guide/api-conventions.md`
2. **Kubernetes API Conventions**: `https://raw.githubusercontent.com/kubernetes/community/master/contributors/devel/sig-architecture/api-conventions.md`

```thinking
I must now read both fetched convention documents in full and extract every rule that applies to
API type generation — field markers, naming, documentation, validation, pointers, unions, enums,
TechPreview gating, etc. I will NOT rely on any pre-built checklist; the fetched documents are the
single source of truth. If the conventions have been updated since this command was written, the
freshly fetched versions take precedence. I will carry all extracted rules forward into the code
generation steps.
```

### Phase 2: Fetch and Analyze the Enhancement Proposal

Read all changed/added files in the enhancement PR to find the proposal document:

```bash
# Get the list of files changed in the PR
echo "Fetching files changed in enhancement PR #$ENHANCEMENT_PR_NUMBER..."
gh pr view "$ENHANCEMENT_PR_NUMBER" --repo openshift/enhancements --json files --jq '.files[].path'
```

```thinking
I need to:
1. Identify the enhancement proposal markdown file(s) — typically under enhancements/<area>/<proposal-name>.md
2. Read the full content of the proposal
3. Extract the following critical information:
   a. Which OpenShift operator/component is being modified
   b. The API group and version (e.g., config.openshift.io/v1, operator.openshift.io/v1)
   c. Whether this is a NEW CRD or modifications to an EXISTING CRD
   d. Whether this is a Configuration API or Workload API
   e. The specific fields/types being added or modified
   f. Validation requirements (enums, patterns, min/max, cross-field)
   g. Whether fields should be TechPreview-gated
   h. Any discriminated unions
   i. Defaulting behavior
   j. Immutability requirements
   k. Status fields and conditions
   l. The FeatureGate name to use
```

Fetch the full content of each proposal file. Use the PR ref (`refs/pull/<number>/head`) which
GitHub always maintains, even if the fork branch has been deleted:

```bash
# For each enhancement .md file found in the file list above, fetch its full content:
gh api "repos/openshift/enhancements/contents/<path-to-file>?ref=refs/pull/$ENHANCEMENT_PR_NUMBER/head" --jq '.content' | base64 -d
```

If the above fails, try fetching the raw file via curl:

```bash
curl -sL "https://raw.githubusercontent.com/openshift/enhancements/refs/pull/$ENHANCEMENT_PR_NUMBER/head/<path-to-file>"
```

As a last resort, fall back to reading the diff which contains the full proposed content:

```bash
gh pr diff "$ENHANCEMENT_PR_NUMBER" --repo openshift/enhancements
```

### Phase 3: Identify Target API Paths in Current Repository

Different OpenShift repositories organize API types differently. Explore the current repo to
determine which layout pattern is in use, then map the enhancement proposal's API changes to
the correct file paths.

#### Known Layout Patterns

**Pattern 1 — openshift/api repository:**
```text
<group>/<version>/types_<resource>.go
<group>/<version>/doc.go
<group>/<version>/register.go
<group>/<version>/tests/<crd-name>/*.testsuite.yaml
features/features.go
```
- File naming: `types_<resource>.go`
- Registration: `doc.go` + `register.go`
- FeatureGates: `features/features.go`

**Pattern 2 — Operator repo with group subdirectory (e.g., cert-manager-operator):**
```text
api/<group>/<version>/<resource>_types.go
api/<group>/<version>/groupversion_info.go
api/<group>/<version>/doc.go
api/<group>/<version>/zz_generated.deepcopy.go
```
- File naming: `<resource>_types.go`
- Registration: `groupversion_info.go` with `SchemeBuilder`
- Each types file has `init()` calling `SchemeBuilder.Register()`

**Pattern 3 — Operator repo with flat version directory (e.g., external-secrets-operator):**
```text
api/<version>/<resource>_types.go
api/<version>/groupversion_info.go
api/<version>/tests/<resource>/*.testsuite.yaml
api/<version>/zz_generated.deepcopy.go
```
- File naming: `<resource>_types.go`
- Registration: `groupversion_info.go` with `SchemeBuilder`

#### Detect the Pattern

Run these commands to identify which layout the current repo uses:

```bash
# Find type definition files
find "$REPO_ROOT" -type f \( -name 'types*.go' -o -name '*_types.go' \) -not -path '*/vendor/*' -not -path '*/_output/*' -not -path '*/zz_generated*' | head -40

# Find registration files
find "$REPO_ROOT" -type f \( -name 'doc.go' -o -name 'register.go' -o -name 'groupversion_info.go' \) -not -path '*/vendor/*' -not -path '*/_output/*' | head -40

# Find CRD manifests
find "$REPO_ROOT" -type f -name '*.crd.yaml' -not -path '*/vendor/*' | head -20

# Find test suites
find "$REPO_ROOT" -type f -name '*.testsuite.yaml' -not -path '*/vendor/*' | head -20

# Find feature gate definitions
find "$REPO_ROOT" -type f -name 'features.go' -not -path '*/vendor/*' | head -10
```

### Phase 4: Read Existing API Types for Context

Before generating new code, read the existing types in the target API package to understand:
- The existing struct layout and naming patterns
- Import conventions used
- Existing markers and annotations
- How other fields in the same struct are documented

```thinking
I must read the existing types file(s) in the target package to:
1. Match the coding style exactly
2. Understand existing struct hierarchy
3. Know where to insert new fields or add new types
4. Identify existing fields/types that need to be modified (e.g., adding new enum values,
   updating validation rules, changing godoc, adding new fields to existing structs)
5. Identify existing imports that may be reused
6. See how feature gates are applied to existing fields
7. Understand the existing validation patterns
```

### Phase 5: Generate or Modify API Type Definitions

Generate or modify Go type definitions based on the enhancement proposal. This may include new
types, new fields, modifications to existing fields, enum types, discriminated unions, or type
registration.

For every marker, tag, or convention applied: derive it from the fetched convention documents
(Phase 1) or the existing code (Phase 4). Conventions take precedence when both differ. Existing
patterns not covered by conventions (e.g., mechanical code-gen markers) should be replicated for
consistency.

Determine from the enhancement proposal whether this is a **Configuration API** or **Workload API**,
as the conventions define different rules for each.

After generating, review every changed line against the conventions. If any violation has a
convention-compliant alternative, apply it and note the deviation in the Phase 7 summary.

### Phase 6: Add FeatureGate Registration (if applicable)

If the repository contains a `features.go` file (found in Phase 3), read it to learn the existing
FeatureGate registration pattern, then add a new FeatureGate following that pattern.

If no `features.go` exists and the enhancement requires a FeatureGate, note this in the summary
and advise the user on where to register it.

### Phase 7: Output Summary

After generating all files, provide a summary:

```text
=== API Generation Summary ===

Enhancement PR: <url>
Enhancement Title: <title>

Generated/Modified Files:
  - <path/to/types_resource.go> — <description of changes>
  - <path/to/features/features.go> — <FeatureGate added> (if applicable)

API Group: <group.openshift.io>
API Version: <version>
Kind: <KindName>
Resource: <resourcename>
Scope: <Cluster|Namespaced>
FeatureGate: <FeatureGateName>

New Types Added:
  - <TypeName> — <description>

New Fields Added:
  - <ParentType>.<fieldName> (<type>) — <description>

Modified Fields/Types:
  - <ParentType>.<fieldName> — <what changed and why>

Validation Rules:
  - <field>: <rule description>

Next Steps:
  1. Review the generated code for correctness
  2. Run 'make update' to regenerate CRDs and deep copy functions
  3. Run 'make verify' to validate all generated code
  4. Run 'make lint' to check for kube-api-linter issues
  5. If FeatureGate was added, verify it appears in the feature gate list
```

---

## Critical Failure Conditions

The command MUST FAIL and STOP immediately if ANY of the following are true:

1. **Invalid PR URL**: The provided URL is not a valid `openshift/enhancements` PR
2. **Missing tools**: `gh`, `go`, or `git` are not installed or `gh` is not authenticated
3. **Not an operator repo**: The current directory is not a Git repository with a Go module that references `openshift/api`
4. **PR not accessible**: The enhancement PR cannot be fetched (permissions, doesn't exist, etc.)
5. **No API changes found**: The enhancement proposal does not describe any API changes
6. **Ambiguous API target**: Cannot determine the target API group, version, or kind from the proposal.

When failing, provide a clear error message explaining:
- Which precheck failed
- What the expected state is
- How to fix the issue

## Behavioral Rules

1. **Never guess**: If the enhancement proposal is ambiguous about API details, STOP and ask the user for clarification rather than guessing.
2. **Convention over proposal**: If the enhancement proposal suggests an API design that violates conventions (e.g., using a Boolean), generate the convention-compliant alternative and document the deviation.
3. **TechPreview when specified**: If the enhancement proposal indicates TechPreview gating, generate the appropriate FeatureGate markers. Follow whatever the proposal specifies regarding API maturity level.
4. **Idempotent**: Running this command multiple times with the same PR should produce the same result (though it should warn if files already exist).
5. **Minimal changes**: Only generate what the enhancement proposes. Do not add extra fields, types, or features not described in the proposal.
6. **Surgical edits**: When modifying existing files, only change what the enhancement proposal requires. Preserve all unrelated code, comments, and formatting. For modifications to existing fields, clearly document what changed and why in the output summary.

## Arguments

- `<enhancement-pr-url>`: GitHub PR URL to the OpenShift enhancement proposal
  - Format: `https://github.com/openshift/enhancements/pull/<number>`
  - Required argument

## Prerequisites

- **gh** (GitHub CLI) — installed and authenticated (`gh auth login`)
- **go** — Go toolchain installed
- **git** — Git installed
- Must be run from within an OpenShift operator repository (Go module that references `github.com/openshift/api`)

## Exit Conditions

- **Success**: API type definitions generated/modified with a summary of all changes
- **Failure Scenarios**:
  - Invalid or missing enhancement PR URL
  - Missing required tools or unauthenticated GitHub CLI
  - Not inside a valid OpenShift operator repository
  - Enhancement PR inaccessible
  - No API changes found in the proposal
  - Ambiguous API target (asks for clarification instead of guessing)
