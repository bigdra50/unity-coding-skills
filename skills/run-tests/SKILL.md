---
name: run-tests
description: >-
  Provides guidelines for running Unity tests with the `unity command run_tests` command (official Unity CLI).
  Make sure to use this skill whenever running, executing, or re-running tests on the Unity editor.
  This includes verifying implementations, debugging test failures, running specific test assemblies, or any task that involves running Unity tests.
  Even if the user just says "run the tests" or "check if it passes", use this skill.
license: Unlicense
metadata:
  author: Koji Hasegawa
---

## Gotchas

- **`unity command` needs the pipeline package.** It reaches the Unity Editor that has the project open through the `com.unity.pipeline` package (install it once with `unity pipeline install`). The project is detected from the working directory; from elsewhere, add `--project-path <path>`.
- **Serialize editor commands around compilation and domain reloads.** A `recompile`, `run_tests`, or Play Mode change can trigger script recompilation / domain reload; issuing the next command mid-reload fails or returns stale state. Wait until `unity command editor_status` reports `"status": "ready"` (or `"playing"` after entering Play Mode) before issuing the next command — do not fire commands back-to-back.
- **On a failed or empty-looking result, re-check state before retrying.** If a command errors or output is unexpectedly empty, poll `unity command editor_status` until `compiling` and `domainReloadInProgress` are both `false`, then retry once. If the same run fails on two consecutive attempts, stop and consult the user instead of looping.
- **Raise the CLI timeout for test runs.** `unity command` gives up after 30 s by default, while `run_tests` itself waits up to its own `--timeout` (300 s). Pass the CLI's `--timeout` too, e.g. `unity command --timeout 600 run_tests ...`.

## Run Tests

Before running tests, complete the following steps in order:

1. If any code was modified, confirm compilation succeeds first with the **Quick Verify** sequence: `unity command clear_console` → `unity command recompile` → poll `unity command recompile_status` until `status` is `completed` or `up_to_date` (≈2 s interval, up to ~30 s) → `unity command console --level error`. Resolve any error (or `compilationFailed: true`) before running tests.
2. To determine the assembly and test mode for a specific test class, run `${CLAUDE_SKILL_DIR}/scripts/resolve-test-target.sh <test-class-cs-path>`. It prints `<assemblyName>\t<testMode>` (e.g. `MyGame.Tests\tPlayMode`). Skip this step when the assembly is already known.

Then run the tests (map `testMode` to `--mode`: `EditMode` → `editor`, `PlayMode` → `playmode`):

```bash
unity command --timeout 600 run_tests --mode editor                                              # all EditMode tests
unity command --timeout 600 run_tests --mode playmode                                            # all PlayMode tests
unity command --timeout 600 run_tests --mode editor --filter_type assembly --filter <assemblyName> # one assembly (from step 2)
unity command --timeout 600 run_tests --mode editor --filter <Namespace.Class.Method>            # tests whose name contains the text
unity command --timeout 600 run_tests --mode editor --filter_type category --filter <Category>   # one NUnit category
```

`run_tests` waits for completion and returns JSON with a `summary` (total / passed / failed / skipped) and per-test `results` (full name, status, message, stack trace); judge pass/fail from the summary's failed count. One filter applies per run: `--filter_type` picks testName (default, case-insensitive partial match), assembly, or category. Test execution can take several minutes — do not start a second run while one is in progress (check with `unity command test_status`). If it times out, narrow the run with a filter and retry. To start without blocking, add `--async_tests true` and poll `unity command test_status`.

## Rules for Test Failures

If the same test(s) fail on two or more consecutive runs, stop and consult the user rather than continuing to fix.

When consulting, clarify:

- Current failure status: what is failing and the likely cause
- Fix history: what was changed, how many times, and the scope of impact
- Planned approach: what options are being considered next

## Troubleshooting

Read the appropriate resource file based on the situation:

- `run_tests` fails, hangs, or the editor is not reachable (no running Editor, or the pipeline package is missing): Read `${CLAUDE_SKILL_DIR}/resources/troubleshooting-run-unity-tests.md`
- A test fails due to an assertion, constraint, or comparer in the `TestHelper` namespace (excluding `TestHelper.UI`): Read `${CLAUDE_SKILL_DIR}/resources/troubleshooting-test-helper.md`
- A test fails due to an exception thrown from the `TestHelper.UI` namespace: Read `${CLAUDE_SKILL_DIR}/resources/troubleshooting-test-helper-ui.md`
