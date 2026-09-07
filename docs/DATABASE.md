# DATABASE.md — Схема данных (Supabase / Postgres)

## 1. Принципы

- Supabase Postgres как единственный источник истины для пользовательских данных (профили, дети, дневник, прогресс), когда V2.0 (см. [ROADMAP.md](ROADMAP.md)) подключит реальный бэкенд.
- Row Level Security (RLS) включён на всех таблицах — пользователь видит и изменяет только свои данные / данные своих детей.
- Контент (статьи, нормы, чек-листы, игры, факты, советы, спокойные моменты) на V1.5 — mock-данные в коде; раздел 4 описывает целевую модель, к которой контент переедет на V2.0.
- **Статус этого документа:** таблицы из §3 уже описаны реальным SQL-файлом миграции — [`supabase/migrations/20260717000000_init_schema.sql`](../app/supabase/migrations/20260717000000_init_schema.sql) — и им уже соответствуют интерфейсы Repository Pattern на фронтенде (`IDiaryRepository`, `IProgressRepository`, см. §8). Миграция ещё не применена к работающему проекту Supabase — это происходит на V2.0. Подробный разбор SQL, RLS и офлайн-синхронизации — в [`app/docs/DATABASE_AND_API.md`](../app/docs/DATABASE_AND_API.md); здесь — верхнеуровневая, продуктовая версия той же схемы.

## 2. ER-обзор

```
auth.users (Supabase Auth)
   └── 1:1 ── profiles
                  └── 1:N ── children
                                ├── 1:N ── diary_entries
                                └── 1:N ── completed_activities

Заложено на будущее, ещё не мигрировано:
   families (семейный кабинет, несколько родителей на ребёнка)
   checklist_progress (прогресс по пунктам чек-листа — сейчас localStorage)
   ai_chat_conversations → ai_chat_messages (V2.0, реальный AI)

content (V2.0, сейчас — mock в entities/*/mock):
   articles, milestones, checklists, checklist_items, games, facts, tips, calm_moments, specialists
```

## 3. Таблицы — пользовательский домен (реализованы миграцией `20260717000000_init_schema.sql`)

### `profiles`
Расширение `auth.users` данными родителя. `id` совпадает с `auth.users.id` — отдельного surrogate key нет.

| Колонка | Тип | Описание |
|---|---|---|
| id | uuid, PK, = auth.users.id | |
| full_name | text, nullable | Имя родителя |
| avatar_url | text, nullable | |
| preferred_locale | text, default 'ru', check ('ru'\|'kk'\|'en') | Язык интерфейса; `en` зарезервирован (см. [I18N.md](I18N.md)) |
| updated_at | timestamptz | Автообновляется триггером `set_updated_at` |

RLS: пользователь читает/пишет только строку с `id = auth.uid()`.

Тема оформления (light/dark/system) на V1.5 остаётся клиентской (`next-themes` + `localStorage`), в этой таблице не хранится — добавление `theme` в `profiles` рассматривается вместе с семейным кабинетом (`families`, ниже), когда появится необходимость синхронизировать её между устройствами одного родителя.

### `children`
| Колонка | Тип | Описание |
|---|---|---|
| id | uuid, PK | |
| profile_id | uuid, FK → profiles.id | Владелец |
| name | text | |
| birth_date | date | Используется для расчёта возраста |
| gender | text, default 'other', check ('male'\|'female'\|'other') | |
| created_at / updated_at | timestamptz | |

Индекс: `children_profile_id_idx` на `profile_id`.

RLS: доступ строго по `profile_id = auth.uid()`.

> **Известное расхождение:** клиентский тип `ChildGender` (`entities/child/model/types.ts`) сейчас использует `'male' | 'female' | 'unspecified'`, схема — `'other'` вместо `'unspecified'`. Маппинг на границе будущего Supabase-репозитория — см. `app/docs/DATABASE_AND_API.md`.

### `diary_entries`
Дневник развития — заметки родителя о ребёнке.

| Колонка | Тип | Описание |
|---|---|---|
| id | uuid, PK | |
| child_id | uuid, FK → children.id | |
| category | text, check ('word'\|'gesture'\|'game'\|'sleep'\|'behavior'\|'observation') | |
| content | text | Текст заметки |
| created_at / updated_at | timestamptz | |

Индекс: составной `diary_entries_child_id_created_at_idx` на `(child_id, created_at desc)` — под ленту дневника.

RLS: `child_id in (select id from children where profile_id = auth.uid())`.

Сегодня (V1.5) хранится в LocalStorage за интерфейсом `IDiaryRepository` (см. §8) — эта таблица описывает, во что она переедет на V2.0, без изменения контракта, который видит UI.

### `completed_activities`
Отметка «выполнено» для контента (игра/факт/статья/чек-лист/совет) для конкретного ребёнка — не путать с `checklist_progress` (ниже), который отмечает отдельные пункты внутри одного чек-листа.

| Колонка | Тип | Описание |
|---|---|---|
| id | uuid, PK | |
| child_id | uuid, FK → children.id | |
| activity_type | text, check ('game'\|'fact'\|'article'\|'checklist'\|'tip') | |
| activity_id | text | slug контента |
| completed_at | timestamptz | |

Индекс: уникальный составной `completed_activities_child_type_activity_idx` на `(child_id, activity_type, activity_id)` — гарантирует идемпотентность повторной отметки.

RLS: тот же принцип, что у `diary_entries` — доступ только через `children.profile_id = auth.uid()`.

### `families` (заложено на будущее, ещё не мигрировано)
Контейнер для будущего семейного кабинета (несколько родителей на одну семью). Сегодня `children.profile_id` указывает прямо на одного родителя — это сознательное упрощение V1.5/V2.0; переход к `families` не потребует миграции данных «с нуля», только добавление промежуточной таблицы и смену внешнего ключа.

| Колонка | Тип | Описание |
|---|---|---|
| id | uuid, PK | |
| name | text, nullable | Например, «Семья Ержановых» |
| owner_id | uuid, FK → profiles.id | Создатель семьи |
| created_at / updated_at | timestamptz | |

### `checklist_progress` (заложено на будущее, сейчас — localStorage)
Прогресс выполнения пунктов чек-листа для конкретного ребёнка (контент чек-листа — mock/будущий `checklist_items`).

| Колонка | Тип | Описание |
|---|---|---|
| id | uuid, PK | |
| child_id | uuid, FK → children.id | |
| checklist_id | text | slug чек-листа (пока mock-контент) |
| item_id | text | slug пункта |
| is_completed | boolean, default false | |
| completed_at | timestamptz, nullable | |

RLS (будущее): `child_id in (select id from children where profile_id = auth.uid())` — тот же принцип, что у `diary_entries`/`completed_activities`.

### `ai_chat_conversations`
| Колонка | Тип | Описание |
|---|---|---|
| id | uuid, PK | |
| profile_id | uuid, FK → profiles.id | |
| child_id | uuid, FK → children.id, nullable | Контекст ребёнка, если применимо |
| title | text, nullable | |
| created_at / updated_at | timestamptz | |

### `ai_chat_messages`
| Колонка | Тип | Описание |
|---|---|---|
| id | uuid, PK | |
| conversation_id | uuid, FK → ai_chat_conversations.id | |
| role | text | 'user' \| 'assistant' \| 'system' |
| content | text | |
| created_at | timestamptz | |

RLS: доступ через `conversation_id → profile_id = auth.uid()`.

## 4. Таблицы — контентный домен (целевая схема, V2.0+)

На V1.5 эти сущности существуют только как типы + mock-данные в `entities/*/mock` (430+ единиц, см. [CONTENT.md](CONTENT.md), [ROADMAP.md](ROADMAP.md)). Ниже — схема, к которой они мигрируют, чтобы архитектура данных не требовала переписывания фронтенда (те же поля, тот же контракт).

### `articles` (База знаний)
`id, slug, category, title, excerpt, cover_url, body (markdown/richtext), read_time_minutes, min_age_months, max_age_months, tags text[], locale, published_at`

### `milestones` (Возрастные нормы)
`id, domain ('physical'|'speech'|'cognitive'|'social_emotional'|'self_care'), age_range_min_months, age_range_max_months, title, description, locale`

### `checklists` / `checklist_items`
`checklists: id, slug, title, description, category, min_age_months, max_age_months, locale`
`checklist_items: id, checklist_id, title, description, order_index`

### `games`
`id, slug, title, description, min_age_months, max_age_months, development_goals text[], duration_minutes, materials_needed text[], cover_url, locale`

### `facts` («А вы знали?»)
`id, slug, category, title, body, age_range_min_months, age_range_max_months, read_time_seconds, locale`

### `tips` (Советы дня, «Полезно сегодня»)
`id, slug, category, title, body, age_range_min_months, age_range_max_months, locale`

### `calm_moments` (Спокойные моменты)
`id, slug, title, description, duration_minutes, when_to_use, parent_tip, locale`

### `specialists` (V4.0, каталог логопедов и детских центров РК)
`id, name, specialty, city, contact_info, verified boolean, is_b2b_account boolean`

### `subscriptions` (будущее)
`id, profile_id, plan, status, current_period_end`

## 5. Индексы и производительность

- `children(profile_id)`, `diary_entries(child_id, created_at desc)`, `completed_activities(child_id, activity_type, activity_id)` уникальный — уже созданы миграцией, см. §3.
- Будущие: `checklist_progress(child_id, checklist_id)`, `ai_chat_messages(conversation_id, created_at)` — составные индексы под частые запросы, когда эти таблицы будут мигрированы.
- Полнотекстовый поиск по `articles.title/body` через `pg_trgm` или `tsvector` (V2.0, см. [BACKLOG.md](BACKLOG.md) #8).

## 6. Миграции

- Миграции хранятся в `app/supabase/migrations/*.sql`, версионируются вместе с кодом.
- [`20260717000000_init_schema.sql`](../app/supabase/migrations/20260717000000_init_schema.sql) создаёт таблицы пользовательского домена, реализованные сегодня: `profiles`, `children`, `diary_entries`, `completed_activities`, с RLS и триггерами `updated_at`. Она ещё не применена к работающему проекту Supabase.
- Ещё предстоит на V2.0: триггер, создающий `profiles` при регистрации пользователя (`on_auth_user_created`); миграции для `families`, `checklist_progress`, `ai_chat_conversations`/`ai_chat_messages` и контентного домена (§4); настройка Supabase CLI-процесса миграций (см. [BACKLOG.md](BACKLOG.md) #4).

## 7. Безопасность данных

См. также [SAFETY.md](SAFETY.md).

- RLS включён по умолчанию на всех таблицах, политика "deny by default".
- Персональные данные ребёнка (имя, дата рождения) хранятся только в привязке к родительскому профилю, доступ строго по `auth.uid()` (напрямую для `children`, через подзапрос по `children` для `diary_entries`/`completed_activities` — см. §3).
- История AI-чата доступна только владельцу профиля; не используется для обучения моделей без явного согласия (заложить в Privacy Policy).

## 8. Repository Pattern (переход mock/LocalStorage → Supabase без переписывания UI)

Фронтенд не обращается к источнику данных напрямую — только через интерфейсы в `entities/*/model/repository.ts`:

- `IDiaryRepository` (`app/src/entities/diary-entry/model/repository.ts`) — `getEntries`, `addEntry`, `deleteEntry`.
- `IProgressRepository` (`app/src/entities/progress/model/repository.ts`) — `getCompletedActivities`, `markAsCompleted`.

Сегодня (V1.5) обе реализованы поверх LocalStorage (`LocalStorageDiaryRepository`, `LocalStorageProgressRepository`), с общими `useSyncExternalStore`-примитивами, которые использует и React-хук, и репозиторий — поэтому оба источника всегда видят одно и то же состояние. На V2.0 добавляются классы `SupabaseDiaryRepository`/`SupabaseProgressRepository`, реализующие те же интерфейсы поверх таблиц из §3 — хуки и компоненты, использующие интерфейс, не меняются.

Полный разбор реализации, идемпотентности `markAsCompleted` и схемы офлайн-синхронизации (FIFO-очередь мутаций, накапливаемых при `navigator.onLine === false`) — [`app/docs/DATABASE_AND_API.md`](../app/docs/DATABASE_AND_API.md).
