# Ethereum Consensus Specs

Executable specifications for Ethereum's consensus layer. Specs are written in
markdown with embedded Python; a build pipeline generates executable Python
modules ("pyspec") from them.

## Quick Reference

```bash
make test                              # Run all tests (minimal preset, parallel)
make test fork=deneb k=test_name       # Single test on a specific fork
make test preset=mainnet fork=electra  # Mainnet preset
make lint                              # Linters, formatters, type checks
make coverage fork=electra             # Coverage report (opens in browser)
make _pyspec                           # Regenerate pyspec from markdown (auto-run by test/lint)
make serve_docs                        # Local docs server at http://127.0.0.1:8000
make clean                             # Delete all untracked files (git clean -fdx)
```

## Build System

- **Dependency manager**: `uv` (install: `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- **Python**: 3.10–3.13
- **Config**: `pyproject.toml` is the single source of truth; `uv.lock` is committed
- `make _sync` runs `uv sync --all-extras` automatically before most targets

## Test Parameters

| Param | Values | Default |
|-------|--------|---------|
| `k` | test name pattern | (all) |
| `fork` | phase0, altair, bellatrix, capella, deneb, electra, fulu, gloas, heze | (all) |
| `preset` | minimal, mainnet | minimal |
| `bls` | py_ecc, milagro, arkworks, fastest | fastest |
| `kzg` | spec, ckzg | ckzg |
| `component` | all, pyspec, fw | all |

When `k=` is set, parallelism is disabled automatically for easier debugging.

## Fork Hierarchy

```
phase0 → altair → bellatrix → capella → deneb → electra → fulu → gloas → heze
                                          │        │                │
                                          capella  deneb            fulu
                                          └ eip7441 └ eip6800       ├ eip7928
                                                                    └ eip8025
```

Defined in `pysetup/md_doc_paths.py:PREVIOUS_FORK_OF`. Each fork inherits all
ancestor specs. Feature branches (eipNNNN) fork off specific points.

## Key Directories

| Path | Purpose |
|------|---------|
| `specs/<fork>/` | Source-of-truth markdown specs |
| `specs/_features/eipNNNN/` | Experimental feature specs |
| `presets/{mainnet,minimal}/<fork>.yaml` | Compile-time constants (affect type sizes) |
| `configs/{mainnet,minimal}.yaml` | Runtime network parameters |
| `pysetup/` | Markdown→Python generation pipeline |
| `pysetup/spec_builders/<fork>.py` | Per-fork spec builder customization |
| `tests/core/pyspec/eth_consensus_specs/<fork>/` | Generated pyspec (do not edit) |
| `tests/core/pyspec/eth_consensus_specs/test/<fork>/` | Test files |
| `tests/core/pyspec/eth_consensus_specs/test/helpers/` | Shared test helpers |
| `tests/core/pyspec/eth_consensus_specs/test/context.py` | Test decorators and context |

## Conventions

### Spec Markdown

- Python code blocks must be valid, fully type-hinted Python
- Keep code simple for multi-language portability (avoid map/filter/lambda)
- Mark changes with fork comments:
  - `# [New in Fork:EIP]` — new field or function
  - `# [Modified in Fork:EIP]` — changed existing code

### Presets vs Configs

- **Presets** (`presets/`): compile-time limits that affect container sizes (e.g. `MAX_COMMITTEES_PER_SLOT`). Changing requires spec regeneration.
- **Configs** (`configs/`): runtime network parameters (e.g. `GENESIS_DELAY`). Can change without recompilation.

### Test Decorators (from `context.py`)

- `@with_all_phases` — run on every fork
- `@with_phases([FORK1, FORK2])` — specific forks
- `@with_deneb_and_later` (and similar) — fork and all successors
- `@spec_state_test` — state transition test (yields pre/post state)
- `@spec_test` — general spec test
- `@always_bls` — force BLS verification on

### Test Pattern

```python
@with_all_phases
@spec_state_test
def test_example(spec, state):
    yield "pre", state
    # ... modify state ...
    yield "post", state
```

## Lint Checks (run by `make lint`)

uv lock check, codespell, fork comment validation, trailing whitespace,
markdown heading validation, value annotation check, mdformat (wrap 80),
ruff (check + format), mypy

## Reference Test Generation

```bash
make reftests                                    # All
make reftests runner=operations fork=fulu        # Specific runner and fork
make reftests runner=operations k=case1,case2    # Specific cases
```

Output goes to `../consensus-spec-tests/tests/`.
