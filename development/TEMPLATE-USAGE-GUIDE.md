# Template & Workflow Architecture

## Overview

Kiven uses a **three-layer developer platform** to ensure every repo starts with the right tooling, CI/CD, and conventions — without the developer thinking about any of it.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1: COPIER TEMPLATES                                      │
│  platform-templates-service-go, platform-templates-sdk-go, ...  │
│  ─ Scaffold a new repo with all files, config, CI/CD            │
│  ─ copier copy gh:kivenio/platform-templates-service-go ./repo  │
│  ─ copier update (evolve existing repos when template changes)  │
└───────────────────────┬─────────────────────────────────────────┘
                        │ references
┌───────────────────────▼─────────────────────────────────────────┐
│  Layer 2: PLATFORM-GITHUB-MANAGEMENT                            │
│  Declarative YAML → GitHub repos, settings, rulesets, labels    │
│  ─ repos/backend/core-services.yaml defines all svc-*           │
│  ─ template: service-go links to the Copier template            │
│  ─ config/enforced.yaml locks security settings                 │
│  ─ sync-repos.py applies changes on PR merge                    │
└───────────────────────┬─────────────────────────────────────────┘
                        │ includes
┌───────────────────────▼─────────────────────────────────────────┐
│  Layer 3: REUSABLE WORKFLOWS + COMPOSITE ACTIONS                │
│  Repo: reusable-workflows 
│  reusable-workflows/.github/workflows/ci-go-reusable.yml       │
│  reusable-workflows/.github/actions/setup-go/action.yml        │
│  ─ Called by each repo's CI workflow (generated from template)   │
│  ─ One source of truth for all CI/CD logic                      │
└─────────────────────────────────────────────────────────────────┘
```

## How It All Connects

### Creating a new repo

```
Developer                  platform-github-management         Copier template
    │                              │                               │
    │  1. Add YAML entry           │                               │
    │  in repos/backend/*.yaml     │                               │
    │  with template: service-go   │                               │
    │─────────────────────────────►│                               │
    │                              │                               │
    │  2. PR merged → sync-repos   │                               │
    │  creates GitHub repo         │                               │
    │  with settings, labels,      │                               │
    │  rulesets, topics, teams      │                               │
    │                              │                               │
    │  3. Developer clones repo    │                               │
    │  and runs copier copy        │                               │
    │◄─────────────────────────────│                               │
    │                              │                               │
    │  4. copier copy              │                               │
    │  gh:kivenio/platform-        │  answers copier.yml questions │
    │  templates-service-go .      │──────────────────────────────►│
    │                              │                               │
    │  5. Template generates       │  .editorconfig, .golangci.yml │
    │  ALL files:                  │  .mise.toml, Taskfile.yml     │
    │  tooling, CI, Dockerfile,    │  .pre-commit-config.yaml      │
    │  vscode, go.mod, cmd/...     │  .github/workflows/ci.yml    │
    │◄─────────────────────────────│◄──────────────────────────────│
    │                              │                               │
    │  6. task init → ready        │                               │
```

### Updating existing repos when template evolves

```bash
cd svc-api
copier update 
# Copier shows diff, applies new changes, respects .copier-answers.yml
```

This is the **key advantage** of Copier over `sed`-based scaffolding: when you add a new
linter rule, a new shared GitHub Action, or update Go version across all services, you
update the template once and each repo pulls the update via `copier update`. =>>>> this should be done via github resuable action and the target repo (the repo created baed on the tempalte) will triggered.

## Template Inventory

| Template | Repo | For | Key Features |
|----------|------|-----|--------------|
| `service-go` | `platform-templates-service-go` | Go microservices (`svc-*`) | chi, gRPC, OTel, Dockerfile, air, Testcontainers |
| `sdk-go` | `platform-templates-sdk-go` | Go libraries (`kiven-go-sdk`, `provider-*`, `kiven-cli`) | No Dockerfile, no cmd/, library-focused |
| `infrastructure` | `platform-templates-infrastructure` | Terraform modules (`bootstrap`, `infra-customer-*`) | tflint, Checkov, terraform-docs |
| `platform-component` | `platform-templates-platform-component` | GitOps components (`platform-gitops`, `platform-security`) | Helm/Kustomize, Flux integration |
| `documentation` | `platform-templates-documentation` | Doc sites (`docs`) | MkDocs Material, ADR template |

## Template Structure (Copier)

Each `platform-templates-*` repo follows the Copier convention:

```
platform-templates-service-go/
├── copier.yml                           # Questions + config
├── .copier-answers.yml.jinja            # Records answers in generated project
├── {{project_name}}/                    # (not used — we generate at root)
│
├── .editorconfig                        # Static — copied as-is
├── .gitignore                           # Static
├── .golangci.yml                        # Static
├── .pre-commit-config.yaml              # Static
├── Dockerfile                           # Static
│
├── .mise.toml.jinja                     # Templated — injects service name
├── Taskfile.yml.jinja                   # Templated — injects service name, port
├── go.mod.jinja                         # Templated — injects module path
├── README.md.jinja                      # Templated — injects name, description
│
├── .vscode/
│   ├── settings.json                    # Static
│   ├── extensions.json                  # Static
│   └── launch.json.jinja               # Templated — injects port, env vars
│
├── renovate.json                        # Static — Renovate dep updates (auto-merge, grouping)
├── .github/
│   ├── CODEOWNERS.jinja                 # Templated — injects team
│   ├── pull_request_template.md         # Static — shared PR checklist
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.yml               # Static
│   │   └── feature_request.yml          # Static
│   └── workflows/
│       ├── ci.yml.jinja                 # Templated — injects service name
│       ├── release.yml.jinja            # Templated — injects service name
│       └── copier-update.yml            # Static — weekly template sync check
│
└── cmd/
    └── main.go.jinja                    # Templated — basic service scaffold
```

### `copier.yml` — The Questionnaire

```yaml
# copier.yml — Kiven Go Service Template
_min_copier_version: "9.0.0"
_subdirectory: ""
_answers_file: .copier-answers.yml

project_name:
  type: str
  help: "Service name (e.g., svc-api, svc-auth)"
  validator: "{% if not project_name | regex_search('^[a-z][a-z0-9-]+$') %}Must be lowercase with hyphens{% endif %}"

project_description:
  type: str
  help: "One-line description"

owner_team:
  type: str
  default: "@kivenio/backend"
  help: "GitHub team that owns this repo"

port:
  type: int
  default: 8080
  help: "HTTP port"

grpc_port:
  type: int
  default: 0
  help: "gRPC port (0 = no gRPC)"

enable_kafka:
  type: bool
  default: false
  help: "Does this service consume/produce Kafka events?"

go_version:
  type: str
  default: "1.23"
  help: "Go version"
```

### How variables connect to `platform-github-management`

In `repos/backend/core-services.yaml`, each repo definition contains a `template_variables`
field with ALL the Copier variables needed for that repo:

```yaml
- name: svc-api                    # → auto-mapped to project_name
  description: "API Gateway"       # → auto-mapped to project_description
  type: service
  template: service-go             # → which Copier template to use
  ruleset: strict
  topics: [core, api, graphql]
  template_variables:              # → ALL Copier variables for this repo
    port: 8080
    grpc_port: 0
    enable_kafka: false
```

**Variable resolution order:**

1. **Auto-mapped** (always set, derived from repo definition fields):
   - `name` → `project_name`
   - `description` → `project_description`
   - Owner team from YAML file path (`repos/backend/` → `@kivenio/backend`) → `owner_team`

2. **Explicit** (`template_variables` dict — overrides auto-mapped if same key):
   - `port`, `grpc_port`, `enable_kafka`, or any Copier variable from `copier.yml`

3. **Template defaults** (from `copier.yml` — used for any variable not specified above):
   - `go_version: "1.23"`, etc.

When `sync-repos.py` creates a repo, it passes all variables to `copier copy` via `-d key=value`.
Copier generates `.copier-answers.yml` in the new repo, recording every variable used.
This file is essential for future `copier update` runs — it tells Copier which template
was used and with which answers.

```yaml
# .copier-answers.yml (auto-generated by Copier in the new repo)
_src_path: gh:kivenio/platform-templates-service-go
_commit: abc1234
project_name: svc-api
project_description: "API Gateway — REST + GraphQL, request routing"
owner_team: "@kivenio/backend"
port: 8080
grpc_port: 0
enable_kafka: false
go_version: "1.23"
```

## Reusable Workflows vs Composite Actions

### Reusable Workflows (job-level reuse)

A reusable workflow replaces an **entire job** (or set of jobs). Each service repo calls them
in its `.github/workflows/ci.yml`:

```yaml
# In svc-api/.github/workflows/ci.yml
jobs:
  ci:
    uses: kivenio/reusable-workflows/.github/workflows/ci-go-reusable.yml@main
    with:
      service-name: "svc-api"
      go-version: "1.23"
    secrets: inherit
```

| Workflow | File | Purpose |
|----------|------|---------|
| `ci-go-reusable.yml` | `reusable-workflows/.github/workflows/` | Lint + Test + Build + Security + Docker |
| `ci-frontend-reusable.yml` | `reusable-workflows/.github/workflows/` | Lint + TypeCheck + Test + Build + Audit |
| `ci-terraform-reusable.yml` | `reusable-workflows/.github/workflows/` | Format + Validate + tflint + Trivy + Checkov |
| `docker-build-reusable.yml` | `reusable-workflows/.github/workflows/` | Buildx + GHCR push + semver tags |
| `release-reusable.yml` | `reusable-workflows/.github/workflows/` | Conventional commits → semver → changelog → GitHub Release |
| `copier-update-reusable.yml` | `reusable-workflows/.github/workflows/` | Detect template drift → auto-PR with updates |
| `ci-copier-template-reusable.yml` | `reusable-workflows/.github/workflows/` | Validate Copier templates (syntax, dry-run, build test) |

**When to use:** When you want to share a complete CI/CD pipeline across repos.

### Composite Actions (step-level reuse)

A composite action replaces **individual steps** within a job. Use them when multiple
workflows share the same setup/teardown logic but differ in the middle.

```yaml
# In any workflow
steps:
  - uses: kivenio/reusable-workflows/.github/actions/setup-go@main
    with:
      go-version: "1.23"
  # setup-go handles: checkout + install Go + cache + download deps
```

| Action | File | Purpose |
|--------|------|---------|
| `setup-go` | `reusable-workflows/.github/actions/setup-go/action.yml` | Checkout + Go install + cache + `go mod download` |
| `setup-node` | `reusable-workflows/.github/actions/setup-node/action.yml` | Checkout + Node install + cache + `npm ci` |
| `security-scan` | `reusable-workflows/.github/actions/security-scan/action.yml` | Trivy fs + Gitleaks in one step |
| `docker-metadata` | `reusable-workflows/.github/actions/docker-metadata/action.yml` | Generate tags (sha, branch, semver) |
| `copier-update` | `reusable-workflows/.github/actions/copier-update/action.yml` | Run `copier update`, detect drift, create PR |

**When to use:** When reusable workflows are too rigid. For example, a workflow that needs
custom steps between setup and test can use composite actions for the common parts.

### How They Compose

```
Service repo CI workflow
│
├── uses: kivenio/reusable-workflows/.github/workflows/ci-go-reusable.yml
│   │
│   ├── Job: lint
│   │   └── uses: kivenio/reusable-workflows/.github/actions/setup-go
│   │   └── golangci-lint
│   │
│   ├── Job: test
│   │   └── uses: kivenio/reusable-workflows/.github/actions/setup-go
│   │   └── go test
│   │
│   ├── Job: security
│   │   └── uses: kivenio/reusable-workflows/.github/actions/security-scan
│   │
│   └── Job: docker
│       └── uses: kivenio/reusable-workflows/.github/actions/docker-metadata
│       └── docker build + push
```

## What's Mutualized in Templates

Every repo created from a Copier template automatically gets:

### Tooling (developer experience)
- `.editorconfig` — Consistent formatting (tabs for Go, spaces for YAML)
- `.vscode/settings.json` — Formatter, linter, language settings
- `.vscode/extensions.json` — Recommended VS Code extensions
- `.vscode/launch.json` — Debug configurations
- `.mise.toml` — Tool versions (Go, Node, Terraform, etc.)
- `.pre-commit-config.yaml` — Git hooks (format, lint, secrets detection)

### Quality (code standards)
- `.golangci.yml` — 18 linters with opinionated config (Go repos)
- `.prettierrc` — Code formatting (frontend repos)
- `Taskfile.yml` — Standard tasks (init, test, lint, build, clean)

### CI/CD (automation)
- `.github/workflows/ci.yml` — Calls reusable workflow from kiven-dev
- `.github/workflows/release.yml` — Release automation
- `renovate.json` — Automated dependency updates (Renovate — grouping, auto-merge patches/minors)
- `Dockerfile` — Multi-stage, non-root, healthcheck (service repos)

### Collaboration (team workflow)
- `.github/CODEOWNERS` — Review routing (auto-assigned from `owner_team`)
- `.github/pull_request_template.md` — PR checklist (tests, docs, breaking changes)
- `.github/ISSUE_TEMPLATE/bug_report.yml` — Structured bug reports
- `.github/ISSUE_TEMPLATE/feature_request.yml` — Feature request form

### Project (bootstrapping)
- `go.mod` — Module initialized with `github.com/kivenio/<name>`
- `cmd/main.go` — Minimal service entrypoint
- `README.md` — Generated with name, description, badges
- `.copier-answers.yml` — Records template answers for future updates

## Applying Templates to Existing Repos

For repos that were created before Copier was set up:

```bash
cd ../svc-api
copier copy gh:kivenio/platform-templates-service-go . --overwrite
# Copier asks questions, generates files, respects existing code
```

For selective file application (tooling only, no code scaffolding), use
`copier copy --exclude 'cmd/**'` to skip code directories.

## Template Repo CI

Every `platform-templates-*` repo has its own CI (NOT a `.jinja` file — it runs on the template
repo itself) that validates the Copier template works correctly:

```yaml
# platform-templates-service-go/.github/workflows/ci.yml
jobs:
  ci:
    uses: kivenio/reusable-workflows/.github/workflows/ci-copier-template-reusable.yml@main
    with:
      template-type: "go"
```

**What it validates:**
1. `copier.yml` exists and is valid YAML
2. Jinja2 template files (`.jinja`) exist
3. Dry-run `copier copy --defaults` generates all critical files (editorconfig, gitignore, CI, CODEOWNERS, etc.)
4. No unresolved Jinja2 variables in generated output
5. Generated Go project compiles (`go build`, `go vet`)
6. Security scan (Trivy + Gitleaks) on the template itself

This means every PR to a template repo is validated end-to-end before merge.

## Automatic Scaffolding from Templates

When a new repo is defined in `platform-github-management` with a `template` field:

```yaml
- name: svc-foo
  description: "New service"
  template: service-go         # ← this triggers Copier scaffolding
```

The `sync-repos.py` script automatically:
1. Creates the GitHub repo (settings, labels, rulesets, teams)
2. Resolves the template source from `config/templates.yaml` (`service-go` → `gh:kivenio/platform-templates-service-go`)
3. Clones the new repo
4. Runs `copier copy --trust --defaults` with variables extracted from the YAML:
   - `name` → `project_name`
   - `description` → `project_description`
   - Owner team from the YAML file path (`repos/backend/` → `@kivenio/backend`)
5. Commits and pushes the scaffolded files

The developer gets a fully configured, ready-to-code repo without touching `copier` manually.

## Adding a New Template

1. Create the repo in `platform-github-management/repos/platform/templates.yaml`
2. Create the Copier template repo with `copier.yml` + files
3. Add a `ci.yml` that calls `ci-copier-template-reusable.yml` (validates the template)
4. Register it in `platform-github-management/config/templates.yaml`
5. Document it in this guide

## Evolving Templates

When you change a template (e.g., update Go version, add a linter):

1. Update the `platform-templates-*` repo
2. Tag a new version (e.g., `v1.2.0`)
3. Each service repo pulls the update **automatically via CI**:

### Automated Template Sync (CI)

Every repo created from a Copier template includes a `copier-update.yml` workflow:

```yaml
# .github/workflows/copier-update.yml (auto-generated from template)
on:
  schedule:
    - cron: "0 7 * * 1"    # Every Monday 7am
  workflow_dispatch:        # Can also trigger manually

jobs:
  copier-update:
    uses: kivenio/reusable-workflows/.github/workflows/copier-update-reusable.yml@main
    secrets: inherit
```

**How it works:**

```
Template repo changes          Downstream repo (svc-api)
       │                              │
       │  1. Developer updates        │
       │  platform-templates-         │
       │  service-go (new linter,     │
       │  Go version bump, etc.)      │
       │                              │
       │                              │  2. Monday 7am: copier-update
       │                              │  workflow runs automatically
       │                              │
       │                              │  3. Composite action:
       │                              │  - Installs copier
       │                              │  - Reads .copier-answers.yml
       │                              │  - Runs copier update --trust
       │                              │  - Detects git diff
       │                              │
       │                              │  4. If drift detected:
       │                              │  - Creates branch chore/copier-update
       │                              │  - Opens PR with changes
       │                              │  - Labels: dependencies
       │                              │
       │                              │  5. Developer reviews PR
       │                              │  - Resolves conflicts if any
       │                              │  - Merges when ready
```

### Manual Update

You can also update manually at any time:

```bash
cd svc-api
copier update --trust
# Shows diff of changes, applies non-conflicting updates
# Conflicts are shown for manual resolution
```

### Triggering Update Across All Repos

After a significant template change, trigger all downstream repos at once via
`workflow_dispatch` on each repo's `copier-update.yml`. This can be scripted:

```bash
repos=(svc-api svc-auth svc-provisioner svc-clusters svc-backups)
for repo in "${repos[@]}"; do
  gh workflow run copier-update.yml --repo kivenio/$repo
done
```

This is how the entire fleet stays consistent without manual copy-paste.

## YAML Config Validator (YCC)

For detailed documentation on the planned YAML validation tool that validates
`platform-github-management` structures and Copier template variables, see
[YAML-CONFIG-VALIDATOR.md](../platform/YAML-CONFIG-VALIDATOR.md).
