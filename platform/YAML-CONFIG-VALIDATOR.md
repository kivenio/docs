# YCC — YAML Config Validator

> **Priority:** Low
> **Repo:** `yaml-config-validator`
> **Language:** Python (Pydantic v2)
> **Status:** Planned

## Problem

The Kiven platform relies on YAML files across multiple repos:
- **`platform-github-management`** — Repo definitions (`repos/backend/*.yaml`, `config/enforced.yaml`, `config/defaults.yaml`)
- **Copier templates** — `copier.yml` with questions, validators, conditional logic
- **Service configs** — `.mise.toml`, `Taskfile.yml`, CI workflows reference variables

These YAMLs are validated only at runtime (sync script fails, Copier fails, CI fails).
There is no **static validation** before merge. A typo in `template: servce-go` passes review
and breaks provisioning.

## Solution: YCC (YAML Config Checker)

A Python CLI tool that validates YAML structures **statically** (schema) and **dynamically**
(cross-references, variable resolution) using Pydantic v2.

## What It Validates

### Static Validation (schema)

| Source | Validates |
|--------|-----------|
| `repos/backend/*.yaml` | Required fields (`name`, `description`, `type`), valid types, valid rulesets |
| `repos/platform/*.yaml` | Template references exist in `config/templates.yaml` |
| `config/enforced.yaml` | Schema matches expected structure |
| `config/defaults.yaml` | Rulesets have required fields, label colors are valid hex |
| `copier.yml` | Question types are valid, validators use valid Jinja2 syntax |
| `renovate.json` | Matches Renovate JSON schema |

### Dynamic Validation (cross-references)

| Check | Description |
|-------|-------------|
| Template exists | If `template: service-go` is set, `config/templates.yaml` must have a `service-go` entry |
| No orphan repos | Every repo in `repos/` must have a corresponding template (or `# No template` comment) |
| Copier variables used | Variables defined in `copier.yml` must be referenced in at least one `.jinja` file |
| Copier variables resolved | `.jinja` files must not reference undefined variables |
| Workflow references valid | CI workflows referencing `kivenio/reusable-workflows` must point to existing workflow files |
| Ruleset exists | `ruleset: strict` must exist in `config/defaults.yaml` rulesets |

## Architecture

```
yaml-config-validator/
├── ycc/
│   ├── __init__.py
│   ├── cli.py                    # Click CLI entrypoint
│   ├── schemas/
│   │   ├── repo_definition.py    # Pydantic model for repos/*.yaml
│   │   ├── enforced_config.py    # Pydantic model for enforced.yaml
│   │   ├── defaults_config.py    # Pydantic model for defaults.yaml
│   │   ├── templates_config.py   # Pydantic model for templates.yaml
│   │   └── copier_config.py     # Pydantic model for copier.yml
│   ├── validators/
│   │   ├── static.py             # Schema-only validation
│   │   ├── cross_ref.py          # Cross-reference validation
│   │   └── copier_vars.py        # Copier variable resolution
│   └── reporters/
│       ├── console.py            # Pretty terminal output
│       └── github.py             # PR comment with validation results
├── tests/
│   ├── test_schemas.py
│   ├── test_cross_ref.py
│   └── fixtures/                 # Sample valid/invalid YAMLs
├── pyproject.toml
└── README.md
```

## Usage (planned)

```bash
# Validate platform-github-management
ycc validate ../platform-github-management/

# Validate a Copier template
ycc validate-template ../platform-templates-service-go/

# Validate a specific file
ycc validate-file repos/backend/core-services.yaml

# CI mode (exit code 1 on failure, PR comment)
ycc validate --ci --github-pr 42
```

## CI Integration (planned)

A reusable workflow in `reusable-workflows` will call YCC on PRs to `platform-github-management`:

```yaml
# In platform-github-management/.github/workflows/validate.yml
jobs:
  validate:
    uses: kivenio/reusable-workflows/.github/workflows/ycc-validate-reusable.yml@main
    with:
      config-path: "."
```

## Pydantic Model Example

```python
from pydantic import BaseModel, field_validator

class RepoDefinition(BaseModel):
    name: str
    description: str
    type: str  # service, library, platform, template, testing, documentation
    template: str | None = None
    ruleset: str | None = None
    topics: list[str] = []
    visibility: str = "private"

    @field_validator("type")
    @classmethod
    def validate_type(cls, v: str) -> str:
        valid = {"service", "library", "platform", "template", "testing", "documentation", "bootstrap", "sdk"}
        if v not in valid:
            raise ValueError(f"Invalid type '{v}', must be one of {valid}")
        return v

    @field_validator("name")
    @classmethod
    def validate_name(cls, v: str) -> str:
        if not v.replace("-", "").isalnum():
            raise ValueError(f"Name '{v}' must be alphanumeric with hyphens")
        return v
```

## Why Pydantic v2

- **Type safety** — Python type hints map directly to YAML schema
- **Custom validators** — `@field_validator` for cross-reference checks
- **Error messages** — Clear, structured errors with field paths
- **JSON Schema export** — `model_json_schema()` generates JSON Schema for IDE autocomplete
- **Fast** — Pydantic v2 (Rust core) validates thousands of files in milliseconds
