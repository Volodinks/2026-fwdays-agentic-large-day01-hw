# Domain glossary (Excalidraw)

Це словник **доменних термінів**, які мають конкретне значення в цьому репозиторії (monorepo Excalidraw). Назви термінів подані **англійською** — як у коді.

---

## Element

- **Name (as in code)**: `Element` (зазвичай мається на увазі *Excalidraw element record*)
- **Definition (project-specific)**: “Елемент” на полотні — одна фігура/об’єкт сцени (rectangle/ellipse/text/arrow/image/frame/…); у коді представлений як JSON‑серіалізований запис, який можна зберігати/відновлювати/синхронізувати між клієнтами.
- **Where used (key files)**:
  - `packages/element/src/types.ts` — базова структура елемента, `ExcalidrawElement` та його підтипи
  - `packages/element/src/Scene.ts` — зберігання та оновлення списку елементів
  - `packages/excalidraw/components/App.tsx` — створення/мутація елементів через `newElement*` / `scene.mutateElement`
- **NOT to confuse with**:
  - DOM `Element` (Web API) — HTML/SVG вузол; у проєкті “Element” майже завжди про **дані сцени**, а не про DOM.
  - “UI element” (кнопка/панель) — компоненти React не є “elements” сцени.

---

## ExcalidrawElement

- **Name (as in code)**: `ExcalidrawElement`
- **Definition (project-specific)**: Union‑тип усіх можливих “сценових” елементів Excalidraw. Важлива властивість — **JSON serializable** (призначений для збереження/експорту/колаборації) і не має містити peer‑local computed state.
- **Where used (key files)**:
  - `packages/element/src/types.ts` — визначення `ExcalidrawElement` і підтипів + ключові поля (`id`, `version`, `versionNonce`, `index`, `isDeleted`, …)
  - `packages/element/src/Scene.ts` — `Scene` працює з масивами/мапами `ExcalidrawElement`
  - `packages/excalidraw/actions/types.ts` — `ActionResult.elements?: readonly ExcalidrawElement[] | null`
- **NOT to confuse with**:
  - `ExcalidrawNonSelectionElement` — те саме, але без `selection` елемента.
  - `NonDeletedExcalidrawElement` / `NonDeleted<T>` — елементи з `isDeleted === false`.
  - `OrderedExcalidrawElement` — елемент із гарантованим `index` (fractional index) для порядку/синхронізації.

---

## Scene

- **Name (as in code)**: `Scene`
- **Definition (project-specific)**: Авторитетний контейнер елементів редактора. Тримає:
  - повний список елементів (включно з видаленими),
  - derived колекції (non-deleted, frames),
  - мапи за `id`,
  - `sceneNonce` як nonce для інвалідації кешів рендерера,
  - callback‑підписки `onUpdate`.
- **Where used (key files)**:
  - `packages/element/src/Scene.ts` — реалізація `Scene` (`replaceAllElements`, `mutateElement`, `triggerUpdate`, `getSceneNonce`, …)
  - `packages/excalidraw/components/App.tsx` — `App` володіє `this.scene` і читає/пише елементи через нього (див. також `docs/technical/architecture.md`)
- **NOT to confuse with**:
  - “SceneData” як payload для API (`updateScene`) — це структура даних для оновлень (див. `packages/excalidraw/types.ts` → `SceneData`), а не сам контейнер/клас.
  - “Canvas scene” (рендер) — рендеринг відбувається в `packages/excalidraw/renderer/*`, але `Scene` — про **дані**, не про малювання.

---

## SceneData

- **Name (as in code)**: `SceneData`
- **Definition (project-specific)**: Payload‑тип для передачі “часткового” стану сцени в API редактора (наприклад, `excalidrawAPI.updateScene(...)`): може містити `elements`, `appState`, `collaborators` і policy `captureUpdate` (як фіксувати зміни в історії/Store).
- **Where used (key files)**:
  - `packages/excalidraw/types.ts` — `export type SceneData = { elements?; appState?; collaborators?; captureUpdate? }`
  - `packages/excalidraw/components/App.tsx` — застосування оновлень сцени через API (оновлення елементів/стану/колабораторів)
  - `excalidraw-app/collab/Collab.tsx` — пуш remote updates через `excalidrawAPI.updateScene({ elements, captureUpdate: CaptureUpdateAction.NEVER })`
- **NOT to confuse with**:
  - `Scene` (клас/контейнер) — `SceneData` не зберігає стан, це лише **контракт на вхід**.
  - “scene file format” (експорт JSON) — формат експорту містить більше полів/метаданих; `SceneData` — runtime API.

---

## AppState

- **Name (as in code)**: `AppState`
- **Definition (project-specific)**: React‑state редактора — “UI + interaction state” (активний інструмент, вибір, newElement, viewport, модалки, налаштування, колабораційні курсори/списки, тощо). Це не тільки “налаштування”, а й поточний стан взаємодії.
- **Where used (key files)**:
  - `packages/excalidraw/types.ts` — `export interface AppState { ... }` (включає `activeTool`, `newElement`, `selectedElementIds`, `collaborators`, viewport, …)
  - `packages/excalidraw/components/App.tsx` — головний власник `state: AppState` (див. `docs/technical/architecture.md`)
  - `packages/excalidraw/actions/types.ts` — `ActionResult.appState?: Partial<AppState> | null`
- **NOT to confuse with**:
  - “Application state” хост‑додатку (`excalidraw-app/*`) — у хості є власний app-level стан/atoms, але `AppState` — це стан **редактора**.
  - `UIAppState` — підмножина `AppState`, яку віддають у деякі UI‑API/рендер‑колбеки.

---

## Tool / ToolType / ActiveTool

- **Name (as in code)**: `ToolType`, `ActiveTool` (і похідне `activeTool` в `AppState`)
- **Definition (project-specific)**:
  - `ToolType` — перелік інструментів редактора (selection, rectangle, arrow, text, image, hand, eraser, frame, laser, …).
  - `ActiveTool` — поточний вибір інструмента: або стандартний (`{ type: ToolType; customType: null }`), або кастомний (`{ type: "custom"; customType: string }`).
  - `AppState["activeTool"]` — `ActiveTool` + редакторські метадані (`lastActiveTool`, `locked`, `fromSelection`).
- **Where used (key files)**:
  - `packages/excalidraw/types.ts` — `ToolType`, `ActiveTool`, `AppState.activeTool`
  - `packages/excalidraw/components/App.tsx` — логіка взаємодії “який інструмент активний” (створення `newElement`, selection, eraser/hand, …)
- **NOT to confuse with**:
  - `ExcalidrawElementType` — тип елемента сцени (rectangle/text/arrow/…), це не завжди 1:1 з інструментами (є `laser`, `hand`, `eraser`, які не є елементами).
  - “Tool” як будь-яка утиліта/хелпер — тут маємо на увазі саме **інструмент редактора**.

---

## Action

- **Name (as in code)**: `Action` (і `ActionName`, `ActionResult`)
- **Definition (project-specific)**: “Дія редактора” (команда), яку можна викликати з UI/keyboard/API/command palette. Дія має `perform(...)`, опційно `keyTest`, `predicate`, `PanelComponent`, і повертає `ActionResult` (зміни для elements/appState/files + policy `captureUpdate`).
- **Where used (key files)**:
  - `packages/excalidraw/actions/types.ts` — контракт `Action`, `ActionName`, `ActionResult`
  - `packages/excalidraw/actions/manager.tsx` — виконання/фільтрація/рендер панелей дій (`ActionManager`)
  - `packages/excalidraw/actions/action*.ts(x)` — конкретні реалізації дій
- **NOT to confuse with**:
  - Redux‑style “actions” — це не dispatcher payload, а об’єкт‑команда з `perform`.
  - DOM events (click/keydown) — це тригери, а “Action” — абстракція команди поверх них.

---

## ActionManager

- **Name (as in code)**: `ActionManager`
- **Definition (project-specific)**: Реєстр і виконавець `Action`. Вміє:
  - зібрати підходящі дії для шорткату (`handleKeyDown`),
  - виконати дію з джерела (`executeAction`),
  - прокинути результат у `updater` (який апдейтить елементи/стан в `App`).
- **Where used (key files)**:
  - `packages/excalidraw/actions/manager.tsx` — реалізація
  - `packages/excalidraw/components/App.tsx` — створення `ActionManager` і “apply point” для `ActionResult` (див. `docs/technical/architecture.md`)
- **NOT to confuse with**:
  - “Command palette” як UI — palette лише один зі способів запуску action; `ActionManager` — ядро виконання.

---

## Collaboration

- **Name (as in code)**: `Collab`, `CollabAPI`, `Portal`, `Collaborator`, `SocketId`
- **Definition (project-specific)**: Продуктова фіча в `excalidraw-app`, яка:
  - створює/підключає “кімнату” (roomId/roomKey),
  - синхронізує **scene elements** через websocket (`Portal.broadcastScene`) і бекенд‑сховище (Firebase),
  - шле “епемерні” сигнали (курсор/idle/viewport bounds/follow) і оновлює `AppState.collaborators`.
- **Where used (key files)**:
  - `excalidraw-app/collab/Collab.tsx` — orchestration: start/stop, reconcile/restore, `syncElements`, `updateScene({ collaborators })`
  - `excalidraw-app/collab/Portal.tsx` — websocket transport + encryption + broadcast елементів/курсорів
  - `packages/excalidraw/types.ts` — `SocketId`, `Collaborator`, `AppState.collaborators`
- **NOT to confuse with**:
  - “Multiplayer” як server‑authoritative модель — тут **клієнти** обмінюються оновленнями елементів (version/versionNonce/index) і роблять reconcile.
  - “Share/export” — експорт/посилання існують окремо; Collaboration — це live sync під час редагування.

---

## Collaborator

- **Name (as in code)**: `Collaborator`
- **Definition (project-specific)**: Представлення іншого учасника сесії: pointer (x/y + tool pointer/laser), стан кнопки, selectedElementIds, username, idle state, колір курсора, avatar, socketId, тощо. Живе в `AppState.collaborators: Map<SocketId, Collaborator>`.
- **Where used (key files)**:
  - `packages/excalidraw/types.ts` — `SocketId`, `Collaborator`, `AppState.collaborators`
  - `excalidraw-app/collab/Collab.tsx` — `updateCollaborator`, `setCollaborators` і пуш у `excalidrawAPI.updateScene({ collaborators })`
  - `packages/excalidraw/components/UserList.tsx` — UI списку користувачів (споживає collaborators)
- **NOT to confuse with**:
  - “User account” — Collaborator не дорівнює системному юзеру; це runtime‑учасник кімнати (може мати `id?`, але ключ — `socketId`).

---

## Library

- **Name (as in code)**: `Library`, `LibraryItems`, `LibraryItem`
- **Definition (project-specific)**: “Бібліотека фігур” — колекція reusable шматків сцени (кожен item містить набір `elements`), яку можна додавати/оновлювати/публікувати/мержити та (опційно) персистити через adapter.
- **Where used (key files)**:
  - `packages/excalidraw/types.ts` — `LibraryItem`, `LibraryItems`, `onLibraryChange`, `ExcalidrawImperativeAPI.updateLibrary`
  - `packages/excalidraw/data/library.ts` — клас `Library` + merge/diff/update queue + adapter/persistence/migration
  - `packages/excalidraw/components/LibraryMenu.tsx` — UI бібліотеки
- **NOT to confuse with**:
  - npm “library” (`@excalidraw/excalidraw`) — це бібліотека коду; `Library` тут — **дані** користувацьких фігур.
  - “Assets library” (зображення/файли) — файли (`BinaryFiles`) зберігаються окремо; library items містять **елементи сцени**, які можуть посилатись на fileId, але library ≠ file store.

---

## LibraryItem

- **Name (as in code)**: `LibraryItem`
- **Definition (project-specific)**: Один запис у бібліотеці: `{ id, status, elements, created, name?, error? }`, де `elements` — масив non-deleted `ExcalidrawElement` (композиція фігури/групи), призначений для вставки в сцену.
- **Where used (key files)**:
  - `packages/excalidraw/types.ts` — визначення `LibraryItem` / `LibraryItems`
  - `packages/excalidraw/data/library.ts` — merge/diff/hash/persist/update
  - `packages/excalidraw/components/LibraryUnit.tsx` — рендер одного item в UI
- **NOT to confuse with**:
  - “Template” / “Scene” — LibraryItem це шматок, а не вся сцена.
  - `LibraryItem_v1` — legacy формат (депрекейт; лише для міграцій).

---

## ExcalidrawImperativeAPI

- **Name (as in code)**: `ExcalidrawImperativeAPI` (часто в документації/інтеграціях як `excalidrawAPI`)
- **Definition (project-specific)**: Імперативний API‑об’єкт, який хост‑додаток отримує через `onExcalidrawAPI` / `onMount` і використовує для інтеграцій: `updateScene`, `getSceneElements*`, `getAppState`, `addFiles`, `updateLibrary`, підписки `onChange/onIncrement/...`, тощо.
- **Where used (key files)**:
  - `packages/excalidraw/types.ts` — `export interface ExcalidrawImperativeAPI { ... }`
  - `excalidraw-app/collab/Collab.tsx` — колаб інтеграція читає scene/appState/files і пушить оновлення через `updateScene`
  - `excalidraw-app/App.tsx` — хостова обгортка навколо `<Excalidraw />` зазвичай тримає посилання на API та використовує `onChange`
- **NOT to confuse with**:
  - React props `<Excalidraw />` — props налаштовують поведінку, а `ExcalidrawImperativeAPI` — runtime “кермо”.
  - Внутрішній клас `App` (`packages/excalidraw/components/App.tsx`) — API лише проксі/контракт назовні, не сам редакторський компонент.

