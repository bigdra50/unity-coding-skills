# Troubleshooting `u tests run`

`u -i <instance> tests run <edit|play>` runs Unity Test Runner suites over the Relay Server and waits for the result (exit code 0 = all passed, 5 = one or more failed). `<instance>` is the target Unity Editor; run `u instances` to list connected editors.

## The editor is not reachable

1. **No Unity Editor connected** — `u instances` prints an empty list. Open the Unity Editor with the target project and start the Relay Server with `unity-relay --port 6500` (override the port with `UNITY_RELAY_PORT`).
2. **Wrong `-i <instance>`** — the value must match a `Project` name from `u instances` (path / project name / unique prefix).
3. **Editor busy** — if a compile or domain reload is in flight, the run can fail to start. Poll `u -i <instance> state --json` until `isCompiling` is `false`, then start the run.

## Compilation errors block the run

Tests cannot run while compile errors are present. Before running, complete Quick Verify and resolve every error:

```bash
u -i <instance> console clear
u -i <instance> refresh
# poll until isCompiling == false
u -i <instance> state --json
u -i <instance> console get -l E        # must be empty
```

## No tests ran / zero results

The filter matched nothing.

1. **List what is discoverable** — `u -i <instance> tests list <edit|play>` prints every test name in that mode. Confirm your target appears.
2. **Mode mismatch** — EditMode tests only run under `edit`, PlayMode tests only under `play`. Map `resolve-test-target.sh`'s `testMode` output: `EditMode` → `edit`, `PlayMode` → `play`.
3. **Assembly name** — pass the exact `.asmdef` `name` to `-a`. Derive it with `${CLAUDE_SKILL_DIR}/scripts/resolve-test-target.sh <test-file>`.
4. **Filter spelling** — `-n` needs the fully-qualified test name (`Namespace.Class.Method`); `-c` needs an exact NUnit category; `-g` is a regex on the test name. Filters are ANDed — over-constraining yields zero matches.

## The run hangs or times out

1. **Narrow the scope** — add `-a` / `-n` / `-c` / `-g` to run fewer tests per call.
2. **Raise the timeout** — `u` has a global `--timeout, -t <seconds>` option: `u -i <instance> -t 600 tests run play`.
3. **Run non-blocking** — start with `--no-wait`, then poll `u -i <instance> tests status` (reports running / started / finished / passed / failed / skipped). Do not start a second run while one is in progress.
4. **Suspected infinite loop** — a test that never returns blocks the run. Add `Debug.Log` at the start of each suspect test, run them one at a time with `-n`, and inspect `u -i <instance> console get -s` / `Editor.log` to see which test started last.

## PlayMode specifics

- PlayMode runs disable Domain Reload automatically (to keep the Relay connection alive across entering Play Mode) and restore the setting afterward. Tests that depend on a domain reload between play-mode entries may behave differently — prefer EditMode for such cases, or account for it explicitly.
- A crash or exception during a PlayMode run can drop the connection; re-check `u instances` and `u -i <instance> state --json`, wait for `isCompiling` to settle, and retry once.

## Reading failures

`u tests run` prints the failed test names and messages in its summary; the exit code is 5 on failure. For the full exception / stack trace of a failing test, also read the console: `u -i <instance> console get -l E,X -s`.

## Logs to investigate

- **Unity Console (primary)** — `u -i <instance> console get -s`: assertions, exceptions, `Debug.Log` from tests.
- **Unity Editor log** — `~/Library/Logs/Unity/Editor.log` (macOS), `%LOCALAPPDATA%\Unity\Editor\Editor.log` (Windows), `~/.config/unity3d/Editor.log` (Linux): editor-side crashes, domain reloads, compile errors.
- **Relay Server output** — the terminal running `unity-relay`, if runs never start.
