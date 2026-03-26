# Active context

## Поточний стан

- **Ініціалізується Memory Bank документація** у `docs/memory/`.
- Вже підготовлено/оновлено:
  - `projectbrief.md` — місія/цілі/scope/non-goals/критерії успіху
  - `productContext.md` — UX-цілі, аудиторія, user stories
  - `techContext.md` — стек/версії/команди + інфраструктура/env
  - `systemPatterns.md` — архітектурні патерни, dependency direction, “карта” папок

## Фокус зараз

- **Завершити базовий набір 7 memory-файлів**: далі створити `progress.md` і `decisionLog.md`, а також переконатися, що всі файли узгоджені між собою та не дублюються.

## Найближчі кроки

- Створити `progress.md`:
  - “done” — ініціалізація 1–4 файлів
  - “in progress” — 5–7 файлів
  - “planned” — подальші уточнення (env, runtime залежності, collab specifics)
- Створити `decisionLog.md` з первинними рішеннями (монорепо, Yarn, Node>=18, Vite, esbuild для пакетів, Vitest/ESLint/Prettier, Docker+nginx для статики).

## Відомі невизначеності / ризики

- **Продуктові інтеграції** (Sentry/Firebase/Collab) можуть мати специфічні env-ключі та потоки, які не зафіксовані в memory-файлах без окремого опису/`.env.example`.
- **Контекст репозиторію** виглядає як навчальний форк/домашнє завдання; якщо є додаткові вимоги курсу — їх варто додати в `decisionLog.md` або окремий brief.

## Details

For detailed architecture → see `docs/technical/architecture.md`

For domain glossary → see `docs/product/domain-glossary.md`

