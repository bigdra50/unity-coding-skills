# unity-coding-skills

A [Claude Code](https://claude.ai/code) plugin providing skills and an agent for developing Unity projects — maintainable test design, test-first workflow, coding guidelines, scene editing, and more.

## Included Skills

| Skill                      | Description                                                                           | Required                                                                                                                                                                                                     |
|----------------------------|---------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `code-writing-guide`       | Coding conventions and guidelines for Unity C# projects                               |                                                                                                                                                                                                              |
| `edit-scene`               | Creates and modifies `.unity` and `.prefab` files                                     | JetBrains [MCP server](https://www.jetbrains.com/help/rider/mcp-server.html) and [MCP Server Extension for Unity](https://plugins.jetbrains.com/plugin/30357-mcp-server-extension-for-unity) plugin |
| `fix-bug`                  | Diagnoses and fixes bugs using a test-first workflow (reproduce, diagnose, fix)       |                                                                                                                                                                                                              |
| `plan-feature`             | Orchestrates the test-first planning workflow for feature implementation in plan mode |                                                                                                                                                                                                              |
| `run-tests`                | Running Unity tests via the `run_unity_tests` tool                                    | JetBrains [MCP server](https://www.jetbrains.com/help/rider/mcp-server.html) and [MCP Server Extension for Unity](https://plugins.jetbrains.com/plugin/30357-mcp-server-extension-for-unity) plugin |
| `test-designing-guide`     | Test design methodology for deriving test cases from requirements                     |                                                                                                                                                                                                              |
| `test-writing-guide`       | Conventions for writing Unity Test Framework test code                                | [Test Helper](https://github.com/nowsprinting/test-helper) and [UI Test Helper](https://github.com/nowsprinting/test-helper.ui) package                                                                      |
| `unity-yaml-editing-guide` | Guidelines for directly hand-editing Unity YAML asset files                           |                                                                                                                                                                                                              |

## Included Agents

| Agent           | Description                                                                                                         |
|-----------------|---------------------------------------------------------------------------------------------------------------------|
| `test-designer` | Designs test cases during plan mode after class/method designs are produced, using the `test-designing-guide` skill |

## Installation

### User-scope installation

Add the marketplace and install the plugin:

```shell
/plugin marketplace add nowsprinting/unity-coding-skills
/plugin install unity-coding-skills@nowsprinting-unity-coding-skills
```

### Project-scope installation (team sharing)

Add the following to your project's `.claude/settings.json` and commit it to your repository:

```json
{
  "extraKnownMarketplaces": {
    "nowsprinting-unity-coding-skills": {
      "source": {
        "source": "github",
        "repo": "nowsprinting/unity-coding-skills"
      }
    }
  },
  "enabledPlugins": {
    "unity-coding-skills@nowsprinting-unity-coding-skills": true
  }
}
```

> [!TIP]  
> Recommend forking this repository and customizing it to suit your environment and project.

> [!NOTE]  
> When team members trust the project folder, Claude Code prompts them to install the marketplace and plugin automatically.

## Recommended Project Settings

### 1. MCP Server Configuration

The `run-tests` and `edit-scene` skills require JetBrains MCP servers.

1. Enable JetBrains built-in [MCP server](https://www.jetbrains.com/help/rider/mcp-server.html)
2. Install [MCP Server Extension for Unity](https://plugins.jetbrains.com/plugin/30357-mcp-server-extension-for-unity)
3. Add the following to your project `.mcp.json` or user MCP settings:

```json
{
  "mcpServers": {
    "jetbrains": {
      "type": "http",
      "url": "http://localhost:64342/stream"
    }
  }
}
```

> [!IMPORTANT]  
> Do not change the MCP server name `jetbrains`.

> [!TIP]  
> The JetBrains MCP server also provides tools useful for Coding Agents, such as `search_symbol` and `search_in_files_by_regex`.

### 2. Enforcing coding rules via `.editorconfig`

Any coding rules or Roslyn analyzer diagnostics you want Claude to respect should be set to `warning` or higher severity in `.editorconfig`.

For example, to prevent leaving unused code, add the following diagnostics:

```
resharper_unused_type_local_highlighting = warning
resharper_unused_type_global_highlighting = warning
resharper_unused_member_global_highlighting = warning
resharper_unused_member_local_highlighting = warning
```

## Usage

### Feature Implementation

Type in plan mode:

```bash
/plan-feature <SPEC>
```

### Fix bug

Type out of plan mode:

```bash
/fix-bug <INCIDENT>
```

## Contributing

Contributions are welcome. However, we will decline contributions that we cannot maintain — such as adding support for different coding agents or MCP servers. Please fork this repository and customize it for your needs instead.

## License

This project is released into the public domain under the [Unlicense](LICENSE).
