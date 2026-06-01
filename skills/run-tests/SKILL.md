---
name: run-tests
description: >-
  Provides guidelines for running Unity tests with the `u tests run` command (unity-cli).
  Make sure to use this skill whenever running, executing, or re-running tests on the Unity editor.
  This includes verifying implementations, debugging test failures, running specific test assemblies, or any task that involves running Unity tests.
  Even if the user just says "run the tests" or "check if it passes", use this skill.
license: Unlicense
metadata:
  author: Koji Hasegawa
---

## Gotchas

- **Serialize editor commands around compilation and domain reloads.** A `refresh`, `tests run`, or Play Mode change can trigger script recompilation / domain reload; issuing the next command mid-reload fails or returns stale state. Wait for `isCompiling` (and `isPlaying` after a Play Mode change) to settle via `u -i <instance> state --json` before issuing the next command — do not fire commands back-to-back.
- **On a failed or empty-looking result, re-check state before retrying.** If a command errors or output is unexpectedly empty, poll `u -i <instance> state --json` until `isCompiling` is `false`, then retry once. If the same run fails on two consecutive attempts, stop and consult the user instead of looping.

## Run Tests

Before running tests, complete the following steps in order:

1. If any code was modified, confirm compilation succeeds first with the **Quick Verify** sequence: `u -i <instance> console clear` → `u -i <instance> refresh` → poll `u -i <instance> state --json` until `isCompiling` is `false` (≈2 s interval, up to ~30 s) → `u -i <instance> console get -l E`. Resolve any error before running tests. (`<instance>` is the target Unity Editor; run `u instances` to list connected editors.)
2. To determine the assembly and test mode for a specific test class, run `${CLAUDE_SKILL_DIR}/scripts/resolve-test-target.sh <test-class-cs-path>`. It prints `<assemblyName>\t<testMode>` (e.g. `MyGame.Tests\tPlayMode`). Skip this step when the assembly is already known.

Then run the tests with `u` (map `testMode` to the `u` mode argument: `EditMode` → `edit`, `PlayMode` → `play`):

```bash
u -i <instance> tests run edit                                # all EditMode tests
u -i <instance> tests run play                                # all PlayMode tests
u -i <instance> tests run edit -a <assemblyName>              # one assembly (from step 2)
u -i <instance> tests run edit -n <Namespace.Class.Method>    # a single test (full name)
u -i <instance> tests run edit -g "<regex>"                   # tests whose name matches a regex
```

`u tests run` waits for completion and prints a pass/fail summary; its exit code is non-zero (5) when any test fails. Filters (`-a`/`-n`/`-c`/`-g`) are ANDed together. Test execution can take several minutes — do not start a second run while one is in progress (check with `u -i <instance> tests status`). If it times out, narrow the run with filters and retry. To start without blocking, add `--no-wait` and poll `u -i <instance> tests status`.

## Rules for Test Failures

If the same test(s) fail on two or more consecutive runs, stop and consult the user rather than continuing to fix.

When consulting, clarify:

- Current failure status: what is failing and the likely cause
- Fix history: what was changed, how many times, and the scope of impact
- Planned approach: what options are being considered next

## Troubleshooting

Read the appropriate resource file based on the situation:

- `u tests run` fails, hangs, or the editor is not reachable (no connected instance, or the Relay Server is not running): Read `${CLAUDE_SKILL_DIR}/resources/troubleshooting-run-unity-tests.md`
- A test fails due to an assertion, constraint, or comparer in the `TestHelper` namespace (excluding `TestHelper.UI`): Read `${CLAUDE_SKILL_DIR}/resources/troubleshooting-test-helper.md`
- A test fails due to an exception thrown from the `TestHelper.UI` namespace: Read `${CLAUDE_SKILL_DIR}/resources/troubleshooting-test-helper-ui.md`
