# Tech context

Джерела: кореневий і робочі `package.json`, `tsconfig.json`, `excalidraw-app/vite.config.mts`, `vitest.config.mts`, `Dockerfile`, `.github/workflows/test.yml`.

## Менеджер пакетів і workspace

- **Package manager:** `yarn@1.22.22` (поле `packageManager` у корені).
- **Workspaces:** `excalidraw-app`, `packages/*`, `examples/*`.
- **Node:** `engines.node` — `>=18.0.0` (корінь і `excalidraw-app`).

## Мови та основні версії

| Технологія | Версія (де зафіксовано) |
|------------|-------------------------|
| TypeScript | `5.9.3` (root `devDependencies`) |
| React / React DOM | `19.0.0` (`excalidraw-app` `dependencies`) |
| Vite | `5.0.12` (root) |
| Vitest | `3.0.6` (root); `@vitest/coverage-v8` `3.0.7` |
| ESLint / Prettier | через `@excalidraw/eslint-config` `1.0.3`, `prettier` `2.6.2` |

## Збірка та середовище виконання

- **Dev-сервер додатку:** Vite; порт з `VITE_APP_PORT` або **3000** за замовчуванням (`excalidraw-app/vite.config.mts`: `server.port`).
- **Вихід продакшн-білду додатку:** каталог `excalidraw-app/build` (`vite.config.mts` `build.outDir`, `Dockerfile` копіює саме його).
- **Статична видача образу:** `nginx:1.27-alpine` після білду (`Dockerfile`).
- **Docker Compose:** сервіс `excalidraw`, публікація **3000:80** (`docker-compose.yml`).

## Бібліотечні пакети (монорепо)

- **`@excalidraw/excalidraw`:** `0.18.0` — збірка через `node ../../scripts/buildPackage.js` + генерація типів (`packages/excalidraw/package.json` `build:esm`).
- **`@excalidraw/common`, `@excalidraw/element`, `@excalidraw/math`:** `0.18.0` у своїх `package.json`.
- **`@excalidraw/utils`:** `0.1.2` (`packages/utils/package.json`).

Залежності додатку (фрагмент): **Firebase** `11.3.1`, **Jotai** `2.11.0`, **Sentry** `@sentry/browser` `9.0.1`, **socket.io-client** `4.7.2` (`excalidraw-app/package.json`).

## Інфраструктура / деплой

- **Локальна розробка:** Vite dev server (`yarn start`) на `http://localhost:3000` (або порт з `VITE_APP_PORT`).
- **Контейнеризація:** multi-stage `Dockerfile` (build → `nginx:1.27-alpine`), статика з `excalidraw-app/build`.
- **Compose:** `docker-compose.yml` публікує `3000:80` для контейнера.

## Конфігурація (env)

- **Vite env:** як мінімум використовується `VITE_APP_PORT` для порту dev-сервера.
- Продуктові інтеграції (Sentry/Firebase/Collab) очікують власні змінні середовища у `excalidraw-app` (точні назви залежать від налаштувань і мають бути описані в `.env*`/документації додатку, якщо вмикаються).

## Інструменти якості

- **Тести:** Vitest, `environment: "jsdom"`, `setupFiles: ./setupTests.ts`, покриття з порогами (`vitest.config.mts`).
- **Лінт / формат:** `yarn test:code` → ESLint; `yarn test:other` → Prettier `--list-different` (скрипти кореня).
- **Типізація:** `yarn test:typecheck` → `tsc` (без emit, `tsconfig.json` `noEmit: true`).

## Команди (корінь репозиторію)

| Команда | Що робить (за `scripts`) |
|---------|--------------------------|
| `yarn install` | встановлення залежностей для workspaces |
| `yarn start` | `yarn --cwd ./excalidraw-app start` → залежності + `vite` |
| `yarn build` | `yarn --cwd ./excalidraw-app build` (app build + `build:version`) |
| `yarn build:packages` | послідовно `common`, `math`, `element`, `excalidraw` |
| `yarn start:example` | `build:packages` + запуск `examples/with-script-in-browser` |
| `yarn test` / `yarn test:app` | Vitest |
| `yarn test:all` | typecheck + eslint + prettier + `test:app --watch=false` |
| `yarn test:coverage` | Vitest з coverage |
| `yarn fix` | Prettier write + ESLint fix |
| `yarn clean-install` | видалення `node_modules` + `yarn install` |

## CI

- Workflow **Tests** на push у `master`: Node **20.x**, `yarn install`, `yarn test:app` (`.github/workflows/test.yml`).

## Шляхи імпортів у dev

- TypeScript **paths** у `tsconfig.json` і **такі самі alias** у Vite та Vitest для `@excalidraw/*` → вихідні файли в `packages/*/src` або `packages/excalidraw/`.

## Details

For detailed architecture → see `docs/technical/architecture.md`

For domain glossary → see `docs/product/domain-glossary.md`
