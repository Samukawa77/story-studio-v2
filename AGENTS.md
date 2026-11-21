# Repository Guidelines

## Project Structure

Story Studio is a portable Agent Skill with no build step. `SKILL.md` defines workflow routing and behavior, `README.md` documents usage, and `references/` holds shared workflow rules. For module changes, start with the relevant `modules/*/GUIDE.md`.

- `history-writing/` and `characterstory-writing/` contain writing rules and sample corpora under `corpus/`.
- `history-image/` handles storyboards and transcription, with visual references in `assets/` and style rules in `styles/`.
- `render-history-video/` handles timeline alignment, Jianying draft export, and optional MP4 rendering.

Root `scripts/` contains bootstrap, shared runtime configuration, Doubao TTS, and handoff validation. Tests sit beside the scripts they cover.

## Local Development

- `./install.sh` deploys the skill to a local Agent skills directory. On Windows, use `install.ps1` or `install.bat`.
- `bash scripts/bootstrap.sh check --features draft,asr,images,tts` checks runtime dependencies. Replace `check` with `install` to install the selected components. See `references/setup.md` for PowerShell equivalents.
- `python3 modules/history-writing/scripts/history_tool.py search --query 左宗棠` runs a local corpus search.

## Coding and Documentation Style

Use four spaces and `snake_case` for Python functions, variables, and script names. Match the surrounding code. The repo has no configured formatter or linter. Write skill instructions in Chinese, preserve identifiers such as `writing_profile` and `style_id`, and update both `SKILL.md` and the relevant module guide when adding a route or style. Reuse `scripts/studio_runtime.py` for runtime paths and configuration. Keep credentials, machine-specific paths, and generated projects out of commits.

## Testing

Use Python 3.10+ and `unittest`. Name files `test_*.py` and methods `test_*`. The commands below run discovery from each script directory so sibling imports resolve.

```sh
(cd scripts && python3 -m unittest discover)
(cd modules/history-image/scripts && python3 -m unittest discover)
(cd modules/render-history-video/scripts && python3 -m unittest discover)
```

On macOS, resolve the temporary directory before running bootstrap tests with `export TMPDIR="$(cd "${TMPDIR:-/tmp}" && pwd -P)"`. On Windows, also run `powershell -ExecutionPolicy Bypass -File scripts/test_bootstrap_native.ps1` from the repository root.

Use mocks and temporary fixtures for network calls and user data. Draft export tests require `pyJianYingDraft` and Pillow. There is no configured coverage threshold.

## Commits and Pull Requests

Follow the history’s conventional prefixes, such as `feat:`, `docs:`, and `chore(license):`, with a short summary. In pull requests, describe the affected workflow and tests run, link related issues when available, and include a sample output or screenshot when a visual change needs review.
