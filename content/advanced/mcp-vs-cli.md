---
title: "MCP и CLI"
url: /advanced/mcp-vs-cli/
date: 2025-01-01
weight: 20
book_section: "MCP и инструменты"
---

[Официальная документация](https://code.claude.com/docs/en/mcp)

> **Примечание:** MCP и CLI (примеры - kubectl, psql, gh) нужны для двух вещей:
>
> - Чтобы модели не нужно было заново придумывать, как использовать тот или иной внешний источник. Вся внутренняя логика уже описана внутри MCP/CLI, агенту остаётся вызвать нужный инструмент или команду
> - Чтобы ключи и пароли не попадали в контекст агента и не улетали в облака, ведь MCP и CLI заранее авторизуются в нужной системе (или хранят внутри у себя пароли)

Многие сейчас предпочитают CLI, так как он не тратит контекст и не нужно ничего настраивать. Но нужно обязательно убедиться, что через CLI агент не получит лишних ненужных доступов.

Если вы используете MCP, то лучше либо держать их выключенными и включать под задачу, либо использовать MCP CLI (смотри ниже).

Не используйте непроверенные MCP-серверы, особенно ноунеймовые с маленьким количеством звёзд.

Можно запрещать MCP использовать определённые инструменты:

```json
{
  "permissions": {
    "deny": [
      "mcp__postgresql-mcp-cloud-statistics__pg_manage_users",
      "mcp__postgresql-mcp-cloud-statistics__pg_execute_mutation"
    ]
  }
}
```

Можно делать ещё более хитрые штуки, например, блокировать тул выполнения запроса, если в его теле есть слова DROP или DELETE. Это делается с помощью хуков:

```json
{
    "hooks": {
      "PreToolUse": [
        {
          "matcher": "mcp__postgresql-mcp-cloud-statistics__pg_execute_query",
          "hooks": [
            {
              "type": "command",
              "command": "python3 /path/to/check_pg_args.py"
            }
          ]
        }
      ]
    }
  }
```

В питоновском скрипте пишете любые проверки, которые вам нужны, например:

```python
import json
import re
import sys

# Tools that are always safe (read-only by design)
ALWAYS_ALLOW = {
    "pg_analyze_database",
    "pg_debug_database",
    "pg_get_setup_instructions",
    "pg_monitor_database",
}

# Tools that are always blocked (mutations, schema changes, user management)
ALWAYS_BLOCK = {
    "pg_execute_mutation",
    "pg_copy_between_databases",
    "pg_import_table_data",
    "pg_manage_comments",
    "pg_manage_constraints",
    "pg_manage_functions",
    "pg_manage_indexes",
    "pg_manage_rls",
    "pg_manage_schema",
    "pg_manage_triggers",
    "pg_manage_users",
}

# SQL patterns that indicate mutation
DANGEROUS_SQL = re.compile(
    r"\b(INSERT|UPDATE|DELETE|DROP|ALTER|CREATE|TRUNCATE|GRANT|REVOKE|"
    r"REPLACE|MERGE|UPSERT|COPY\s+.+\s+FROM|VACUUM\s+FULL|REINDEX)\b",
    re.IGNORECASE,
)

def deny(reason: str):
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": reason,
        }
    }))
    sys.exit(0)

def allow():
    sys.exit(0)

def get_tool_short_name(tool_name: str) -> str:
    parts = tool_name.split("__")
    return parts[-1] if parts else tool_name

def extract_sql(tool_input: dict) -> str:
    for field in ("query", "sql", "statement"):
        if field in tool_input:
            return tool_input[field]
    return ""

def main():
    data = json.load(sys.stdin)
    tool_name = data.get("tool_name", "")
    tool_input = data.get("tool_input", {})
    short_name = get_tool_short_name(tool_name)

    if short_name in ALWAYS_ALLOW:
        allow()
    if short_name in ALWAYS_BLOCK:
        deny(f"Blocked: {short_name} is not allowed in read-only mode")

    if short_name in ("pg_execute_query", "pg_execute_sql", "pg_export_table_data"):
        sql = extract_sql(tool_input)
        if not sql.strip():
            deny(f"Blocked: empty SQL in {short_name}")
        if DANGEROUS_SQL.search(sql):
            deny(f"Blocked: mutation detected in {short_name}: {sql[:80]}...")
        allow()

    deny(f"Blocked: unknown PostgreSQL tool '{short_name}'")

if __name__ == "__main__":
    main()
```

Но главное правило в любом случае остаётся таким - не давайте Claude Code доступ к CLI или MCP, у которых есть доступы, которые могут похерить что-то важное. Скажем, если вы включаете MCP к БД статистики, то обязательно используйте только read-only пользователя.

### Включение experimental MCP CLI

Есть такой прикольный вариант: добавить в настройки в env-секцию

`ENABLE_EXPERIMENTAL_MCP_CLI=1`

Таким образом Claude будет запрашивать инфу об инструментах сервера только когда ему это нужно. Это сильно экономит контекст, некоторые MCP отжирают до 20 тысяч токенов только на описании инструментов.

![MCP CLI](/images/mcp-cli.png)

---

## Уровни настройки MCP

Local и user серверы хранятся в `~/.claude.json`.

Разница в том, что local-серверы распространяются на конкретный проект, а user — на все проекты.

Project-серверы хранятся в `.mcp.json` в корне проекта. Этот файл можно закоммитить в репозиторий команды. Таким образом, у всех пользователей будут одинаково настроенные MCP.

> **Внимание:** Ни в коем случае не добавляйте в этот файл секреты. Используйте энв-переменные.

---

## Примеры настройки MCP

### Notion

Кушает кучу токенов просто на описании инструментов.

```json
"notionApi": {
  "command": "docker",
  "args": [
    "run",
    "--rm",
    "-i",
    "-e",
    "NOTION_TOKEN",
    "mcp/notion"
  ],
  "env": {
    "NOTION_TOKEN": "API_TOKEN"
  }
}
```

### Postgres

MCP для доступа к БД.

```json
"postgresql-mcp-full-statistics": {
  "command": "npx",
  "args": [
    "@henkey/postgres-mcp-server",
    "--connection-string",
    "postgresql://USER:PASSWORD@host:port/postgres"
  ]
}
```

Целиком файл бы выглядел так с настройкой через энв-переменные:

```json
{
    "mcpServers": {
      "notionApi": {
        "command": "docker",
        "args": [
          "run",
          "--rm",
          "-i",
          "-e",
          "NOTION_TOKEN",
          "mcp/notion"
        ],
        "env": {
          "NOTION_TOKEN": "${NOTION_TOKEN}"
        }
      },
      "postgresql-mcp-full-statistics": {
        "command": "npx",
        "args": [
          "@henkey/postgres-mcp-server",
          "--connection-string",
          "${PG_FULL_STATS_CONNECTION_STRING}"
        ]
      }
    }
  }
```

Энв-переменные можно положить в свой settings.json в секцию env.
