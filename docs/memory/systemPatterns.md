# System patterns and architecture

Перевірено за: кореневим/workspace `package.json`, `tsconfig.json`, `packages/excalidraw/index.tsx`, `packages/excalidraw/actions/manager.tsx`, `excalidraw-app/App.tsx`, `excalidraw-app/vite.config.mts`, `vitest.config.mts`, `scripts/buildPackage.js`.

## Архітектурний стиль

- **Модульний монорепо (workspaces)** з чіткими межами відповідальності між “ядром редактора”, доменними пакетами та продуктовим хост-додатком.
- **Не мікросервіси**: розгортання — статичний фронтенд (app) + окремі інтеграції (колаб/бекенд) підключаються на рівні `excalidraw-app`.

## Монорепозиторій і межі відповідальності

- **Три шари продукції:**
  1. **Бібліотека** — `packages/excalidraw`: публічний React-компонент і супутні модулі (експорт з `index.tsx`, внутрішній `App` у `components/App`).
  2. **Доменні пакети** — `@excalidraw/common` (константи, утиліти), `@excalidraw/math` (геометрія), `@excalidraw/element` (модель елементів, операції), `@excalidraw/utils`.
  3. **Хост-додаток** — `excalidraw-app`: збирання Vite, інтеграції (колаб, бекенд, аналітика), UI-обгортка навколо бібліотеки (`App.tsx` імпортує десятки модулів з `@excalidraw/excalidraw` та `@excalidraw/common` / `element`).

## Напрямок залежностей (dependency direction)

- `excalidraw-app` → залежить від `@excalidraw/excalidraw` та доменних пакетів.
- `@excalidraw/excalidraw` → залежить від доменних пакетів (`common/element/math/utils`).
- Доменні пакети **не повинні** тягнути `excalidraw-app` і (в ідеалі) не повинні тягнути UI-специфіку, якщо це не їхній контракт.

## Розділення збірки: додаток vs пакети

- **Додаток:** Vite + `@vitejs/plugin-react`, PWA, checker, EJS/HTML, кастомні chunk для локалей і важких залежностей (`vite.config.mts` `manualChunks`).
- **Пакет `@excalidraw/excalidraw`:** **esbuild** + `esbuild-sass-plugin`, інжекція env з `.env.development` / `.env.production` через `scripts/buildPackage.js` і `packages/excalidraw/env.cjs` — окремий конвеєр від Vite-додатку.

## Узгоджені alias для вихідників

- У **IDE/типах:** `tsconfig.json` `paths` маплять `@excalidraw/common|element|math|utils|excalidraw` на `packages/...`.
- У **runtime dev/test:** ті самі правила продубльовані в `resolve.alias` Vite та Vitest — імпорти з пакетів вказують на **TS/TSX джерела**, а не лише на `dist`.

## Патерн «дії редактора» (Command / shortcut)

- Клас **`ActionManager`** (`packages/excalidraw/actions/manager.tsx`): реєструє об’єкти `Action`, обробляє **keyboard** (пріоритет `keyPriority`), делегує оновлення через `updater`, підтримує **`Promise`** у результаті дії (`isPromiseLike`).
- Це центральний механізм узгодження UI, гарячих клавіш і змін стану редактора.

## Стан: Jotai і контекст API

- У бібліотеці: **`EditorJotaiProvider`** / `editorJotaiStore` імпортуються з `editor-jotai` у `packages/excalidraw/index.tsx` (рядки поблизу імпорту `EditorJotaiProvider`).
- **Публічний контекст API:** `ExcalidrawAPIProvider`, `ExcalidrawAPIContext` — дозволяють викликати `useExcalidrawAPI()` поза деревом `<Excalidraw />` (коментар у `index.tsx`).
- У додатку: окремий **`appJotaiStore`** і провайдер з `excalidraw-app/app-jotai` (імпорт у `App.tsx`) — стан рівня продукту (колаб, атоми для UI додатку).

## Рендеринг сцени та модель даних

- Модуль **`packages/excalidraw/scene`** реекспортує селекцію та нормалізацію з `@excalidraw/element` і власних файлів (`scroll`, `normalize`) — межа між «геометрією елементів» і «поданням сцени».
- Операції відновлення / порівняння елементів у додатку йдуть через `@excalidraw/excalidraw/data/*` і `@excalidraw/element` (наприклад `restore`, `reconcile` у імпортах `App.tsx`).

## PWA і продакшн-оболонка додатку

- `excalidraw-app/index.tsx`: **`registerSW`** з `virtual:pwa-register`, `StrictMode`, ініціалізація Sentry через side-effect імпорт `excalidraw-app/sentry.ts` (реальний шлях у `index.tsx`: `../excalidraw-app/sentry`) перед монтуванням React.
- Git SHA передається як `import.meta.env.VITE_APP_GIT_SHA` у `window.__EXCALIDRAW_SHA__`.

## Приклади інтеграції

- Workspace **`examples/with-nextjs`** та **`examples/with-script-in-browser`** — референсні патерни вбудовування (клієнт-only / динамічний імпорт описані в `packages/excalidraw/README.md`).

## Орієнтовна структура папок (для навігації)

Це не повний перелік, а “карта” ключових зон:

- `excalidraw-app/`
  - `index.tsx` — bootstrap, PWA SW, Sentry init
  - `App.tsx` — продуктова інтеграція навколо `<Excalidraw />`
  - `collab/` — колаборація (продуктовий шар)
  - `data/` — імпорт/експорт/перетворення даних на рівні app
- `packages/excalidraw/`
  - `index.tsx` — публічний експорт бібліотеки, провайдери/API
  - `actions/` — дії/гарячі клавіші (ActionManager)
  - `scene/` — рендеринг/нормалізація сцени
- `packages/element/`, `packages/math/`, `packages/common/`, `packages/utils/` — доменні та утилітарні модулі

## Підсумок

- Архітектурно це **vertical slice**: спільна модель і утиліти в малих пакетах, великий UI-редактор у `excalidraw`, продуктовий шар і деплой у `excalidraw-app`, з **двома конвеєрами збірки** (Vite для app, esbuild для publishable package).

## Details

For detailed architecture → see `docs/technical/architecture.md`

For domain glossary → see `docs/product/domain-glossary.md`
