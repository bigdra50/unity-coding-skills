# Troubleshooting `run_tests`

`unity command run_tests --mode <editor|playmode>` runs Unity Test Runner suites in the Editor that has the project open, and returns a JSON result with a `summary` (total / passed / failed / skipped) and per-test `results`. It goes through the `com.unity.pipeline` package; the project is detected from the working directory (or pass `--project-path <path>`).

## The editor is not reachable

1. **No Unity Editor running** — open the Unity Editor with the target project. `unity command editor_status` should answer with `"status": "ready"`.
2. **Pipeline package missing** — `unity command` cannot reach an Editor without `com.unity.pipeline`. Install it once with `unity pipeline install`, then let the Editor import it.
3. **Wrong project** — when running outside the project directory, pass `--project-path <path>` so the command reaches the right Editor.
4. **Editor busy** — if a compile or domain reload is in flight, the run can fail to start. Poll `unity command editor_status` until `compiling` and `domainReloadInProgress` are both `false`, then start the run.

## Compilation errors block the run

Tests cannot run while compile errors are present. Before running, complete Quick Verify and resolve every error:

```bash
unity command clear_console
unity command recompile
# poll until status is completed or up_to_date (≈2 s interval, up to ~30 s)
unity command recompile_status
unity command console --level error     # must be empty, and compilationFailed must be false
```

## No tests ran / zero results

The filter matched nothing.

1. **List what is discoverable** — `unity command list_tests --mode <editor|playmode>` prints every test in that mode. Confirm your target appears.
2. **Mode mismatch** — EditMode tests only run under `--mode editor`, PlayMode tests only under `--mode playmode`. Map `resolve-test-target.sh`'s `testMode` output: `EditMode` → `editor`, `PlayMode` → `playmode`.
3. **Assembly name** — with `--filter_type assembly`, pass the exact `.asmdef` `name` to `--filter`. Derive it with `${CLAUDE_SKILL_DIR}/scripts/resolve-test-target.sh <test-file>`.
4. **Filter type** — only one filter applies per run. `testName` (the default) is a case-insensitive partial match on the test name; `category` needs an NUnit category name.

## The run hangs or times out

1. **Narrow the scope** — add `--filter` (with `--filter_type`) to run fewer tests per call.
2. **Raise both timeouts** — the CLI waits 30 s by default; `run_tests` itself waits up to 300 s. Raise the CLI side with `unity command --timeout 600 run_tests ...`.
3. **Run non-blocking** — start with `--async_tests true`, then poll `unity command test_status`. Stop a stuck run with `unity command cancel_tests`. Do not start a second run while one is in progress.
4. **Suspected infinite loop** — a test that never returns blocks the run. Add `Debug.Log` at the start of each suspect test, run them one at a time with `--filter`, and inspect `unity command console` / `Editor.log` to see which test started last.

## PlayMode specifics

- A crash or exception during a PlayMode run can leave the Editor in Play Mode or mid-reload. Check `unity command editor_status`; if `playMode` is not `stopped`, run `unity command editor_stop`, wait for `"status": "ready"`, and retry once.

## Reading failures

The `results` entries of failed tests carry the message and stack trace. For exceptions logged outside the assertion (e.g. in `SetUp` or from `Debug.LogException`), also read the console: `unity command console --level error`.

## Logs to investigate

- **Unity Console (primary)** — `unity command console` (entries include stack traces; `--level warn` / `--level error` filter by minimum severity, `--tail <n>` limits the count).
- **Unity Editor log** — `~/Library/Logs/Unity/Editor.log` (macOS), `%LOCALAPPDATA%\Unity\Editor\Editor.log` (Windows), `~/.config/unity3d/Editor.log` (Linux): editor-side crashes, domain reloads, compile errors.
