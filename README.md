# unity-coding-skills

A [Claude Code](https://claude.ai/code) plugin providing skills and agents for developing Unity projects — maintainable test design and implementation, test-first workflow, coding guidelines, scene editing, and more.

> [!NOTE]
> This is a fork of [nowsprinting/unity-coding-skills](https://github.com/nowsprinting/unity-coding-skills) by [Koji Hasegawa (@nowsprinting)](https://github.com/nowsprinting). The skills, agents, and guides are his original work, and all credit for the design and content belongs to him. This fork only swaps the Unity Editor backend from JetBrains MCP to [unity-cli](https://github.com/bigdra50/unity-cli) and distributes it under the `bigdra50` marketplace name. The upstream repository remains the canonical source.

## Included Skills

| Skill                      | Description                                                                                                    | Required                                                                                            |
|----------------------------|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| `code-writing-guide`       | Coding conventions and guidelines for Unity C# projects                                                        |                                                                                                     |
| `edit-scene`               | Creates and modifies `.unity` and `.prefab` files                                                              | [unity-cli](https://github.com/bigdra50/unity-cli) (`u`) + a running Unity Editor via the Relay Server |
| `fix-bug`                  | Diagnoses and fixes bugs using a test-first workflow (reproduce, diagnose, fix)                                |                                                                                                     |
| `plan-feature`             | Orchestrates the test-first planning workflow for feature implementation in plan mode                          |                                                                                                     |
| `refine-tests`             | Reviews existing test code for conformance to the test design and writing guides, then plans the refinement    |                                                                                                     |
| `run-tests`                | Running Unity tests via `u tests run`                                                                          | [unity-cli](https://github.com/bigdra50/unity-cli) (`u`) + a running Unity Editor via the Relay Server |
| `test-designing-guide`     | Design maintainable test cases; reduce redundant tests, tests without assertions, and unnecessary test doubles |                                                                                                     |
| `test-writing-guide`       | Conventions for writing Unity Test Framework test code                                                         | [Test Helper](https://github.com/nowsprinting/test-helper) and [UI Test Helper](https://github.com/nowsprinting/test-helper.ui) package |
| `unity-yaml-editing-guide` | Guidelines for directly hand-editing Unity YAML asset files                                                    |                                                                                                     |

## Included Agents

| Agent                 | Description                                                                                                             |
|-----------------------|-------------------------------------------------------------------------------------------------------------------------|
| `failing-test-writer` | Implements test code from the plan file's Test Cases table and confirms tests fail as expected (Step 2 of dev workflow) |
| `test-deduplicator`   | Removes duplicate tests and merges parameterizable tests in modified test files (Step 4 of dev workflow)                |
| `test-designer`       | Designs test cases during plan mode after class/method designs are produced, using the `test-designing-guide` skill     |

## Installation

### User-scope installation

Add the marketplace and install the plugin:

```shell
/plugin marketplace add bigdra50/unity-coding-skills
/plugin install unity-coding-skills@bigdra50-unity-coding-skills
```

### Project-scope installation (team sharing)

Add the marketplace and install the plugin with `--scope project`:

```shell
/plugin marketplace add bigdra50/unity-coding-skills
/plugin install unity-coding-skills@bigdra50-unity-coding-skills --scope project
```

Commit the resulting `.claude/settings.json` to your repository.

> [!NOTE]
> When team members trust the project folder, Claude Code prompts them to install the marketplace and plugin automatically.

## Recommended Project Settings

### 1. unity-cli (Unity Editor backend)

Most skills drive the Unity Editor through [unity-cli](https://github.com/bigdra50/unity-cli) — the `u` command — which talks to a running Editor over a local Relay Server (TCP:6500).

1. Install `unity-cli` and put the `u` command on your `PATH` (see the unity-cli README).
2. Start the Relay Server: `unity-relay --port 6500` (override the port with `UNITY_RELAY_PORT`).
3. Open your Unity project in the Editor, then confirm the connection:

```shell
u instances            # list connected Editors
u -i <project> state   # play mode / compilation / active scene of one Editor
```

`<instance>` in the skill docs is the target Editor — a project name, path, or unique prefix from `u instances`. When several Editors are connected, always pass `-i <instance>` so commands reach the right one.

### 2. Diagnostics tooling (jb / unilyze)

The `fix-bug`, `plan-feature`, and `refine-tests` skills resolve analyzer diagnostics that honor `.editorconfig` severity (which the Unity compiler does not surface). Install the ReSharper command-line inspector:

```shell
dotnet tool install -g JetBrains.ReSharper.GlobalTools   # provides `jb inspectcode`
```

Optionally install `unilyze` for complementary complexity/maintainability metrics. See the `code-writing-guide` skill's `diagnostics-review-feedback.md` for usage.

### 3. Enforcing coding rules via `.editorconfig`

Any coding rules or Roslyn analyzer diagnostics you want Claude to respect should be set to `warning` or higher severity in `.editorconfig`.

For example, to prevent leaving unused code, add the following diagnostics:

```
resharper_unused_type_local_highlighting = warning
resharper_unused_type_global_highlighting = warning
resharper_unused_member_global_highlighting = warning
resharper_unused_member_local_highlighting = warning
```

For complexity metrics — cyclomatic / cognitive complexity, plus CodeHealth and code-smell detection — run `unilyze` (see *Diagnostics tooling* above).

## Usage

### Test-first feature implementation planning

Type in plan mode:

```bash
/plan-feature <SPEC>
```

The created plan file includes the following:

- Layered-designed test cases
  - Reduce redundant tests, tests without assertions, and unnecessary test doubles
  - Editor tests
  - Unit tests (Play Mode tests)
  - Integrated tests including UI operation
  - Visual verification tests using image analysis
- Test-first development workflow
  - Effective (failable) test code
  - Definition of Done

### Bug fixes through reproduction testing

Type out of plan mode:

```bash
/fix-bug <INCIDENT>
```

First, create, run, and verify a test that reproduces the bug, and then fix the bug.

### Refine existing test code for conformance to the test design and writing guides

Type in plan mode:

```bash
/refine-tests <PATH>
```

## Contributing

Contributions are welcome. However, we will decline contributions that we cannot maintain — such as adding support for different coding agents or MCP servers. Please fork this repository and customize it for your needs instead.

## License

This project is released into the public domain under the [Unlicense](LICENSE).
