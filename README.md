# Balumi — тёплый помощник для совместных занятий с ребёнком

> Помогаем родителям детей от 0 до 7 лет выбирать занятия, понимать развитие ребёнка и проводить время вместе: возрастные нормы, чек-листы, база знаний, развивающие игры, спокойные моменты и AI-помощник.
>
> Внутреннее кодовое имя проекта на этапе разработки — «Kids Growth»: оно всё ещё встречается в части документации, миграций БД и storage-ключей (менять их означало бы терять данные пользователей или переписывать историю миграций). Пользовательское имя приложения — **Balumi**.

Проект **не является медицинским сервисом**. Он не ставит диагнозы, не назначает лечение и не заменяет врача — он помогает родителям ориентироваться в развитии ребёнка и вовремя обращаться к специалистам.

---

## Статус проекта

**V1.5 — offline-first ежедневный помощник.** Следующая версия — V2.0 (Supabase-бэкенд). См. [docs/ROADMAP.md](docs/ROADMAP.md).

## Документация

| Документ | Описание |
|---|---|
| [docs/PRODUCT.md](docs/PRODUCT.md) | Продуктовое видение, целевая аудитория, ценностное предложение |
| [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) | Функциональные и нефункциональные требования |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Техническая архитектура, Feature-Based Architecture |
| [docs/DATABASE.md](docs/DATABASE.md) | Схема данных Supabase/Postgres |
| [docs/AI.md](docs/AI.md) | Принципы работы AI-помощника, ограничения, safety |
| [docs/AI_COMPANION.md](docs/AI_COMPANION.md) | Реализация «Помощника» в коде на V1.5, план перехода на реальный LLM в V2.0 |
| [docs/UI_UX.md](docs/UI_UX.md) | Дизайн-система, UX-принципы, mobile-first |
| [docs/I18N.md](docs/I18N.md) | Мультиязычность (ru/kk, готовность к en) |
| [docs/CONTENT.md](docs/CONTENT.md) | Контентная модель: база знаний, нормы, чек-листы, игры, факты, спокойные моменты |
| [docs/CONTENT_RULES.md](docs/CONTENT_RULES.md) | Разрешённые/запрещённые формулировки для фактов, музыки, AI |
| [docs/BRAND.md](docs/BRAND.md) | Бренд-бук: миссия, тон, цвет, типографика, иллюстрации |
| [docs/BACKLOG.md](docs/BACKLOG.md) | Технический бэклог (High/Medium/Low), не привязанный к версии |
| [docs/SAFETY.md](docs/SAFETY.md) | Медицинские дисклеймеры, безопасность контента и данных |
| [docs/ROADMAP.md](docs/ROADMAP.md) | Дорожная карта развития продукта |
| [docs/CMS_BACKEND_ARCHITECTURE.md](docs/CMS_BACKEND_ARCHITECTURE.md) | Аудит архитектуры и план перехода к Content Platform + Admin CMS + бэкенду (не реализовано — только планирование) |

## Стек

- **Next.js (App Router)** + **TypeScript (strict)**
- **TailwindCSS** + **shadcn/ui**
- **Supabase** (Auth, Postgres, Storage)
- **React Query** — серверное состояние
- **React Hook Form** + **Zod** — формы и валидация
- **Framer Motion** — анимации
- **next-intl** — i18n (ru/kk, расширяемо на en)
- **PWA** (Serwist) — офлайн-режим, установка на телефон, готовность к WebView-обёртке
- **Vercel** — деплой

## Архитектура

Проект использует **Feature-Based Architecture**:

```
src/
  app/          # Next.js App Router: маршруты, layout, страницы
  features/     # Изолированные фичи (auth, knowledge-base, milestones, checklists, games, ai-chat, profile, settings)
  entities/     # Доменные сущности (child, user, article, milestone, checklist, game)
  widgets/      # Композитные UI-блоки из нескольких entities/features (bottom-nav, header, hero и т.д.)
  shared/       # Переиспользуемые UI-компоненты, хуки, утилиты, конфиги, i18n, api-клиенты
```

Подробнее — [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Быстрый старт

```bash
cd app
npm install
cp .env.example .env.local   # заполнить ключи Supabase
npm run dev
```

Открыть [http://localhost:3000](http://localhost:3000).

### Полезные команды

```bash
npm run dev        # локальная разработка
npm run build       # production build
npm run start        # запуск production build
npm run lint          # ESLint
npm run typecheck      # проверка типов TypeScript
```

## Принципы разработки

Смотри [CLAUDE.md](CLAUDE.md) — главный источник требований и принципов проекта. Ключевое:

- Mobile First — каждый экран проектируется сначала под телефон.
- Простота, забота, спокойствие в тоне и UX.
- Доказательная информация, никакой самодеятельности в медицинском контенте.
- Чистая, масштабируемая, легко расширяемая архитектура (SOLID, DRY, KISS).
- AI — помощник, а не врач.

## Лицензия

Проприетарный проект. Все права защищены.
