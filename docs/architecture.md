# AnimAide C4 Architecture

## Purpose & Scope
AnimAide is a Blender add-on that augments keyframe editing with reusable tools for curve shaping, key management, and non-destructive animation offsets. This document captures the system-level structure using the C4 model so designers and maintainers can reason about integration points, module responsibilities, and runtime behavior.


## Codebase Landmarks

- `scripts/addons/animaide/__init__.py` is the activation entry point; it registers property groups, operators, menus, and Blender event handlers.
- `scripts/addons/animaide/curve_tools` provides modal operators and UI for sculpting f-curves, along with the supporting math and state containers.
- `scripts/addons/animaide/anim_offset` implements the timeline mask workflow and modal handlers that react to user transforms in real time.
- `scripts/addons/animaide/key_manager` centralises key classification, handle adjustments, and frame spacing utilities.
- `scripts/addons/animaide/prefe.py` exposes add-on preferences that dynamically register header/panel variants across Blender editors.
- `scripts/addons/animaide/utils` supplies shared helpers for curve cloning, key selection, UI feedback, and theme manipulation.

## Level 1 – System Context

The add-on executes inside Blender and is operated directly by animators. It relies on Blender's Python API to reach scene data and to receive timeline/handler callbacks.

```mermaid
C4Context
  title System Context – AnimAide Add-on
  Person(animator, "Animator", "Works inside Blender to sculpt animation curves.")
  System_Boundary(host, "Blender Add-on Process") {
    System(animaide, "AnimAide Add-on", "Python operators and UI for precise keyframe manipulation.")
  }
  System_Ext(blenderCore, "Blender Core", "Provides UI, animation playback, and bpy API.")
  System_Ext(assetRepo, "AnimAide Online Docs", "Reference documentation and issue tracker.")
  Rel(animator, animaide, "Invokes tools via menus, panels, and shortcuts")
  Rel(animaide, blenderCore, "Reads/writes f-curves, timeline masks, UI state", "bpy API")
  Rel(blenderCore, animator, "Displays feedback, applies transforms")
  Rel(animator, assetRepo, "Consults", "Web")
  Rel(blenderCore, animaide, "Triggers handlers for scene updates")
```

## Level 2 – Container View

AnimAide is delivered as a single Python package that is decomposed into specialised sub-packages. Most containers collaborate through the shared utilities layer and communicate with Blender through the `bpy` API.

```mermaid
C4Container
  title Container View – AnimAide Add-on
  Person(animator, "Animator", "Operates AnimAide features inside Blender.")
  Container_Boundary(animaideSystem, "AnimAide Add-on (Python)") {
    Container(entry, "__init__.py", "Python module", "Registers classes, handlers, and preferences.")
    Container(curveTools, "curve_tools package", "Python package", "Modal operators for shaping F-curves.")
    Container(animOffset, "anim_offset package", "Python package", "Timeline mask and offset modal workflow.")
    Container(keyManager, "key_manager package", "Python package", "Classification, selection, and handle controls.")
    Container(utils, "utils package", "Python package", "Shared helpers for curves, keys, and UI feedback.")
    Container(prefModule, "prefe.py", "Python module", "Add-on preferences and dynamic UI registration.")
    Container(uiModule, "ui.py", "Python module", "Menus bridging containers into Blender editors.")
  }
  Container_Ext(bpyApi, "Blender Python API (bpy)", "C++ core + Python bindings", "Hosts runtime state and exposes animation data.")
  ContainerDb_Ext(sceneData, "Blender Scene Data", "Keyframes, actions, timeline ranges.")
  Rel(animator, entry, "Enables and configures the add-on")
  Rel(entry, curveTools, "Registers property groups and operators")
  Rel(entry, animOffset, "Registers operators and handlers")
  Rel(entry, keyManager, "Registers operators and property groups")
  Rel(entry, uiModule, "Adds menus to Blender editors")
  Rel(entry, prefModule, "Creates user-configurable options")
  Rel(curveTools, utils, "Uses shared curve/key helpers")
  Rel(animOffset, utils, "Uses markers, math, and key helpers")
  Rel(animOffset, prefModule, "Consults preferences before adding panels")
  Rel(keyManager, utils, "Uses shared polling and key helpers")
  Rel(uiModule, curveTools, "Embeds operator menus")
  Rel(prefModule, animOffset, "Registers/unregisters UI extensions")
  Rel(prefModule, keyManager, "Registers/unregisters UI extensions")
  Rel(curveTools, bpyApi, "Evaluates and edits F-curves", "bpy.data/actions")
  Rel(animOffset, bpyApi, "Manipulates timeline ranges and handlers")
  Rel(keyManager, bpyApi, "Sets handle types, key metadata")
  Rel(utils, bpyApi, "Accesses themes, markers, drawing callbacks")
  Rel(bpyApi, sceneData, "Persists actions and timeline state")
```

## Level 3 – Component View (curve_tools)

The `curve_tools` container contributes the densest runtime surface. It owns the modal operators that edit values in place, backed by property groups and support utilities that maintain global caches and helper curves.

```mermaid
C4Component
  title Component View – curve_tools package
  Container_Boundary(curveTools, "curve_tools") {
    Component(props, "props.py", "PropertyGroup definitions", "Stores tool state, bookmarks, and clone options.")
    Component(ops, "ops.py", "Operator classes", "Modal tools for easing, blending, scaling, smoothing, and noise.")
    Component(support, "support.py", "Support functions", "Caches global context and provides math helpers.")
    Component(uiComp, "ui.py", "UI bindings", "Panels and menus exposing operators across editors.")
  }
  Component_Ext(utilsKey, "utils.key", "Shared helper", "Key selection, insertion, and neighbor lookups.")
  Component_Ext(utilsCurve, "utils.curve", "Shared helper", "Curve duplication, cloning, and helper action management.")
  Component_Ext(prefModule, "prefe.Preferences", "Shared component", "User preferences that toggle marker usage and modal behavior.")
  Component_Ext(blenderApi, "bpy", "External API", "Provides scene data, UI drawing, and operator registration.")
  Rel(props, support, "Triggers cache refresh and marker toggles")
  Rel(ops, props, "Reads tool settings and writes factors")
  Rel(ops, support, "Invokes curve math and global caching")
  Rel(ops, utilsKey, "Queries selection and frame neighbors")
  Rel(ops, utilsCurve, "Creates helper curves and clones")
  Rel(support, utilsKey, "Builds global key metadata")
  Rel(support, utilsCurve, "Builds helper actions for shear/noise")
  Rel(uiComp, ops, "Binds operators to menus and panels")
  Rel(props, prefModule, "Applies preference-driven defaults")
  Rel(ops, blenderApi, "Evaluates f-curves and updates keyframes")
  Rel(uiComp, blenderApi, "Draws UI elements in Graph/Dopesheet/3D View")
  Rel(support, blenderApi, "Accesses markers, actions, and scene state")
```

## Runtime Interaction Highlights

- `scripts/addons/animaide/__init__.py::register` attaches `load_post_handler` and installs menu entries so the add-on can react when a scene is loaded.
- `anim_offset/ops.py::ANIMAIDE_OT_add_anim_offset_mask.modal` continuously updates the timeline mask while the user drags, relying on `anim_offset.support.set_blend_values` to sync the helper curve.
- `curve_tools/ops.py::ANIMAIDE_OT_ease_to_ease.execute` delegates to `curve_tools.support.to_execute`, which orchestrates per-object caching and invokes the tool callback for every affected f-curve.
- `key_manager/ops.py::AAT_OT_set_handles_type.execute` feeds through `key_manager.support.set_handles_type`, ensuring handle adjustments obey the active UI context and selection filters.

## Design Considerations & Risks

- Global caches in `curve_tools.support.global_values` and handler-installed state in `anim_offset.support` demand careful lifecycle management to avoid stale references when Blender scenes change unexpectedly.
- UI placement is configurable at runtime through `prefe.py`; new panels or menus must register and unregister conditionally, otherwise Blender can throw duplicate class registration errors.
- Modal operators such as `ANIMAIDE_OT_add_anim_offset_mask` assume the host area (Graph Editor, Dope Sheet, or 3D View) remains valid. Additional guards may be required when Blender switches contexts mid-operation.
- Utilities under `utils.general` temporarily modify theme colors and UI handlers; extensions should restore defaults through `utils.reset_bar_color` to avoid leaking UI state across sessions.
