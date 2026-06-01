---
name: edit-scene
description: >-
  Creates and modifies Unity scene and prefab files. Use this skill whenever
  creating, editing, or modifying .unity scene files or .prefab prefab files,
  or writing editor scripts under Assets/Editor/ that generate or manipulate
  scenes, prefabs, or scene-bound assets. This includes adding GameObjects,
  building uGUI hierarchies, wiring up components, and any task that results
  in changes to .unity or .prefab files.
context: fork
license: Unlicense
metadata:
  author: Koji Hasegawa
---

Guide for creating and editing Unity scene files in Unity projects.

## Rules

- Do not directly Edit or Write `.unity` or `.prefab` files. Instead, write an editor script under `Assets/Editor/` and execute it in Unity to create or update the scene or prefab — those carry GameObject/Prefab-instance structure that is unsafe to author by hand.
  - For simple, one-off changes (create a GameObject, add or set a component, save the scene), the `u gameobject` / `u component` / `u scene` / `u asset` commands drive the editor directly with no script. Reach for an editor script when the change is structural or repetitive — uGUI hierarchies, prefab variants, bulk wiring.
- **Editor scripts must always end with `EditorSceneManager.SaveScene` (or `PrefabUtility.SaveAsPrefabAsset` for prefabs).** Treat "no dirty scenes/assets at script exit" as a hard postcondition.
  - Scene: `EditorSceneManager.SaveScene(scene, path)` (new) or `EditorSceneManager.SaveScene(scene)` (existing). The return value is `true` on success.
  - Prefab: `PrefabUtility.SaveAsPrefabAsset(go, path)`. When editing via `LoadPrefabContents`, always pair with `SaveAsPrefabAsset` → `UnloadPrefabContents`.
  - When the script also creates side-effect assets (Materials, ScriptableObjects, etc.), call `AssetDatabase.SaveAssets()` after the per-object saves to flush pending writes.
- Before running an editor script, check whether the editor is in Play Mode (`u -i <instance> state --json` → `isPlaying`); if so, exit it with `u -i <instance> stop` first — Play Mode may skip recompilation, leaving stale code active. (`<instance>` is the target Unity Editor; run `u instances` to list connected editors.)
- After modifying code, confirm compilation succeeds before running the script. Run the **Quick Verify** sequence: `u -i <instance> console clear` → `u -i <instance> refresh` → poll `u -i <instance> state --json` until `isCompiling` is `false` (≈2 s interval, up to ~30 s) → `u -i <instance> console get -l E,W`. Any error line is a blocker; fix it before running.
- **Run the editor script via `[MenuItem]` + `u menu exec`.** Add a `[MenuItem("Tools/...")]` attribute to a `public static` parameterless method, then invoke it with `u -i <instance> menu exec 'Tools/...'`. The method's return value is not surfaced — emit results with `Debug.Log(...)` and read them back with `u -i <instance> console get -l L,W,E` after the call. For a method already exposed as `[ContextMenu]` on a Component/ScriptableObject, use `u -i <instance> menu context '<MethodName>' -t <target>` instead. (`u api call` cannot run project-defined types — it is restricted to the `UnityEngine`/`UnityEditor` namespaces.)
- For uGUI buttons and text, use the **legacy variants** (`UnityEngine.UI.Button` / `UnityEngine.UI.Text`). Do not use TextMeshPro unless the user explicitly requests it.
- Apply context-menu-equivalent defaults when creating uGUI components (see Resources below).

## Gotchas

- **Serialize editor commands around compilation and domain reloads.** A `refresh`, `menu exec`, or `tests run` can trigger script recompilation / domain reload; issuing the next command mid-reload fails or returns stale state. Wait for `isCompiling` (and, after `play`, `isPlaying`) to settle via `u -i <instance> state --json` before issuing the next command — do not fire commands back-to-back.
- **On a failed or empty-looking result, re-check state before retrying.** If a command errors or `console get` is unexpectedly empty, poll `u -i <instance> state --json` until `isCompiling` is `false`, then retry once. If it fails again, stop and consult the user instead of looping.
- **`.meta` files follow an asymmetric commit rule.** Never create them manually — Unity generates them automatically. Scene/prefab files (`.unity`, `.prefab`) and their referenced assets (materials, SOs, etc.) must be **committed** (required for GUID resolution); editor scripts (`Assets/Editor/*.cs`) and their `.meta` must **not** be committed — orphaned metas break the other checkout if the script is absent. Do not delete them; leave them for the user to remove manually. Do not add them to `.gitignore`; the user excludes them at commit time.

## Scene lifecycle

- **New scene**: First determine whether the scene will be loaded additively or as a single scene, then choose the setup accordingly.
  - **Additive** (`LoadSceneMode.Additive`): `EditorSceneManager.NewScene(NewSceneSetup.EmptyScene, NewSceneMode.Single)` — no camera or light needed.
  - **Single** (`LoadSceneMode.Single`): `EditorSceneManager.NewScene(NewSceneSetup.DefaultGameObjects, NewSceneMode.Single)` — Main Camera and Directional Light are included automatically; do not add a camera manually.
  - In both cases, place additional GameObjects via `ObjectFactory.CreateGameObject` and save with `EditorSceneManager.SaveScene(scene, "Assets/YourFeature/Scenes/XxxScene.unity")`.
- **Edit existing scene**: `EditorSceneManager.OpenScene(path, OpenSceneMode.Single)` → make changes → `EditorSceneManager.SaveScene(scene)`.
- **New prefab**: build the GameObject hierarchy in memory, then save with `PrefabUtility.SaveAsPrefabAsset(go, "Assets/YourFeature/Prefabs/XxxPrefab.prefab")`.
- **Edit existing prefab**: open with `PrefabUtility.LoadPrefabContents(path)` → modify → `PrefabUtility.SaveAsPrefabAsset(root, path)` → `PrefabUtility.UnloadPrefabContents(root)`.
- Use `ObjectFactory.CreateGameObject` / `ObjectFactory.AddComponent` so Undo history and Presets are applied automatically.
- Parent child objects with `transform.SetParent(parent, worldPositionStays: false)`.

## Resources

- Before writing or modifying any editor script that creates or manipulates uGUI components: Read `${CLAUDE_SKILL_DIR}/resources/ugui.md`

## Troubleshooting

- `u menu exec` fails, the menu path is not found, or the editor is not reachable: Read `${CLAUDE_SKILL_DIR}/resources/troubleshooting-run-editor-script.md`
