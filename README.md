# HendrikHarren/.github — Shared Reusable Workflows

Central repository for GitHub Actions reusable workflows used across all HendrikHarren repositories.

## Available Workflows

| Workflow | Description | Timeout |
|---|---|---|
| `reusable-tests.yml` | Python test suite with coverage | 15 min |
| `reusable-type-check.yml` | mypy type checking with baseline | 15 min |
| `reusable-security.yml` | pip-audit + bandit + TruffleHog (pinned) | 10 min |
| `reusable-claude-review.yml` | Claude Code AI code review | 20 min |

All workflows include: **job timeouts**, **TruffleHog pinned to `v3.93.3`**.

Callers are responsible for: **path filters** and **concurrency groups** (project-specific).

## Usage

In your project's `.github/workflows/tests.yml`:

```yaml
name: Tests

on:
  push:
    branches: [main]
    paths: [src/**, tests/**, requirements*.txt]
  pull_request:
    branches: [main]
    paths: [src/**, tests/**, requirements*.txt]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    uses: HendrikHarren/.github/.github/workflows/reusable-tests.yml@main
    with:
      python-version: '3.12'
      pytest-args: 'tests/unit/ -m "not slow" --cov=src --cov-report=html --cov-report=term-missing'
```

## Inputs Reference

### `reusable-tests.yml`
| Input | Default | Description |
|---|---|---|
| `python-version` | `3.12` | Python version |
| `pytest-args` | `tests/unit/ -m "not slow" --cov=src ...` | pytest arguments |
| `requirements-file` | `requirements-dev.txt` | Requirements file |

### `reusable-type-check.yml`
| Input | Default | Description |
|---|---|---|
| `python-version` | `3.12` | Python version |
| `mypy-baseline` | `0` | Max allowed mypy errors |
| `requirements-files` | `requirements.txt requirements-dev.txt` | Space-separated files |

### `reusable-security.yml`
| Input | Default | Description |
|---|---|---|
| `python-version` | `3.12` | Python version |
| `requirements-files` | `requirements.txt requirements-dev.txt` | Space-separated files |
| `bandit-target` | `src/` | Directory to scan |
| `notify-issue-assignee` | `` | GitHub username to @mention on failure |

### `reusable-claude-review.yml`
| Input | Default | Description |
|---|---|---|
| `runs-on` | _(required)_ | Runner-Label. Für ARC scale-set Runner den Scale-Set-Namen (z. B. `shared-runners`), NIE `self-hosted` (actions/actions-runner-controller#3330). |
| `claude-args` | `''` | Zusätzliche CLI-Argumente für claude-code (z. B. `--model claude-sonnet-4-6 --allowedTools ...`). |
| `review-prompt` | _(required)_ | **Repo-spezifischer** Review-Prompt (WAS reviewen: Stack, Fokus, Posting-Konvention). Severity-Skala und State-Awareness müssen NICHT mitgeliefert werden — die erbt der Caller zentral aus `review-protocol`. |
| `review-protocol` | _(Standard-Preamble)_ | Org-weite Review-Protokoll-Preamble (Severity-Gating + erschöpfender erster Pass + state-aware Folge-Runden). Single Source of Truth — nur in Ausnahmen überschreiben. |
| `show-full-output` | `false` | Volles Claude-Output loggen (WARNUNG: kann Secrets exponieren). |
| `allowed-non-write-users` | `HendrikHarren` | Komma-Liste von Usernamen, die den Write-Permission-Check umgehen (gegen 504-Timeouts der collaborators-API auf K8s-Runnern). |

**Secret required:** `CLAUDE_CODE_OAUTH_TOKEN`

### Review-Protokoll (Severity-Gating + State-Aware Rounds)

Der `reusable-claude-review.yml`-Workflow stellt jedem repo-spezifischen `review-prompt`
automatisch eine org-weite **Protokoll-Preamble** voran (Issue #13). Diese erzwingt:

- **Severity-Labels** an jedem Finding: `[BLOCKER]` / `[WARNING]` (beide blocken den Merge)
  und `[NIT]` (kosmetisch, blockt nicht, wird nicht als Issue nachverfolgt).
- **Runde 1: erschöpfender erster Pass** über den gesamten PR — alle Findings sofort,
  nichts für „später" aufsparen.
- **Folge-Runden: state-aware Delta-Review** — der Reviewer liest seine eigenen früheren
  Review-Comments ein, prüft nur das Delta + verifiziert alte Findings und meldet neue
  Findings nur bei `[BLOCKER]`-Severity (echte Regression durch den Fix).

Damit konvergieren Reviews in ≤3 Runden statt 5–6. Caller müssen nichts ändern; die
nötigen Tools (`gh pr view --comments`, `gh api`) sollten in `claude-args --allowedTools`
freigegeben sein (Standard in den Org-Callern).

## Updating TruffleHog Version

When a new TruffleHog release is available, update the pinned version in `reusable-security.yml`:
```yaml
uses: trufflesecurity/trufflehog@v3.XX.X  # update here
```
All repos using this workflow inherit the update automatically.
