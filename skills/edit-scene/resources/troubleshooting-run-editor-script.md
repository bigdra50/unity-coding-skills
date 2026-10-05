# Troubleshooting `unity command menu` (running editor scripts)

This skill runs editor scripts by attaching a `[MenuItem("Tools/...")]` to a `public static` parameterless method and invoking it with `unity command menu --path 'Tools/...'`. `unity command` reaches the Editor that has the project open through the `com.unity.pipeline` package; the project is detected from the working directory (or pass `--project-path <path>`).

## The editor is not reachable

`unity command editor_status` reports a connection error or no answer.

1. **No Unity Editor running** — open the Unity Editor with the target project. `unity command editor_status` should answer with `"status": "ready"`.
2. **Pipeline package missing** — `unity command` cannot reach an Editor without `com.unity.pipeline`. Install it once with `unity pipeline install`, then let the Editor import it.
3. **Wrong project** — when running outside the project directory, pass `--project-path <path>` so the command reaches the right Editor.
4. **Editor busy compiling / reloading** — if a recompile or domain reload is in flight, commands can fail or hang. Poll `unity command editor_status` until `compiling` and `domainReloadInProgress` are both `false`, then retry.

## The menu path is not found

`unity command menu --path 'Tools/...'` reports that the menu item does not exist.

1. **`[MenuItem]` is missing** — `menu` can only run a registered menu item. Add `[MenuItem("Tools/YourAction")]` to the `public static` method (it must be parameterless), then recompile so Unity registers it.
2. **The script did not compile** — a compile error means the `[MenuItem]` was never registered. Run Quick Verify (`unity command clear_console` → `unity command recompile` → poll `unity command recompile_status` until `status` is `completed` or `up_to_date` → `unity command console --level error`) and fix every error first.
3. **Path spelling / casing** — menu paths are case-sensitive and must match exactly. List the registered paths with `unity command menu` (no `--path`) and copy the path verbatim.
4. **`Assets/Editor/` placement** — editor-only attributes like `[MenuItem]` must live in an Editor assembly. Put the script under an `Editor/` folder (or an Editor `.asmdef`).

## The method ran but nothing happened (or it errored)

`menu` reporting success only means the menu item was found and invoked — it does **not** mean the method completed without throwing.

1. **Always check the console after the call.** An exception thrown inside the method lands in the console, not in the `menu` result. Run `unity command console --level error` (entries include stack traces) immediately after `menu`.
2. **Return values are not surfaced.** `menu` does not return the method's return value. To observe a result, `Debug.Log(...)` it inside the method and read it back with `unity command console`.
3. **`async` methods are fire-and-forget.** An `async Task` / `async void` method is not awaited — side effects after the first `await` may not have happened when the call returns. If completion matters, block synchronously inside the method (e.g. `task.GetAwaiter().GetResult()`).
4. **The scene/asset was not saved.** Per the SKILL rules, an editor script must end with `EditorSceneManager.SaveScene` / `PrefabUtility.SaveAsPrefabAsset` (plus `AssetDatabase.SaveAssets()` for side-effect assets). If your change vanished, confirm the save calls ran (check the console, or `unity command list_open_scenes` → `isDirty`).

## After editing C# source

Always recompile and confirm compilation **before** running the script — a stale or failed compile means the old method (or no method) runs.

```bash
unity command clear_console
unity command recompile
# poll until status is completed or up_to_date (≈2 s interval, up to ~30 s)
unity command recompile_status
unity command console --level error     # must be empty before menu
```

## ContextMenu methods (Component / ScriptableObject)

There is no command that invokes a `[ContextMenu("Name")]` method directly. Add a `[MenuItem]` wrapper in an Editor script that locates the target (e.g. `Object.FindFirstObjectByType<T>()` or `AssetDatabase.LoadAssetAtPath<T>(path)`) and calls the method, then run the wrapper with `unity command menu --path`.

## Logs to investigate

- **Unity Console (primary)** — `unity command console` (entries include stack traces; `--level error` keeps errors and exceptions).
- **Unity Editor log** (editor-side crashes, compile errors, `Debug.Log` output) — `~/Library/Logs/Unity/Editor.log` (macOS), `%LOCALAPPDATA%\Unity\Editor\Editor.log` (Windows), `~/.config/unity3d/Editor.log` (Linux).
