# ARCHITECTURE.md — Техническая архитектура

## 1. Обзор стека

| Слой | Технология | Назначение |
|---|---|---|
| Фреймворк | Next.js (App Router) | SSR/SSG/CSR, файловая маршрутизация, серверные компоненты |
| Язык | TypeScript (strict) | Типобезопасность |
| Стили | TailwindCSS | Utility-first CSS |
| UI-компоненты | shadcn/ui (Radix UI) | Доступные, кастомизируемые примитивы |
| Backend-as-a-Service | Supabase | Auth, Postgres, Storage, Row Level Security |
| Серверное состояние | React Query (TanStack Query) | Кэширование, синхронизация запросов |
| Формы | React Hook Form + Zod | Управление формами и схемная валидация |
| Анимации | Framer Motion | Плавные переходы и микровзаимодействия |
| i18n | next-intl | Мультиязычность ru/kk (+en в будущем) |
| PWA | Serwist (`@serwist/next`) | Service worker, офлайн-кэш, манифест |
| Хостинг | Vercel | Деплой, edge network |

## 2. Feature-Based Architecture

Проект структурирован по принципу, близкому к Feature-Sliced Design, адаптированному под Next.js App Router:

```
src/
├── app/                     # Next.js App Router — только маршрутизация и композиция
│   ├── [locale]/
│   │   ├── (main)/          # Группа маршрутов с основным layout (bottom nav)
│   │   │   ├── page.tsx             # Главная
│   │   │   ├── knowledge/            # База знаний
│   │   │   ├── milestones/           # Возрастные нормы
│   │   │   ├── checklists/           # Чек-листы
│   │   │   ├── games/                # Игры
│   │   │   ├── ai-chat/              # AI Chat
│   │   │   ├── profile/              # Профиль
│   │   │   └── settings/             # Настройки
│   │   ├── (auth)/           # Группа маршрутов авторизации (без bottom nav)
│   │   │   ├── login/
│   │   │   └── register/
│   │   └── layout.tsx
│   ├── manifest.ts / manifest.json
│   └── globals.css
│
├── features/                # Изолированная бизнес-логика конкретных фич
│   ├── auth/                 # форма входа/регистрации, хуки useSession, actions
│   ├── knowledge-base/        # поиск, фильтрация, список/детали статьи
│   ├── milestones/            # логика возрастных норм, расчёт возраста
│   ├── checklists/             # логика прогресса чек-листов
│   ├── games/                   # фильтрация игр по возрасту/цели
│   ├── ai-chat/                  # логика чата, отправка сообщений
│   ├── child-profile/             # CRUD профиля ребёнка
│   └── settings/                   # смена языка/темы
│
├── entities/                 # Доменные модели и их типовые UI-представления
│   ├── user/                  # тип User, UserCard и т.д.
│   ├── child/                   # тип Child, ChildAvatar, возрастные утилиты
│   ├── article/                   # тип Article, ArticleCard
│   ├── milestone/                   # тип Milestone, MilestoneBadge
│   ├── checklist/                     # тип Checklist/ChecklistItem
│   └── game/                            # тип Game, GameCard
│
├── widgets/                  # Композитные блоки, собранные из entities/features
│   ├── bottom-nav/
│   ├── header/
│   ├── hero-section/
│   └── child-summary-card/
│
└── shared/                   # Общее, не зависящее от домена
    ├── ui/                    # обёртки shadcn/ui, дизайн-система
    ├── lib/                    # утилиты (cn, date, format)
    ├── api/                      # supabase client, react-query client
    ├── config/                     # константы, роуты, feature flags
    ├── hooks/                        # общие хуки (useMediaQuery и т.д.)
    ├── i18n/                           # конфигурация next-intl, messages
    └── types/                           # общие типы
```

### Правила зависимостей (важно для масштабируемости)

Направление импортов — только «сверху вниз» по уровню абстракции:

```
app  →  widgets  →  features  →  entities  →  shared
```

- `shared` ничего не импортирует из других слоёв.
- `entities` может импортировать только `shared`.
- `features` может импортировать `entities` и `shared`.
- `widgets` может импортировать `features`, `entities`, `shared`.
- `app` может импортировать всё.
- Горизонтальные импорты между модулями одного слоя (например, `features/games` → `features/checklists`) не допускаются напрямую — через `entities`/`shared` либо переиспользуемый хук.

Это правило предотвращает превращение проекта в «мусорную корзину» из `components/` и обеспечивает независимую эволюцию фич.

## 3. Маршрутизация и локализация

- Локаль — первый сегмент пути: `/ru/...`, `/kk/...`.
- Middleware (`middleware.ts`) определяет локаль по cookie/Accept-Language и делает редирект.
- Группы маршрутов Next.js `(main)` и `(auth)` разделяют layout с навигацией и без.

## 4. Состояние приложения

- **Серверное состояние** (данные, которые в будущем придут из Supabase/API) — React Query: кэш, инвалидация, оптимистичные обновления.
- **Клиентское UI-состояние** (открыт ли модал, выбранный таб) — локальный `useState`/`useReducer`, без глобального стора на Этапе 1.
- **Сессия пользователя** — Supabase Auth (`@supabase/ssr`), пробрасывается через React Context (`shared/api/session-provider`).
- **Формы** — React Hook Form + resolver на Zod-схемах из `entities/*/model/schema.ts`.

## 5. Работа с данными (Этап 1)

- Контент (статьи, нормы, чек-листы, игры) — mock-данные в виде типизированных JSON/TS-модулей внутри `entities/*/mock`, отдаются через локальные функции, имитирующие асинхронный API (`Promise` + задержка), чтобы React Query работал по-настоящему.
- Пользовательские данные (профиль, дети, авторизация) — реальный Supabase (Postgres + Auth), см. [DATABASE.md](DATABASE.md).
- Дизайн слоя данных предполагает бесшовную замену mock-функций на реальные Supabase-запросы без изменения UI-кода (паттерн repository/hook).

## 6. AI-интеграция (архитектурная готовность)

- `features/ai-chat` содержит клиентский UI и серверный route handler-заглушку (`app/api/ai-chat/route.ts`), который на Этапе 1 возвращает заранее заданные безопасные ответы/эхо, а в будущем будет проксировать запрос к LLM-провайдеру с системным промптом безопасности (см. [AI.md](AI.md)).
- Архитектура позволяет заменить mock-обработчик на реальный вызов модели без изменения фронтенд-кода (единый контракт `POST /api/ai-chat`).

## 7. PWA

- `app/manifest.ts` — имя, иконки, theme_color, display: standalone (генерирует `/manifest.webmanifest`).
- Service worker (`src/app/sw.ts`) собирается через `@serwist/next`, кэширует статику и app-shell для офлайн-доступа.
- Все компоненты избегают API, недоступных в WebView (например, полагаются на feature-detection перед использованием Web Share API, Notifications API и т.д.).
- **Важно:** `@serwist/next` на момент Этапа 1 не поддерживает Turbopack (см. [serwist/serwist#54](https://github.com/serwist/serwist/issues/54)), а Next.js 16 использует Turbopack по умолчанию и для `dev`, и для `build`. Поэтому команды `dev`/`build` в `package.json` явно используют флаг `--webpack`, иначе service worker не собирается и PWA-функциональность молча отключается. Как только Serwist получит стабильную поддержку Turbopack, флаг можно будет убрать.

## 8. Тема и дизайн-токены

- CSS-переменные Tailwind + shadcn/ui theming (light/dark), переключение через `next-themes`.
- Токены цвета/радиуса/типографики централизованы в `app/globals.css` и `tailwind.config.ts` — см. [UI_UX.md](UI_UX.md).

## 9. Готовность к будущим фичам

Архитектура намеренно не блокирует:

- **Push-уведомления** — через Service Worker + Web Push API, интеграция в `shared/api/push`.
- **Несколько детей / семейный кабинет** — модель данных уже поддерживает связь `family — parent — child (1:N)`.
- **Подписки** — отдельная фича `features/billing`, таблица `subscriptions` в БД зарезервирована.
- **Каталог специалистов** — `entities/specialist`, `features/specialists-catalog`.
- **Дневник развития** — `entities/diary-entry`, привязан к `child_id`.
- **Админ-панель** — отдельное Next.js приложение или `/admin` route group с проверкой роли — не мешает текущей структуре.

## 10. Сборка и деплой

- `npm run build` — production build (Next.js).
- Vercel — автоматический деплой по git push (в будущем, при подключении репозитория).
- Переменные окружения — через `.env.local` (dev) и Vercel Project Settings (prod), см. `.env.example`.
