# Troubleshooting `u menu exec` (running editor scripts)

This skill runs editor scripts by attaching a `[MenuItem("Tools/...")]` to a `public static` parameterless method and invoking it with `u -i <instance> menu exec 'Tools/...'`. `<instance>` is the target Unity Editor; run `u instances` to list connected editors.

## The editor is not reachable

`u instances` shows no editor, or a command reports a connection error.

1. **No Unity Editor connected** — `u instances` prints an empty list. Open the Unity Editor with the target project, and make sure the Relay Server is running. Start it with `unity-relay --port 6500` (default port 6500; override with `UNITY_RELAY_PORT`).
2. **Wrong `-i <instance>`** — the value must match a `Project` name shown by `u instances` (a path, project name, or unique prefix also works). Re-check with `u instances`.
3. **Editor busy compiling / reloading** — if a refresh or domain reload is in flight, commands can fail or hang. Poll `u -i <instance> state --json` until `isCompiling` is `false`, then retry.

## The menu path is not found

`u menu exec 'Tools/...'` reports that the menu item does not exist.

1. **`[MenuItem]` is missing** — `u menu exec` can only run a registered menu item. Add `[MenuItem("Tools/YourAction")]` to the `public static` method (it must be parameterless), then refresh so Unity registers it.
2. **The script did not compile** — a compile error means the `[MenuItem]` was never registered. Run Quick Verify (`u -i <instance> console clear` → `u -i <instance> refresh` → poll `state --json` until `isCompiling` is false → `u -i <instance> console get -l E`) and fix every error first.
3. **Path spelling / casing** — menu paths are case-sensitive and must match exactly. List the actual registered paths with `u -i <instance> menu list -f <keyword>` and copy the path verbatim.
4. **`Assets/Editor/` placement** — editor-only attributes like `[MenuItem]` must live in an Editor assembly. Put the script under an `Editor/` folder (or an Editor `.asmdef`).

## The method ran but nothing happened (or it errored)

`u menu exec` reporting success only means the menu item was found and invoked — it does **not** mean the method completed without throwing.

1. **Always check the console after the call.** An exception thrown inside the method lands in the console, not in the `menu exec` result. Run `u -i <instance> console get -l E,X -s` (errors + exceptions, with stack traces) immediately after `menu exec`.
2. **Return values are not surfaced.** `u menu exec` does not return the method's return value. To observe a result, `Debug.Log(...)` it inside the method and read it back with `u -i <instance> console get -l L,W,E`.
3. **`async` methods are fire-and-forget.** An `async Task` / `async void` method is not awaited — side effects after the first `await` may not have happened when the call returns. If completion matters, block synchronously inside the method (e.g. `task.GetAwaiter().GetResult()`).
4. **The scene/asset was not saved.** Per the SKILL rules, an editor script must end with `EditorSceneManager.SaveScene` / `PrefabUtility.SaveAsPrefabAsset` (plus `AssetDatabase.SaveAssets()` for side-effect assets). If your change vanished, confirm the save calls ran (check the console, or `u -i <instance> state --json` → `sceneIsDirty`).

## After editing C# source

Always refresh and confirm compilation **before** running the script — a stale or failed compile means the old method (or no method) runs.

```bash
u -i <instance> console clear
u -i <instance> refresh
# poll until isCompiling == false (≈2 s interval, up to ~30 s)
u -i <instance> state --json
u -i <instance> console get -l E        # must be empty before menu exec
```

## ContextMenu methods (Component / ScriptableObject)

For a method exposed with `[ContextMenu("Name")]` on a Component or asset rather than `[MenuItem]`:

```bash
u -i <instance> menu context '<MethodName>' -t <target>   # target = hierarchy path or asset path
```

## Logs to investigate

- **Unity Console (primary)** — `u -i <instance> console get -s` (with stack traces); add `-l E,X` to filter errors and exceptions.
- **Unity Editor log** (editor-side crashes, compile errors, `Debug.Log` output) — `~/Library/Logs/Unity/Editor.log` (macOS), `%LOCALAPPDATA%\Unity\Editor\Editor.log` (Windows), `~/.config/unity3d/Editor.log` (Linux).
- **Relay Server output** — if commands never reach the editor, check the terminal running `unity-relay`.
