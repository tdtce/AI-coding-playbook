---
title: "Как включить сбор логов"
draft: true
url: /company/logging/
date: 2025-01-01
weight: 40
book_section: "Внедрение в компанию"
---

## Зачем нужен сбор логов

Внедрение новых инструментов мы хотим вести прозрачно. Чтобы лучше понимать, что привело к успеху, а что совсем не работает.

Мы начинаем со сбора базовых паттернов использования - частота и сложность запросов. На первом этапе внедрения нужно как можно раньше найти проблемы, связанные с:

- Сложностью освоения
- Плохими результатами
- AI-slop, пока отсутствуют процессы
- Неправильным использованием инструмента

Возникновение этих проблем мы можем понять по метрикам использования инструмента - если инструментом не пользуются, возможно есть проблемы с освоением или недостаточным качеством. Если инструмент используется чересчур, возможно могут возникнуть проблемы с безопасностью, с AI-slop или неправильным использованием инструмента. Все события мы будем отрабатывать в ручном режиме и искать лучшие паттерны использования.

Для сбора нужных нам метрик не подходят встроенные возможности Claude Code. В админскую панель у них на данный момент выгружается единственный лог - использовался ли в этот день инструмент или нет (обещают когда-то завести нормальную статистику в Team-план).

В связи с этим появился наш сборщик логов, который дает более развернутую информацию об использовании.

> **Важно:** Сборщик логов - это инструмент улучшения процессов и принятия решений на уровне компании, а не контроля разработчиков.

## Что собирает скрипт

Скрипт читает локальные JSONL-логи Claude Code из `~/.claude/projects`, агрегирует данные по дням и отправляет в API суточную статистику:

- количество AI-запросов
- входные, выходные и общие токены
- число web search запросов
- количество промптов
- суммарную и медианную длину промптов
- количество сессий
- количество прерываний агента
- dev_id и hostname машины

Сборщик логов не собирает чувствительную информацию - документ с которым вы работаете, текст промптов, результат работы ИИ. Предполагается запуск скрипта на локальной машине по расписанию раз в день.

## Инструкция по запуску сбора логов

### 1. Проверка работы скрипта

Запустите скрипт через Python:

`python claude_usage_client.py --dev-id {ваш логин}`

Если в логах есть ошибка, напишите нам, будем разбираться.

> **Совет:** Если вы используете два устройства для работы с Claude Code, то пропишите дополнительный параметр `--hostname {название устройства}`

### 2. Ежедневный запуск

Если ручной запуск прошел успешно, настройте ежедневное выполнение.

#### Пример для Linux через cron

Откройте crontab:

`crontab -e`

Ежедневный запуск в 12:00:

```bash
0 12 * * * /usr/bin/python3 /absolute/path/to/claude_usage_client.py --dev-id {ваш логин} >> /tmp/claude-usage.log 2>&1
```

#### Пример для Windows через Task Scheduler

Создайте задачу в Task Scheduler:

1. Create Task
2. На вкладке Triggers добавьте ежедневный запуск
3. На вкладке Actions выберите Start a program
4. В Program/script укажите путь к python.exe
5. В Add arguments укажите:

`C:\absolute\path\to\claude_usage_client.py --dev-id {ваш логин}`

Альтернатива через командную строку:

```bash
schtasks /Create /SC DAILY /TN "ClaudeUsageUpload" /TR "\"C:\Python311\python.exe\" \"C:\absolute\path\to\claude_usage_client.py\" --dev-id {ваш логин}" /ST 12:00
```

#### Пример для Mac OS через Launchd

Создать файл `~/Library/LaunchAgents/com.claude.usage.plist` и заменить параметры на корректные:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN">
<plist version="1.0">
<dict>

    <key>Label</key>
    <string>com.claude.usage</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/python3</string>
        <string>/FULL/PATH/claude_usage_client.py</string>
        <string>--dev-id</string>
        <string>{Ваш id}</string>
        <string>--hostname</string>
        <string>{название host-а}</string>
    </array>

    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>12</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>

    <key>StandardOutPath</key>
    <string>/tmp/claude-usage.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/claude-usage.log</string>

</dict>
</plist>
```

Запустить задачу через терминал в автовыполнение:

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude.usage.plist
```

При изменении задачи необходимо выполнить:

```bash
launchctl bootout gui/$(id -u)/com.claude.usage
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude.usage.plist
```
