---
title: "Pre-commit Hooks for a Bilingual Codebase: Python and R with uv"
date: 2025-06-01
draft: true
tags: ["python", "r", "developer-experience", "tooling"]
summary: "How I set up pre-commit hooks for a mixed Python/R project using uv for dependency management — covering linting, formatting, type checking, security scanning, and notebook output stripping."
---

When your codebase has both Python and R — scripts, notebooks, shared config — keeping code quality consistent is harder than it sounds. People forget to format, linting rules drift, and someone inevitably commits a Jupyter notebook with 200 cells of output. Pre-commit hooks fix this by running checks automatically before every commit, so the team doesn't have to think about it.

Here's how I configured pre-commit for a mixed Python/R project managed with [uv](https://docs.astral.sh/uv/).

## The setup

The project uses `uv` for Python dependency management. The `pyproject.toml` defines both production and dev dependencies, with pre-commit itself listed under the dev group:

```toml
[dependency-groups]
dev = [
    "mypy>=1.18.2",
    "pytest>=8.4.2",
    "pytest-cov>=7.0.0",
    "pre-commit>=4.3.0",
    "ruff>=0.13.2",
    "bandit[toml]>=1.8.6",
]

[build-system]
requires = ["uv_build>=0.8.18,<0.9.0"]
build-backend = "uv_build"

[tool.uv]
package = false
```

Installing hooks is two commands:

```bash
uv sync --group dev
uv run pre-commit install
```

After this, every `git commit` triggers the hook suite automatically.

## The hooks

The `.pre-commit-config.yaml` chains together hooks for Python, R, notebooks, and general file hygiene. Here's the full breakdown.

### Keeping uv.lock in sync

```yaml
- repo: https://github.com/astral-sh/uv-pre-commit
  rev: 0.9.5
  hooks:
    - id: uv-lock
```

This runs `uv lock` before every commit, so the lock file never drifts from `pyproject.toml`. Without it, you get the familiar "works on my machine" problem where someone adds a dependency but forgets to update the lock file.

### Python linting and formatting with Ruff

```yaml
- repo: https://github.com/astral-sh/ruff-pre-commit
  rev: v0.14.1
  hooks:
    - id: ruff-check
      args: [--fix]
    - id: ruff-format
```

Ruff handles both linting and formatting in one tool. The `--fix` flag auto-corrects what it can (unused imports, import ordering). The rule selection lives in `pyproject.toml`:

```toml
[tool.ruff]
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "UP", "B", "SIM", "I"]
```

This covers pycodestyle, Pyflakes, pyupgrade, flake8-bugbear, flake8-simplify, and isort — a good baseline that catches real bugs without being pedantic.

### General file checks

```yaml
- repo: https://github.com/pre-commit/pre-commit-hooks
  rev: v6.0.0
  hooks:
    - id: trailing-whitespace
    - id: end-of-file-fixer
    - id: check-yaml
    - id: check-toml
    - id: check-added-large-files
    - id: check-ast
    - id: check-builtin-literals
```

These are the pre-commit project's own hooks. `check-ast` catches Python syntax errors before they reach CI. `check-added-large-files` prevents someone from accidentally committing a data file.

### Type checking with mypy

```yaml
- repo: https://github.com/pre-commit/mirrors-mypy
  rev: v1.18.2
  hooks:
    - id: mypy
      files: ^src/
      additional_dependencies: [types-requests]
```

Scoped to `src/` so it doesn't choke on test files or scripts. The `additional_dependencies` field is how you give mypy access to type stubs — it runs in its own isolated environment, so it won't see your project's installed packages unless you list them here.

### Security scanning with Bandit

```yaml
- repo: https://github.com/PyCQA/bandit
  rev: 1.8.6
  hooks:
    - id: bandit
      args: [-c, pyproject.toml]
      additional_dependencies: ["bandit[toml]"]
```

Bandit flags common security issues in Python code — hardcoded passwords, use of `subprocess` without input validation, insecure hash functions. Configuration goes in `pyproject.toml`:

```toml
[tool.bandit]
exclude_dirs = ["tests", ".venv", "__pycache__"]
skips = ["B603", "B607", "B404"]
```

The skips are for `subprocess` calls that are expected in a project that orchestrates shell commands.

### Stripping notebook outputs

```yaml
- repo: https://github.com/kynan/nbstripout
  rev: 0.8.2
  hooks:
    - id: nbstripout
      files: \.ipynb$
```

Jupyter notebook outputs bloat diffs, leak data, and make code review impossible. This strips all cell outputs before commit. Non-negotiable in any project with notebooks.

### R formatting and linting

This is where things get more involved. There's no off-the-shelf pre-commit hook for R that handles both `.R` files and R code inside Jupyter notebooks. So I wrote two local hooks backed by shell scripts.

**Formatting with styler:**

```yaml
- repo: local
  hooks:
    - id: format-r
      name: Format R Files and Notebooks
      entry: ./scripts/hooks/format-r.sh
      language: script
      files: '\.(R|ipynb)$'
```

The script handles two cases. For plain `.R` files, it calls `styler::style_file()` directly. For `.ipynb` notebooks, it checks the kernel metadata to confirm it's an R notebook, then uses jupytext to convert to `.R`, formats with styler, and syncs back:

```bash
# For .R files:
Rscript -e "styler::style_file('$file', style = styler::tidyverse_style)"

# For R notebooks:
jupytext --set-formats ipynb,R "$file"
jupytext --to R "$file"
Rscript -e "styler::style_file('$r_file', style = styler::tidyverse_style)"
jupytext --sync "$file"
```

**Linting with lintr:**

```yaml
- repo: local
  hooks:
    - id: lint-r
      name: Lint R Files and Notebooks
      entry: ./scripts/hooks/lint-r.sh
      language: script
      files: '\.(R|ipynb)$'
```

Same pattern — lint `.R` files directly, convert notebooks via jupytext for linting, then clean up. The lintr configuration lives in `.lintr` at the repo root, with relaxed rules for things like line length and object naming that tend to generate noise rather than catch real issues.

## What this gets you

With all of this in place, `git commit` triggers the full suite. A typical run looks like:

1. Lock file check — ensures `uv.lock` matches `pyproject.toml`
2. Ruff lint and format — catches and fixes Python issues
3. File hygiene — trailing whitespace, valid YAML/TOML, no large files
4. mypy — type errors in `src/`
5. Bandit — security issues
6. nbstripout — clean notebook diffs
7. R format and lint — consistent R code style

If any hook fails, the commit is blocked and you see exactly what needs fixing. Most of the Python hooks auto-fix and restage, so often you just need to run `git commit` again.

The key thing is that none of this requires anyone to remember to run a formatter or linter. It's enforced at commit time, which means code review can focus on logic and design rather than style arguments.
