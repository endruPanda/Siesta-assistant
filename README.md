# Сиеста — AI-ассистент для салона массажа

Набор из 6 воркфлоу [n8n](https://n8n.io): чат-бот «Сиеста» для салона «Здоровье ручной работы» записывает клиентов в Google Calendar, отвечает по прайсу из базы знаний, напоминает о визитах, собирает отзывы, шлёт отчёты и сам разбирает ошибки.

## Как это устроено

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

- n8n (облако или self-hosted); для Error scenario включить доступ MCP в настройках инстанса
- Telegram-боты: клиентский, для ошибок и для отчётов
- Google: Calendar, Sheets, Drive, Docs, Gmail
- Supabase: векторная таблица `documents`; Postgres с таблицей `n8n_chat_histories` (создаётся памятью агента)
- Ключи: Google Gemini, Tavily (веб-поиск), Anthropic (в Error scenario)

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

4. В `Siesta` и `Calendar` нужный Calendar-воркфлоу выберите заново в ноде `Call_Calendar`, а в настройках каждого воркфлоу укажите `Error scenario` как Error workflow.
5. Положите прайс и правила салона в папку Google Drive: `Rag system` сам загрузит их в базу.
6. Активируйте воркфлоу.

## Безопасность

Из экспортов удалены ID чатов, почты, ID таблиц и папок, адрес инстанса и коды подтверждения. Ключи и токены в JSON не хранятся: они остаются в credentials n8n.

## Статус

VK-вход и ветка VK в `Alerts` пока заготовки. Работает Telegram.
