# oape-ai-e2e

AI-driven Feature Development tools.

## Installation

Add the marketplace:
```shell
/plugin marketplace add chiragkyal/oape-ai-e2e
```

Install the plugin:
```shell
/plugin install oape@oape-ai-e2e
```

Use the commands:
```shell
/oape:api-generate https://github.com/openshift/enhancements/pull/1234
```

## Updating Plugins

Update the marketplace (fetches latest plugin catalog):
```shell
/plugin marketplace update oape-ai-e2e
```

Reinstall the plugin (downloads new version):
```shell
/plugin install oape@oape-ai-e2e
```

## Using Cursor

Cursor can discover the commands by symlinking this repo into your `~/.cursor/commands` directory:

```bash
mkdir -p ~/.cursor/commands
git clone git@github.com:chiragkyal/oape-ai-e2e.git
ln -s oape-ai-e2e ~/.cursor/commands/oape-ai-e2e
```

## Available Plugins

| Plugin | Description | Commands |
| ------------------------- | ---------------------------------------------- | --------------------------------------------------------------------------- |
| **[oape](plugins/oape/)** | AI-driven OpenShift operator development tools (includes ZTWIM test generator) | `/oape:api-generate`, `/oape:api-generate-tests`, `/oape:api-implement`, `/oape:ztwim-generate-all`, `/oape:ztwim-generate-from-pr`, `/oape:ztwim-generate-execution-steps`, `/oape:ztwim-generate-e2e-from-pr` |

## Commands

### `/oape:api-generate` -- Generate API Types from Enhancement Proposal

Reads an OpenShift enhancement proposal PR, extracts the required API changes, and generates compliant Go type definitions in the correct paths of the current OpenShift operator repository.

```shell
/oape:api-generate https://github.com/openshift/enhancements/pull/1234
```

### `/oape:api-generate-tests` -- Generate Integration Tests for API Types

Generates `.testsuite.yaml` integration test files for OpenShift API type definitions, covering create, update, validation, and error scenarios.

```shell
/oape:api-generate-tests api/v1alpha1/myresource_types.go
```

### `/oape:api-implement` -- Generate Controller Implementation from Enhancement Proposal

Reads an OpenShift enhancement proposal PR, extracts the required implementation logic, and generates complete controller/reconciler code following controller-runtime and operator-sdk conventions.

```shell
/oape:api-implement https://github.com/openshift/enhancements/pull/1234
```

**Typical workflow:**
```shell
# Step 1: Generate API types
/oape:api-generate https://github.com/openshift/enhancements/pull/1234

# Step 2: Generate integration tests
/oape:api-generate-tests api/v1alpha1/

# Step 3: Generate controller implementation
/oape:api-implement https://github.com/openshift/enhancements/pull/1234
```

### ZTWIM Test Generator (inside oape)

Generates test scenarios, step-by-step execution with `oc` commands, and e2e Go code for [openshift/zero-trust-workload-identity-manager](https://github.com/openshift/zero-trust-workload-identity-manager) PRs. See [plugins/oape/ztwim-test-generator/README.md](plugins/oape/ztwim-test-generator/README.md) for full docs.

**Single command (all artifacts):**

```shell
/oape:ztwim-generate-all https://github.com/openshift/zero-trust-workload-identity-manager/pull/92
```

Writes `test-cases.md`, `execution-steps.md`, `<prno>_test_e2e.go`, and `e2e-suggestions.md` into `output/ztwim_pr_<number>/`.

**Individual commands:** `/oape:ztwim-generate-from-pr`, `/oape:ztwim-generate-execution-steps`, `/oape:ztwim-generate-e2e-from-pr` (each with a PR URL).

### Adding a New Command

1. Add a new markdown file under `plugins/oape/commands/`
2. The command will be available as `/oape:<command-name>`
3. Update the plugin `README.md` documenting the new command

### Plugin Structure

```text
plugins/oape/
├── ztwim-test-generator/   # ZTWIM fixtures, docs, skills (commands are in commands/)
├── .claude-plugin/
│   └── plugin.json           # Required: plugin metadata
├── commands/
│   └── <command-name>.md     # Slash commands
├── skills/
│   └── <skill-name>/
│       └── SKILL.md          # Reusable agent skills (optional)
└── README.md                 # Plugin documentation
```
