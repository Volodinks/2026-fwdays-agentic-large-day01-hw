# Project brief — Excalidraw monorepo

## Що це за проєкт

- **Назва в репозиторії:** приватний Yarn-workspace монорепозиторій `excalidraw-monorepo` (кореневий `package.json`: `"name": "excalidraw-monorepo"`).
- **Продукт:** екосистема навколо **Excalidraw** — веб-додаток для малювання діаграм і схем у «hand-drawn» стилі та **React-бібліотека** `@excalidraw/excalidraw` для вбудовування редактора в інші застосунки (`packages/excalidraw/README.md`, експорт з `packages/excalidraw/index.tsx`).

## Основна мета

- **Для кінцевого користувача:** повноцінний редактор на полотні (експорт, бібліотека фігур, теми, локалізації тощо) через застосунок `excalidraw-app`.
- **Для інтеграторів:** стабільний публічний API компонента `<Excalidraw />` з CSS та типами, плюс приклади в `examples/with-nextjs` і `examples/with-script-in-browser` (перелік workspace у кореневому `package.json`).

## Склад репозиторію (верхній рівень)

| Частина | Призначення (за кодом і манифестами) |
|--------|--------------------------------------|
| `excalidraw-app/` | Офіційний веб-додаток: Vite + React, точка входу `index.tsx`, обгортка `App.tsx` навколо `@excalidraw/excalidraw`. |
| `packages/common`, `packages/math`, `packages/element`, `packages/utils` | Доменні та допоміжні пакети з версіями в своїх `package.json` (напр. `@excalidraw/common` 0.18.0). |
| `packages/excalidraw` | Основна бібліотека UI та логіки редактора (`@excalidraw/excalidraw` 0.18.0). |
| `examples/*` | Демонстрація вбудовування в Next.js і в браузері через скрипт. |
| `scripts/` | Збірка пакетів (esbuild), релізи, локалі тощо. |

## Що робить саме `excalidraw-app` (з `App.tsx`)

- Імпортує **ядро** з `@excalidraw/excalidraw` (`Excalidraw`, API-хуки, діалоги, `CommandPalette`, аналітика).
- Підключає **спільні утиліти** з `@excalidraw/common` та типи / модель елементів з `@excalidraw/element`.
- Містить власний шар: **колаборація** (`collab/Collab`), **Firebase**, **Jotai**-стан (`app-jotai`), експорт/імпорт через `excalidraw-app/data`, **Sentry** — side-effect імпорт `excalidraw-app/sentry.ts` на старті в `index.tsx`.

## Контекст назви гілки / курсу

- Шлях репозиторію (`2026-fwdays-agentic-large-day01-hw`) вказує на **навчальне домашнє завдання**; у самому коді домен залишається **Excalidraw** — немає окремого `README` у корені з описом курсу, тому опис продукту базується на `package.json` і вихідному коді пакетів.

## Обмеження цієї пам’яті

- Файл короткий навмисно; деталі стеку та команд — у `techContext.md`, архітектура — у `systemPatterns.md`.
