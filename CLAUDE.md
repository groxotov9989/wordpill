# WordPill — Project Context

Telegram Mini App для изучения медицинского английского. ~200 учеников, 16+ преподавателей.

## Stack

- **Frontend**: чистый HTML/JS, без фреймворков
- **Backend**: Supabase (Postgres + RLS + Edge Functions)
- **Деплой**: GitHub Pages (статика) + Supabase (БД/функции)

## Файлы

| Файл | Описание |
|------|----------|
| `index.html` | Telegram Mini App для учеников |
| `teacher.html` | Дашборд преподавателя (браузер, по ссылке с токеном) |
| `analytics.html` | Аналитика |
| `pitch.html` | Питч-страница |
| `~/Downloads/admin.html` | Локальный админ-кабинет (НЕ на гите, НЕ деплоить) |

## URLs

- GitHub repo: `https://github.com/groxotov9989/wordpill`
- GitHub Pages: `https://groxotov9989.github.io/wordpill/`
- Supabase project ID: `sqwnkjpflxskrmwladyn`

## Аутентификация

**Ученики** (`index.html`):
- Telegram `initData` → Edge Function `telegram-auth` → JWT → роль `authenticated`
- Если JWT не прошёл — фоллбэк на `anon` (намеренно, нужен для bootstrap `upsertUser`)

**Преподаватели** (`teacher.html`):
- Токен из URL (`?token=...`) → читается один раз, сохраняется в `sessionStorage`
- Supabase-клиент создаётся с `global.headers: { 'x-teacher-token': token }`
- RLS проверяет токен через `get_teacher_id_from_token()` (SECURITY DEFINER)
- Никакого JWT — только anon роль + кастомный заголовок

## Таблицы

| Таблица | Назначение |
|---------|-----------|
| `users` | Ученики (id = Telegram user_id) |
| `teachers` | Преподаватели, у каждого уникальный `token` |
| `teacher_students` | Связь преподаватель ↔ ученик (many-to-many) |
| `words` | Личный словарь ученика |
| `packs` | Паки слов. `teacher_id IS NULL` = системный (виден всем), иначе — пак преподавателя |
| `pack_words` | Слова в паке. FK → `packs.id ON DELETE CASCADE` |
| `pack_assignments` | Кому назначен пак. `scope`: `school`/`group`/`user`. FK → `packs.id ON DELETE CASCADE` |
| `study_sessions` | Сессии изучения/квиза |
| `user_categories` | Категориальные настройки ученика |

## RPC-функции

| Функция | Что делает |
|---------|-----------|
| `get_teacher_id_from_token()` | Читает `x-teacher-token` из заголовков, возвращает `teachers.id` |
| `get_packs_for_student(p_user_id)` | Паки видимые ученику: системные + явные `pack_assignments` |
| `get_pack_word_counts()` | Количество слов в каждом паке |
| `get_student_word_counts(student_ids)` | Количество слов у каждого ученика |
| `search_users_for_teacher(p_query)` | Поиск пользователей для добавления в группу |
| `create_teacher_admin(p_name, p_tg_id, p_admin_secret)` | Создать препода (только для admin.html) |
| `delete_teacher_admin(p_teacher_id, p_admin_secret)` | Удалить препода (только для admin.html) |

## Edge Functions

- `telegram-auth` — валидирует Telegram initData, возвращает JWT
- `send-reminders` — рассылка push-уведомлений (cron)

## Ключевые решения (не менять без понимания)

- **`users_anon_read: USING (true)`** — намеренно открытая политика. Попытка ограничить её сломала загрузку приложения у учеников: JWT ещё не применился → `upsertUser` падал с 406 → дубликат → зависание.

- **Автоназначение паков при создании удалено** — было добавлено и убрано. Новые паки появлялись у всех учеников группы сразу, создавая мусор. Теперь назначение только явное через кнопку "Назначить".

- **pack_assignments — единственный источник видимости** — пак преподавателя виден ученику только если есть запись в `pack_assignments`. Принадлежность пака преподавателю (`teacher_id`) не даёт автоматической видимости.

- **Токен препода в sessionStorage** — убирается из URL сразу после чтения (история браузера), но сохраняется в `sessionStorage` чтобы обновление страницы не ломало сессию.

## Как добавить нового преподавателя

Открыть `~/Downloads/admin.html` в браузере, ввести пароль, нажать "+ Добавить преподавателя". Файл локальный — не деплоить и не коммитить.
