# Сиеста — AI-ассистент для салона массажа

Набор из 6 воркфлоу [n8n](https://n8n.io): чат-бот «Сиеста» для салона «Здоровье ручной работы» записывает клиентов в Google Calendar, отвечает по прайсу из базы знаний, напоминает о визитах, собирает отзывы, шлёт отчёты и сам разбирает ошибки.

## Как это устроено

![Полная схема архитектуры Siesta](All.png)
*Общая архитектура автоматизации и взаимодействия модулей*

```
Telegram ──► Siesta (AI-агент) ──► Calendar (запись, перенос, отмена)
                │  ├─ search_docs2 ◄── Rag system ◄── Google Drive
                │  ├─ Search in Tavily
                │  └─ Call_admin
                └─ история диалогов ──► Postgres ──► Report ──► Gmail

Alerts: календарь ──► напоминания и опрос ──► Telegram
Error scenario: любой упавший воркфлоу ──► админ в Telegram
```

| Воркфлоу | Что делает |
|---|---|
| `Siesta` | Основной AI-агент (Gemini, память в Postgres на 25 сообщений). Знает прайс из базы знаний, ведёт запись, зовёт админа. Принимает записи только с 10:00 до 20:00 (Екатеринбург). Ошибки уходят админу и в Google Таблицу. Вход через VK пока заготовка |
| `Calendar` | Саб-воркфлоу для агента. Проверяет часы работы, ищет занятые слоты, решает create / update / delete / get, пишет в Google Calendar и уведомляет админа в Telegram. Возвращает агенту `status`, `decision`, `message` |
| `Rag system` | Следит за папкой в Google Drive, режет документы на части, делает эмбеддинги (Gemini) и кладёт в векторную таблицу Supabase. Обновлённые и удалённые файлы тоже синхронизирует |
| `Alerts` | Каждые 30 минут: напоминание клиентам за час до записи и опрос об услуге через час после. Сообщения сохраняются в историю диалога |
| `Report` | Отчёт по диалогам в виде HTML-письма (Gmail): каждый день в 12:00 или по запросу в Telegram за любой период |
| `Error scenario` | Общий обработчик ошибок. Временные сбои и проблемы с доступами передаёт админу. Остальное разбирает AI-агент (Claude Haiku через MCP n8n), предлагает правку и применяет её только после ответа админа |

## Что нужно

- n8n (облако или self-hosted).
- Telegram-боты: клиентский, для ошибок и для отчётов. `Alerts` должен писать клиентам от того же бота, что и `Siesta`, иначе ответы на опрос не дойдут до агента.
- Google: Calendar, Sheets, Drive, Docs, Gmail.
- Supabase: Postgres с двумя таблицами (SQL ниже).
- Ключи: Google Gemini, Tavily (веб-поиск), Anthropic (в Error scenario).
- Для Error scenario: включить доступ MCP в настройках инстанса n8n и вписать выданный токен в credential Bearer Auth ноды `MCP Client`.

## Подготовка базы (Supabase)

Выполните в SQL Editor.

**1. База знаний** (`Rag system` пишет в неё, `Siesta` читает). Размерность 3072 соответствует модели эмбеддингов Gemini по умолчанию в нодах `Embeddings Google Gemini`.

```sql
create extension if not exists vector;
create extension if not exists "uuid-ossp";

create table if not exists documents (
  id text primary key default uuid_generate_v4()::text,
  content text,
  metadata jsonb,
  embedding vector(3072)
);

create or replace function match_documents(
  query_embedding vector,
  match_count int default null,
  filter jsonb default '{}'
) returns table (id text, content text, metadata jsonb, similarity float)
language plpgsql as $$
begin
  return query
  select documents.id, documents.content, documents.metadata,
         1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where documents.metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

**2. История диалогов** (память агента и источник для `Report`). Таблицу сам создаёт n8n, но без колонки `created_at`, а `Report` отбирает сообщения по времени. Добавьте колонку:

```sql
alter table n8n_chat_histories
  add column if not exists created_at timestamptz default now();
```

Сообщения, записанные до добавления колонки, получат время добавления, а не реальное.

## Установка

1. В n8n: **Import from file** для каждого JSON из папки `workflows/`.
2. Создайте credentials и выберите их в нодах с предупреждением.
3. Замените плейсхолдеры в нодах:

| Плейсхолдер | Где | Что подставить |
|---|---|---|
| `YOUR_ADMIN_CHAT_ID` | Siesta, Calendar, Error scenario | Telegram ID админа |
| `your-calendar@gmail.com` | Calendar, Alerts | Google-календарь салона |
| `admin@example.com` | Report | Почта для отчётов |
| `YOUR_SHEET_ID` | Siesta | Таблица для журнала ошибок |
| `YOUR_DRIVE_FOLDER_ID` | Rag system | Папка с документами для базы знаний |
| `YOUR_VK_CONFIRMATION_CODE`, `vk-webhook` | Siesta | Только если подключаете VK |
| `YOUR-INSTANCE.app.n8n.cloud` | Error scenario | Адрес вашего n8n |

4. В `Siesta` в ноде `Call_Calendar` выберите воркфлоу `Calendar` заново. Во всех воркфлоу, кроме `Error scenario`, укажите `Error scenario` как Error workflow (Settings воркфлоу).
5. Положите прайс и правила салона в папку Google Drive: `Rag system` сам загрузит их в базу.
6. Активируйте воркфлоу: сначала `Rag system` и `Calendar`, затем `Siesta`, `Alerts`, `Report`, `Error scenario`.

## Формат события в календаре

`Calendar` создаёт событие так: название `услуга: имя клиента`, в описании строки `Телефон: …`, `tg: <ID>` (или `vk: <ID>`) и `Услуга: …`.

- По метке `tg: ID` `Calendar` находит записи клиента при переносе и отмене.
- По ней же `Alerts` определяет, кому слать напоминание и опрос. Название услуги он берёт из названия события до двоеточия.
- События, созданные вручную или старой версией без метки `tg: ID`, напоминаний не получат.
- Если меняете формат, обновите обе стороны: `Decide` в `Calendar` и `Code in JavaScript` в `Alerts`.

## Настройка под себя

- **Часы работы** (10:00–20:00): нода `Check hours` в `Calendar` и раздел «Часы работы» в промте агента `Siesta`. Менять нужно в обоих местах.
- **Часовой пояс**: `Asia/Yekaterinburg` встречается в `Siesta`, `Calendar` и `Report` (промты, выражения времени и код).
- **Тон и правила ассистента**: системный промт в ноде `AI Agent` воркфлоу `Siesta`.

## Безопасность

Из экспортов удалены ID чатов, почты, ID таблиц и папок, адрес инстанса и коды подтверждения. Ключи и токены в JSON не хранятся: они остаются в credentials n8n.

## Статус

VK-вход пока заготовка. Работает Telegram.

![Воркфлоу записи в календарь](Calendar.png)
*Сценарий проверки свободных слотов и записи в Google Calendar*

![Пользовательский чат](Chat.png)
*Логика обработки сообщений, память и RAG-поиск*

![Синхронизация с Google Drive](Cloud.png)
*Автоматическая загрузка документов и обновлений базы знаний в векторный стор*

![Обработка ошибок и MCP-агент](Error.png)
*Система логирования ошибок и AI-агент редактора сценариев*

![Отправка отчетов на gmail](Gmail.png)
*Формирование и отправка HTML-отчетов по диалогам на почту*
