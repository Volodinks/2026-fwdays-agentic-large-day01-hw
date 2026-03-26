# Decision log

Нижче зафіксовано ключові технічні та продуктові рішення, які впливають на розробку і підтримку проєкту.

| ID | Дата | Категорія | Рішення | Статус | Обґрунтування | Наслідки |
|----|------|-----------|---------|--------|---------------|----------|
| DEC-001 | 2026-03-26 | Архітектура | Використовувати **модульний монорепозиторій** (`excalidraw-app`, `packages/*`, `examples/*`) | Прийнято | Спільна розробка app + бібліотеки в одному життєвому циклі | Потрібна дисципліна меж залежностей і узгоджені релізні процеси |
| DEC-002 | 2026-03-26 | Package management | Використовувати **Yarn 1 workspaces** (`yarn@1.22.22`) | Прийнято | Стабільна робота існуючих скриптів і структури репозиторію | Оновлення менеджера пакетів потребуватиме окремої міграції |
| DEC-003 | 2026-03-26 | Runtime | Базова версія Node.js: **>=18.0.0** | Прийнято | Узгоджено з `engines.node` у root/app | CI/локальне середовище мають відповідати мінімальній версії |
| DEC-004 | 2026-03-26 | Frontend platform | Основний стек app: **React + TypeScript + Vite** | Прийнято | Швидкий dev-loop, сучасний build-tooling, типобезпечність | Конфігурація Vite/TS є критичною точкою для DX |
| DEC-005 | 2026-03-26 | Build system | Розділити конвеєри збірки: **Vite для app**, **esbuild для publishable packages** | Прийнято | Різні цілі артефактів: runtime app vs бібліотечні бандли | Потрібно підтримувати консистентність env/alias між конвеєрами |
| DEC-006 | 2026-03-26 | Code quality | Базовий quality gate: **Vitest + ESLint + Prettier + TypeScript typecheck** | Прийнято | Підвищення надійності змін та швидкий фідбек у CI | Повільніші повні прогони, але менше регресій |
| DEC-007 | 2026-03-26 | Deployment | Контейнеризація app через **Docker multi-stage + nginx**; compose порт `3000:80` | Прийнято | Простий і відтворюваний спосіб локального/демо деплою | Потрібно стежити за сумісністю build output і nginx конфігу |
| DEC-008 | 2026-03-26 | State management | Визнати двошаровий state-підхід: **Jotai у бібліотеці** + **app-level store у host app** | Прийнято | Розділення стану редактора та продуктового orchestration шару | Важливо уникати дублювання відповідальностей між stores |
| DEC-009 | 2026-03-26 | Product API | Підтримувати публічний інтеграційний контракт через `<Excalidraw />` + API context/hooks | Прийнято | Це основа використання бібліотеки зовнішніми продуктами | Зміни API потребують контрольованої еволюції/сумісності |
| DEC-010 | 2026-03-26 | Documentation process | Вести Memory Bank у `docs/memory` як “живу” документацію | Прийнято | Прозорий контекст для команди та швидше онбординг/планування | Потрібен регулярний review і оновлення після значущих змін |
| DEC-011 | 2026-03-26 | Undocumented behavior | **Код:** у `packages/element/src/store.ts` реалізована неявна state machine `CaptureUpdateAction` (`IMMEDIATELY/NEVER/EVENTUALLY`) з пріоритетами планування. **Документація:** у `docs/technical/architecture.md` і `docs/product/PRD.md` не описані стани/переходи цього механізму. | Прийнято до документування | Це критичний механізм історії/батчингу змін; без явної специфікації висока ймовірність регресій при рефакторингу | Додати окрему секцію про transition rules і інваріанти commit/schedule в технічну документацію |
| DEC-012 | 2026-03-26 | Undocumented behavior | **Код:** у `packages/excalidraw/components/App.tsx` і `excalidraw-app/App.tsx` init-flow залежить від порядку `isLoading -> initializeScene -> updateScene -> onChange/commit`, включно зі спецгілкою на `hashchange`. **Документація:** порядок ініціалізації host app vs editor не формалізований. | Прийнято до документування | Це initialization-order dependency з високим ризиком прихованих багів при зміні lifecycle | Зафіксувати sequence-діаграму запуску і список допустимих side effects під час init |
| DEC-013 | 2026-03-26 | Undocumented behavior | **Код:** у `packages/excalidraw/data/restore.ts` під час restore є неочевидний side effect: порожні text-елементи позначаються `isDeleted=true` і викликається `bumpVersion(element)` (поруч `TODO` про ризик для sync/versioning). **Документація:** поведінка restore як mutating-step не описана. | Прийнято до документування | Впливає на дельта-синхронізацію і відтворюваність стану між клієнтами | Документувати mutating-правила restore і обмеження для sync-контракту |
| DEC-014 | 2026-03-26 | Undocumented behavior | **Код:** у `packages/element/src/selection.ts` `isSomeElementSelected` використовує closure-cache (internal mutable state), а у `packages/excalidraw/components/App.tsx` присутній `HACK` для mobile transform handles. **Документація:** немає явного контракту про stateful-утиліти і тимчасові UI-обмеження. | Прийнято до документування | Приховані side effects і hack-логіка ускладнюють передбачуваність поведінки та тестування | Додати "Known behavioral constraints" (stateful helpers, mobile-specific hacks, план deprecation) |

## Open decisions

- **OD-001:** Формалізувати джерело істини для env-змінних інтеграцій (`Sentry`, `Firebase`, `Collab`) — окремий `.env.example` або секція в технічній документації.
- **OD-002:** Зафіксувати cadence оновлення Memory Bank (наприклад: раз на спринт або після архітектурних змін).

## Details

For detailed architecture → see `docs/technical/architecture.md`

For domain glossary → see `docs/product/domain-glossary.md`

