# Python Virtual Environments: venv vs uv

*Published: 2026-02-02*

When working with Python projects, managing dependencies in isolated environments is essential. This post compares two approaches: the built-in `venv` and the modern `uv` tool.

---

## venv (Built-in)

Python's standard virtual environment tool - comes with Python, no installation needed.

### Usage

```bash
# Create a virtual environment
python3 -m venv .venv

# Activate it
source .venv/bin/activate  # Linux/Mac
.venv\Scripts\activate     # Windows

# Install packages
pip install -r requirements.txt

# When done, deactivate
deactivate
```

### Pros
- ✅ Built into Python 3.3+
- ✅ No extra installation required
- ✅ Universally understood
- ✅ Works everywhere Python works

### Cons
- ❌ Slow dependency resolution
- ❌ No lock file support out of the box
- ❌ Basic features only
- ❌ pip can have dependency conflicts

---

## uv (Modern, Fast)

A Rust-based package manager by [Astral](https://astral.sh) (the creators of Ruff). It's designed as a drop-in replacement for pip, pip-tools, virtualenv, and pyenv.

### Installation

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# Or via pip
pip install uv
```

### Usage

```bash
# Create venv + install in one go
uv venv
uv pip install -r requirements.txt

# Even simpler - auto-creates venv if needed
uv pip install flask requests

# Initialize a new project with pyproject.toml
uv init myproject
cd myproject

# Add dependencies (updates pyproject.toml + uv.lock)
uv add flask
uv add pytest --dev

# Sync environment from lock file
uv sync

# Run commands in the venv without activating
uv run python app.py
uv run pytest
```

### Pros
- ✅ **10-100x faster** than pip
- ✅ Better dependency resolution (no conflicts)
- ✅ Built-in lock files (`uv.lock`) for reproducibility
- ✅ Replaces multiple tools (pip, pip-tools, virtualenv, pyenv)
- ✅ `uv run` - no need to activate venv
- ✅ Cross-platform consistency

### Cons
- ❌ Newer tool (less battle-tested)
- ❌ Requires installation
- ❌ Team needs to adopt it

---

## Head-to-Head Comparison

| Feature | venv + pip | uv |
|---------|-----------|-----|
| Installation | Built-in | Required |
| Speed | Slow | 10-100x faster |
| Lock files | ❌ (needs pip-tools) | ✅ Built-in |
| Dependency resolution | Basic | Advanced |
| Python version management | ❌ (needs pyenv) | ✅ Built-in |
| Learning curve | Minimal | Minimal |
| Maturity | Very mature | Newer (but stable) |

---

## When to Use What

### Use `venv` when:
- Learning Python or writing quick scripts
- Working in environments where you can't install tools
- Contributing to projects that use venv
- Maximum compatibility is required

### Use `uv` when:
- Starting new projects
- Speed matters (CI/CD pipelines)
- You need reproducible builds across team/environments
- Managing multiple Python versions
- You want a modern developer experience

---

## Quick Migration

Already have a `requirements.txt`? Migrate to uv in seconds:

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create venv and install existing requirements
uv venv
uv pip install -r requirements.txt

# Generate a lock file for reproducibility
uv pip compile requirements.txt -o requirements.lock
```

Or start fresh with `pyproject.toml`:

```bash
uv init myproject
cd myproject
uv add flask sqlalchemy  # adds to pyproject.toml + uv.lock
```

---

## Recommendation

**For new projects in 2026: use `uv`.**

The speed difference alone is worth it - waiting for pip to resolve dependencies is a thing of the past. The built-in lock file support ensures your team and CI/CD get identical environments every time.

That said, `venv` isn't going anywhere. It's the standard, it's reliable, and it's always available. Know both, choose based on your needs.

---

## Resources

- [uv Documentation](https://docs.astral.sh/uv/)
- [Python venv Documentation](https://docs.python.org/3/library/venv.html)
- [Astral (uv creators)](https://astral.sh)
