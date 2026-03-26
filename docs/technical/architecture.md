# Architecture

> Scope: this document is derived from the source code in this repository. It avoids design guesses and documents only what is directly observable in code.
>
> Primary runtime layers:
> - Host app: `excalidraw-app/` (product wrapper)
> - Editor component: `packages/excalidraw/` (published as `@excalidraw/excalidraw`)
> - Element model & scene/store primitives: `packages/element/` (published as `@excalidraw/element`)
> - Shared primitives & constants: `packages/common/` (`@excalidraw/common`)
> - Math helpers: `packages/math/` (`@excalidraw/math`)
> - Utility helpers (export, within-bounds, etc.): `packages/utils/` (`@excalidraw/utils`)

---

## High-level Architecture

At runtime the editable canvas experience is built around the `@excalidraw/excalidraw` React component (`packages/excalidraw/index.tsx`), which renders a stateful class component `App` (`packages/excalidraw/components/App.tsx`). The `App` owns:

- `state: AppState` (editor UI + interaction state)
- `scene: Scene` from `@excalidraw/element` (authoritative elements container)
- `actionManager: ActionManager` (action registry + execution)
- `store: Store` from `@excalidraw/element` (captures/commits changes into increments and feeds history)
- `history: History` (undo/redo built on Store deltas)
- three canvas render targets (static, interactive overlay, and a separate canvas for a “new element” preview)

The product wrapper app (`excalidraw-app/App.tsx`) embeds `<Excalidraw />` and wires persistence/collaboration via the `onChange` callback.

```mermaid
flowchart TB
  subgraph HostApp[excalidraw-app]
    HA[ExcalidrawWrapper\nexcalidraw-app/App.tsx]
    LS[LocalData.save(...)\nexcalidraw-app/data/LocalData]
    CO[Collab API\nexcalidraw-app/collab]
  end

  subgraph ExcalidrawPkg[@excalidraw/excalidraw\npackages/excalidraw]
    EX[<Excalidraw />\npackages/excalidraw/index.tsx]
    APP[App class\npackages/excalidraw/components/App.tsx]
    AM[ActionManager\npackages/excalidraw/actions/manager.tsx]
    RND[Renderer\npackages/excalidraw/scene/Renderer.ts]
    CAN[React canvas components\nStaticCanvas/InteractiveCanvas/NewElementCanvas]
    REN[Renderer functions\nstaticScene/interactiveScene/renderNewElementScene]
  end

  subgraph ElementPkg[@excalidraw/element\npackages/element]
    SC[Scene\npackages/element/src/Scene.ts]
    ST[Store + StoreSnapshot/Delta\npackages/element/src/store.ts]
    EL[Element model + helpers\nmutateElement/renderElement/...]
  end

  subgraph CommonMathUtils[Shared packages]
    CM[@excalidraw/common]
    MT[@excalidraw/math]
    UT[@excalidraw/utils]
  end

  HA -->|renders| EX --> APP
  APP --> AM
  APP --> SC
  APP --> ST
  APP --> RND

  APP -->|props| CAN -->|calls| REN
  REN --> EL

  APP -->|onChange| HA
  HA -->|persist| LS
  HA -->|sync| CO

  ExcalidrawPkg --> CM
  ExcalidrawPkg --> MT
  ExcalidrawPkg --> UT
  ElementPkg --> CM
  ElementPkg --> MT
```

Key code anchors:

- `<Excalidraw />` composition and API provider: `packages/excalidraw/index.tsx`
- Main editor component + ownership of state/scene/store/history/actions: `packages/excalidraw/components/App.tsx`
- Action dispatch: `packages/excalidraw/actions/manager.tsx`
- Scene container & mutation triggers: `packages/element/src/Scene.ts`
- Store commit/increment/delta pipeline: `packages/element/src/store.ts`
- Canvas rendering entry points:
  - `packages/excalidraw/renderer/staticScene.ts`
  - `packages/excalidraw/renderer/interactiveScene.ts`
  - `packages/excalidraw/renderer/renderNewElementScene.ts`
  - Components: `packages/excalidraw/components/canvases/*.tsx`

---

## Data Flow: як дані рухаються через систему

### 1) Вхідні дані (initial load)

In the host app (`excalidraw-app/App.tsx`) `initializeScene()` constructs initial `scene` data from local storage or external sources, restoring elements and appState:

- `restoreElements(...)` and `restoreAppState(...)` are imported from `@excalidraw/excalidraw/data/restore`.
- In collaboration cases, elements are reconciled via `reconcileElements(...)`.

The host passes `initialData={...}` into `<Excalidraw />` (`excalidraw-app/App.tsx`), and later uses `excalidrawAPI.updateScene(...)` to apply new scene payloads (e.g. on hash changes).

### 2) User interaction → editor update

Inside the editor (`packages/excalidraw/components/App.tsx`), state changes and element changes converge into two “authoritative” containers:

- Elements live in `this.scene` (an instance of `Scene` from `@excalidraw/element`).
- App UI/interaction state lives in `this.state` (the `AppState` interface from `packages/excalidraw/types.ts`).

The primary mutation paths visible in code:

- **Actions**: `ActionManager.executeAction()` calls `action.perform(elements, appState, value, app)` and forwards the result into an updater (`App.syncActionResult`).
  - Action result shape is `ActionResult` (`packages/excalidraw/actions/types.ts`) and may contain:
    - `elements?: readonly ExcalidrawElement[] | null`
    - `appState?: Partial<AppState> | null`
    - `files?: BinaryFiles | null`
    - `captureUpdate: CaptureUpdateActionType`

- **Direct element mutation**: `App.mutateElement(...)` delegates to `this.scene.mutateElement(...)` (`packages/excalidraw/components/App.tsx` → `packages/element/src/Scene.ts`).
  - `Scene.mutateElement` calls the element-level `mutateElement(...)` helper and triggers `scene.triggerUpdate()` if the element version changed and mutation is meant to inform.

### 3) App-level reconciliation step (syncActionResult)

`App.syncActionResult(...)` (batched) applies the action’s result:

- If `actionResult.elements` is present, it calls `this.scene.replaceAllElements(actionResult.elements)`.
- If `actionResult.appState` is present, it merges into React state via `this.setState(prev => ({...prev, ...actionAppState, ...}))`.
- Files are added via `this.addMissingFiles(...)` etc.
- If neither appState nor elements were updated, it still triggers `this.scene.triggerUpdate()` as a fallback.

This function is the central “apply” point for ActionManager results.

### 4) Commit + external notifications

On every React update cycle, `App.componentDidUpdate(...)` commits editor state into the Store and triggers host callbacks:

- `this.store.commit(elementsMap, this.state)` (commit into `@excalidraw/element` Store)
- if `!this.state.isLoading`, it calls both:
  - `this.props.onChange?.(elements, this.state, this.files)`
  - `this.onChangeEmitter.trigger(elements, this.state, this.files)`

So, **data “leaves” the editor through `onChange`** (and also through `store.onStoreIncrementEmitter` for increment-based consumers).

### 5) Host persistence/collab

The host app’s `onChange` handler (`excalidraw-app/App.tsx`) uses the emitted `elements/appState/files`:

- If collaborating: `collabAPI.syncElements(elements)`.
- If saving isn’t paused: `LocalData.save(elements, appState, files, callback)`.
- The callback may update element statuses and push them back into the editor using `excalidrawAPI.updateScene({ elements, captureUpdate: CaptureUpdateAction.NEVER })`.

This is a concrete feedback loop: host-side persistence can cause a follow-up editor update.

---

## State Management: детальний опис (appState, elements, actionManager)

This project uses multiple “state carriers” at once:

- React component state (`AppState`) inside `packages/excalidraw/components/App.tsx`
- Scene container (`Scene`) for elements inside `@excalidraw/element`
- A Store (`Store`) that snapshots observed state, emits increments, and powers undo/redo
- An Action system (`ActionManager`) that orchestrates updates as `ActionResult`
- Additionally in `packages/excalidraw` there is a Jotai store for editor UI atoms (`editorJotaiStore` in `packages/excalidraw/index.tsx`), and the host app also uses Jotai (`excalidraw-app/app-jotai`).

This section focuses on the requested three: `appState`, `elements`, `actionManager`.

### appState (React state)

- Type: `AppState` interface in `packages/excalidraw/types.ts`.
- Defaults: `getDefaultAppState()` in `packages/excalidraw/appState.ts` returns an object with most keys except `offsetTop/offsetLeft/width/height` (those are injected by `App` constructor).

`AppState` contains both:

- **UI & modal state** (`openDialog`, `openMenu`, `openSidebar`, `toast`, `contextMenu`, etc.)
- **Interaction state** (`activeTool`, `newElement`, `selectionElement`, `selectedElementIds`, `hoveredElementIds`, `editingTextElement`, etc.)
- **Viewport state** (`scrollX`, `scrollY`, `zoom`, `offsetTop/offsetLeft`, `width/height`)
- **Collaboration pointers** (`collaborators: Map<SocketId, Collaborator>`, follow mode state, etc.)

Concrete behaviors visible in code that define how `AppState` is managed:

1) `syncActionResult` merges `actionResult.appState` into `this.state` via `setState`.

2) `componentDidUpdate` calls `this.store.commit(elementsMap, this.state)` and then (if not loading) triggers `onChange` to the host.

There is also storage filtering logic:

- `APP_STATE_STORAGE_CONF` in `packages/excalidraw/appState.ts` declares per-key “keep” flags for `browser`, `export`, and `server` storage.
- Helper functions exist:
  - `clearAppStateForLocalStorage(appState)`
  - `cleanAppStateForExport(appState)`
  - `clearAppStateForDatabase(appState)`

This is a concrete mechanism for deciding which parts of `AppState` are exportable/persistable.

### elements (Scene-owned element list)

Elements are Excalidraw element records defined in `@excalidraw/element` (e.g. `packages/element/src/types.ts`). Each element has:

- identity and geometry: `id`, `x`, `y`, `width`, `height`, `angle`
- style props: `strokeColor`, `backgroundColor`, `fillStyle`, `roughness`, etc.
- versioning fields used for reconciliation/order/history:
  - `version` / `versionNonce`
  - `index: FractionalIndex | null` (ordering)
- grouping and binding fields: `groupIds`, `frameId`, `boundElements`, `containerId` (type-specific)
- lifecycle: `isDeleted`

The authoritative container for elements inside `App` is `this.scene` (a `Scene` instance from `packages/element/src/Scene.ts`).

Key `Scene` facts observable in code:

- It maintains multiple representations:
  - `elements` and `elementsMap` (including deleted)
  - `nonDeletedElements` and `nonDeletedElementsMap`
  - `frames` and `nonDeletedFramesLikes`
- `replaceAllElements(nextElements)`:
  - converts map/array to array (`toArray`)
  - validates fractional indices (throttled) unless `skipValidation`
  - normalizes indices with `syncInvalidIndices(...)`
  - rebuilds maps and derived arrays
  - triggers `triggerUpdate()` which regenerates a `sceneNonce` and calls subscribers
- `mutateElement(element, updates, options)`:
  - delegates to element-level `mutateElement(...)`
  - triggers update only if:
    - element exists in scene map
    - version changed
    - `informMutation` is true

That means: element changes are tracked by the `Scene` object which has its own update callbacks.

### actionManager (action registry & execution)

The action system is centered on `ActionManager` (`packages/excalidraw/actions/manager.tsx`). Observable behavior:

- Holds `actions: Record<ActionName, Action>`.
- `registerAll(actions)` and `registerAction(action)` populate that map.
- `executeAction(action, source, value)`:
  - reads `elements` via `getElementsIncludingDeleted()`
  - reads `appState` via `getAppState()`
  - calls `action.perform(elements, appState, value, app)`
  - forwards the result into `updater(...)`
- `handleKeyDown(event)`:
  - filters actions based on host UIOptions (canvasActions), keyTest, and viewMode restrictions
  - calls `perform(...)` and updates via `updater(...)`

In `App` constructor (`packages/excalidraw/components/App.tsx`) the ActionManager is created as:

- `new ActionManager(this.syncActionResult, () => this.state, () => this.scene.getElementsIncludingDeleted(), this)`

So `ActionManager` is wired directly to:

- **read**: current React `AppState` and current `Scene` elements
- **write**: `syncActionResult` (which updates both Scene and AppState)

---

## Rendering Pipeline: від React component до canvas

Rendering is split into three canvases rendered by separate React components inside `App.render()`:

- `StaticCanvas` (main scene)
- `NewElementCanvas` (preview layer when `appState.newElement` exists)
- `InteractiveCanvas` (overlay for selection, handles, remote cursors, scrollbars, etc.)

`App.render()` passes:

- the canvas element(s)
- `elementsMap` / `allElementsMap` (maps of renderable/non-deleted vs all)
- `visibleElements`
- `appState` (full state, but memoized canvases select “relevant props”)
- `renderConfig` objects (image cache, theme, grid settings, etc.)

### StaticCanvas → renderStaticScene

Component: `packages/excalidraw/components/canvases/StaticCanvas.tsx`.

- Ensures canvas pixel size tracks `appState.width/height` multiplied by `scale` (`devicePixelRatio`).
- On effect, calls `renderStaticScene({ canvas, rc, scale, elementsMap, allElementsMap, visibleElements, appState, renderConfig }, throttleFlag)`.
- `StaticCanvas` is memoized with a custom comparator:
  - invalidates on `sceneNonce`, `scale`, `elementsMap`, `visibleElements`
  - otherwise shallow-compares a selected subset of AppState plus `renderConfig`

Renderer: `packages/excalidraw/renderer/staticScene.ts`.

Observable steps inside `renderStaticScene` / `_renderStaticScene`:

- Determine normalized canvas dimensions via `getNormalizedCanvasDimensions(canvas, scale)`.
- `bootstrapCanvas({ canvas, scale, normalizedWidth, normalizedHeight, theme, isExporting, viewBackgroundColor })` to get a 2D context.
- Apply zoom: `context.scale(appState.zoom.value, appState.zoom.value)`.
- Optionally stroke grid using `gridSize`, `gridStep`, `scrollX/scrollY`, zoom.
- For each visible element, call `renderElement(element, elementsMap, allElementsMap, rc, context, renderConfig, appState)`.
  - If frame clipping is enabled, it may call `frameClip(...)` before rendering.
  - If the element has bound text, it renders that bound text element too.
- For non-export rendering, it may render a link icon overlay for elements with `link`.

### NewElementCanvas → renderNewElementScene

Component: `packages/excalidraw/components/canvases/NewElementCanvas.tsx`.

- Owns its own `<canvas>`.
- Each effect calls `renderNewElementScene({ canvas, scale, newElement: appState.newElement, elementsMap, allElementsMap, rc, renderConfig, appState }, throttleFlag)`.

Renderer: `packages/excalidraw/renderer/renderNewElementScene.ts`.

Observable steps:

- Normalize dimensions, call `bootstrapCanvas`.
- Apply zoom.
- If `newElement` exists and is not selection:
  - skip rendering if `isInvisiblySmallElement(newElement)`.
  - optionally apply frame clip (if enabled).
  - call `renderElement(newElement, ...)`.
- Else clear rect.

This layer exists specifically to draw the in-progress element separate from the main static scene.

### InteractiveCanvas → renderInteractiveScene (+ animation loop)

Component: `packages/excalidraw/components/canvases/InteractiveCanvas.tsx`.

- Computes remote pointer viewport coordinates and selection state from `appState.collaborators`.
- Builds `rendererParams.current` including:
  - `renderConfig` with selection color, remote pointers, and scrollbar rendering options
  - `editorInterface` and a callback `renderInteractiveSceneCallback`
  - `animationState` and `deltaTime`
- Starts a render loop via `AnimationController.start(INTERACTIVE_SCENE_ANIMATION_KEY, ...)`.
  - Each tick calls `renderInteractiveScene({ ...rendererParams.current, deltaTime, animationState })`.
  - The returned `animationState` can be persisted across frames.

Renderer: `packages/excalidraw/renderer/interactiveScene.ts`.

Observable early steps (and consistent with the static pipeline):

- Normalize dimensions and `bootstrapCanvas({ canvas, scale, normalizedWidth, normalizedHeight })`.
- Apply zoom: `context.scale(appState.zoom.value, appState.zoom.value)`.
- Draw interaction overlays (selection element, linear editor handles, snaps, scrollbars, remote cursors, etc.; helpers are imported from `@excalidraw/element` and other internal modules).

### Visibility culling / elementsMap production

`Renderer` class (`packages/excalidraw/scene/Renderer.ts`) builds two things used by the canvases:

- `elementsMap` (renderable elements map) that excludes:
  - the `newElement` by id
  - the `editingTextElement` while it’s being edited (text is rendered separately)
- `visibleElements` computed by scanning `elementsMap.values()` and keeping only elements that are in the viewport via `isElementInViewport(...)`.

This function is memoized and keyed by viewport parameters (`zoom`, offsets, scroll, width/height), and also by `sceneNonce` (a cache invalidation nonce provided by `Scene` on update).

---

## Package Dependencies: взаємозв'язки між packages

This repository is a Yarn workspaces monorepo (`package.json` at repo root lists workspaces `excalidraw-app`, `packages/*`, `examples/*`).

### Workspace packages (core)

- `@excalidraw/common` (`packages/common/package.json`)
  - depends on: `tinycolor2`

- `@excalidraw/math` (`packages/math/package.json`)
  - depends on: `@excalidraw/common`

- `@excalidraw/element` (`packages/element/package.json`)
  - depends on: `@excalidraw/common`, `@excalidraw/math`

- `@excalidraw/utils` (`packages/utils/package.json`)
  - depends on external libs (e.g. `roughjs`, `pako`, `browser-fs-access`, etc.)

- `@excalidraw/excalidraw` (`packages/excalidraw/package.json`)
  - depends on: `@excalidraw/common`, `@excalidraw/element`, `@excalidraw/math`, and many UI/runtime deps
  - peer deps: `react`, `react-dom`

- `excalidraw-app` (`excalidraw-app/package.json`)
  - consumes `@excalidraw/excalidraw` (imports from `@excalidraw/excalidraw` in `excalidraw-app/App.tsx`)

### Dependency graph (workspace-level)

```mermaid
flowchart LR
  Common[@excalidraw/common]
  Math[@excalidraw/math]
  Element[@excalidraw/element]
  Utils[@excalidraw/utils]
  Excalidraw[@excalidraw/excalidraw]
  App[excalidraw-app]

  Math --> Common
  Element --> Common
  Element --> Math

  Excalidraw --> Common
  Excalidraw --> Math
  Excalidraw --> Element
  Excalidraw --> Utils

  App --> Excalidraw
  App --> Common
  App --> Element
```

Notes grounded in code:

- `packages/excalidraw` imports heavily from `@excalidraw/element` for element type checks, geometry, selection helpers, and `renderElement(...)`.
  - Example: `packages/excalidraw/renderer/staticScene.ts` calls `renderElement` from `@excalidraw/element`.
- `packages/excalidraw/renderer/interactiveScene.ts` imports math primitives from `@excalidraw/math` and many helpers from `@excalidraw/element`.
- `packages/excalidraw/index.tsx` re-exports multiple APIs from `@excalidraw/common`, `@excalidraw/element`, and `@excalidraw/utils`.
- The host app imports from both `@excalidraw/excalidraw` and `@excalidraw/common`, and also directly uses `@excalidraw/element` helpers (`isElementLink`, `newElementWith`, etc.) in `excalidraw-app/App.tsx`.

---

## Appendix: Concrete “facts-only” mapping of requested terms

### appState

- Type: `packages/excalidraw/types.ts` → `export interface AppState { ... }`.
- Defaults: `packages/excalidraw/appState.ts` → `getDefaultAppState()`.
- Commit point: `packages/excalidraw/components/App.tsx` → `this.store.commit(elementsMap, this.state)`.

### elements

- Model: `packages/element/src/types.ts`.
- Container: `packages/element/src/Scene.ts`.
- Scene update trigger: `Scene.triggerUpdate()` generates `sceneNonce` and calls update callbacks.

### actionManager

- Class: `packages/excalidraw/actions/manager.tsx`.
- Execute path: `executeAction()` → `action.perform(...)` → `App.syncActionResult(...)`.
- Keyboard shortcut path: `handleKeyDown()` filters actions by `UIOptions.canvasActions`, `keyTest`, and `viewModeEnabled`.

