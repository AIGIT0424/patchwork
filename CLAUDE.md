# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Patchwork is a Python CLI (`patchwork-cli` on PyPI, AGPL-3.0) that runs LLM-assisted automations ("patchflows") for tasks like PR review, vulnerability autofix, dependency upgrades, docstring generation, and issue resolution. Patchflows are built by composing reusable atomic actions ("steps") and prompt templates.

## Tooling and common commands

The project uses Poetry (`poetry.toml` pins venv in-project under `.venv/`). Python 3.9 is the supported floor; Black targets py310 with line length 120.

```bash
# Install (build-from-source flow; see INSTALL.md)
poetry install --all-extras

# Run the CLI (entry point: patchwork.app:cli)
poetry run patchwork <PatchflowName> key=value --flag
poetry run patchwork --list                      # list available patchflows
poetry run patchwork <Patchflow> --config /path  # config file or directory

# Tests
poetry run pytest                                # full suite
poetry run pytest tests/steps/test_CallAPI.py    # single file
poetry run pytest tests/steps/test_CallAPI.py::test_call_api_outputs  # single test

# Lint / format (poethepoet task; runs autoflake, isort, then black)
poetry run poe lint
```

The `poe lint` task and `[tool.poe.poetry_hooks] post_update = "lint"` hook in `pyproject.toml` mean lint runs after `poetry update`. There is no separate build step beyond `poetry build`.

## Architecture

Three layers compose every run:

1. **Steps** (`patchwork/steps/<Name>/<Name>.py`) — atomic actions (call an LLM, scan with Semgrep, commit changes, create a PR, …). Each step subclasses `patchwork.step.Step`, declares optional `input_class` / `output_class` TypedDicts via class kwargs, validates required keys in `__init__`, and returns a dict from `run()`.
2. **Patchflows** (`patchwork/patchflows/<Name>/<Name>.py`) — orchestrators that load `defaults.yml` + `default_prompt.json`, then run a sequence of Steps, threading each step's output dict into `self.inputs` for the next step. Most patchflows currently subclass `Step` (not `Patchflow`); `patchflow.py`'s `Patchflow` base is a partial alternative. Use the existing AutoFix patchflow as the canonical example.
3. **CLI** (`patchwork/app.py`) — Click command that resolves the patchflow name, parses `key=value` and bare `--flag` opts into the inputs dict, merges them on top of `--config` YAML and the patchflow's `defaults.yml`, then constructs and runs the patchflow inside a `PatchedClient` telemetry context.

### Step contract (important specifics in `patchwork/step.py`)

- Subclass with `class MyStep(Step, input_class=MyInputs, output_class=MyOutputs):` — those TypedDicts drive required-key validation (`__required_keys__`) at construction time and feed the `validate_steps_with_inputs` helper.
- The base `__init__` wraps `run` in `__managed_run`, which logs status, honors `set_status(StepStatus.…)`, and rethrows exceptions. Don't override `__managed_run`; just implement `run(self) -> DataPoint`.
- `inputs.get("debug")` triggers an interactive pause before each step (waits for Enter on stdin). API-key-like keys are masked in debug output.

### Patchflow conventions

- Place defaults in `defaults.yml`, prompt in `default_prompt.json` next to the patchflow class. Load with `yaml.safe_load(_DEFAULT_INPUT_FILE.read_text())` and `final_inputs.update(inputs)` so user inputs win.
- Register progress reporting with `PatchflowProgressBar(self).register_steps(...)`.
- Call `validate_steps_with_inputs(set(inputs.keys()).union({...}), Step1, Step2, ...)` in `__init__` to fail fast on missing keys before any LLM call.
- Set conventional output keys: `pr_title`, `branch_prefix=f"{name.lower()}-"`, and any `response_partitions` used by `ExtractModelResponse`.
- The CI workflow excludes branches matching `autofix-*`, `dependencyupgrade-*`, `generatereadme-*`, `generatedocstring-*`, `generateunittests-*`, `generatecodeusageexample-*`, `resolveissue-*` — keep that list in sync if you add a patchflow that opens PRs.

### Adding a step or patchflow

Mirror existing folders and update the package's `__init__.py` (`patchwork/steps/__init__.py` or `patchwork/patchflows/__init__.py`) to import the new class and add it to `__all__`. The CLI's `--list` discovers patchflows by scanning for `patchflows/<Name>/<Name>.py` files, so the folder + filename must match the class name. Note the "Compatibility Aliases" section at the bottom of `steps/__init__.py` (e.g. `JoinListPB = JoinList`) — keep aliases when renaming a public step.

### LLM clients

`patchwork/common/client/llm/` contains per-provider clients (`openai_.py`, `anthropic.py`, `google.py`) plus `aio.py`, a fan-in client that picks whichever provider matches the configured key/model. `CallLLM` (`patchwork/steps/CallLLM/CallLLM.py`) wires inputs (`openai_api_key`, `google_api_key`, `anthropic_api_key`, `patched_api_key`, `client_base_url`, `model`, `model_*` extras) into those clients. Default model is `gpt-4o-mini`. Inputs prefixed `model_` flow through as kwargs to the model call; `client_` kwargs go to the OpenAI client constructor (this is how custom `client_base_url` for OpenAI-compatible endpoints works).

### Optional extras

`pyproject.toml` defines extras `security` (semgrep, owasp-depscan — needed for AutoFix/DependencyUpgrade), `rag` (chromadb, sentence-transformers, torch — needed for ResolveIssue), `notifications` (slack-sdk), and `all`. CI splits these into separate jobs (`all-test`, `security-test`, `rag-test`, `main-test`) — when adding a step, install the matching extra locally rather than reaching for `all` if you want to mirror CI.

## Style (from `style.md`)

- snake_case for Python; descriptive names.
- Each module gets an `__init__.py`; export new public symbols there and update `__all__`.
- TypedDict for step `input_class`/`output_class`; `Annotated[..., StepTypeConfig(...)]` for cross-key validation rules consumed by `validate_step_with_inputs`.
- Dynamic patchflow loading uses `importlib.util.spec_from_file_location` + `module_from_spec` (see `find_patchflow` in `app.py`) — do not switch to `import_module` for user-supplied paths.
- Avoid f-strings with untrusted user input.
