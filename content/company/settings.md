---
title: "Настройки"
url: /company/settings/
date: 2025-01-01
weight: 20
book_section: "Безопасность системы"
---

## Уровни настроек

- `.claude/settings.json` — project-level настройки, они должны быть закоммичены в репу
- `.claude/settings.local.json` — личные добавления к проектным настройкам, добавьте в .gitignore
- `~/.claude/settings.json` — user-level настройки в HOME-директории
- Организационный `managed-settings.json` — дополняет локальные настройки. Скажем, если у вас есть свой deny-лист, то организационный добавится к нему. Пользователь не может снять ограничения, установленные на уровне организации
- `~/.claude.json` — личные настройки MCP, общие и проектные
- `.mcp.json` — проектные настройки MCP, они должны быть закоммичены в репу

---

## Текущие организационные настройки

```json
{
  "availableModels": [
    "sonnet",
    "opus"
  ],
  "cleanupPeriodDays": 180,
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "opus"
  },
  "model": "opus",
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Read(~/.kube/**)",
      "Read(~/.docker/**)",
      "Read(~/.config/gh/**)"
    ]
  },
  "sandbox": {
    "allowUnsandboxedCommands": false,
    "autoAllowBashIfSandboxed": false,
    "enabled": true,
    "excludedCommands": [
      "git",
      "docker",
      "docker-compose"
    ],
    "filesystem": {
      "allowWrite": [
        "//tmp"
      ],
      "denyRead": [
        "~/.ssh/id_*",
        "~/.aws/credentials"
      ],
      "denyWrite": [
        "//etc",
        "//usr/bin"
      ]
    }
  },
  "statusLine": {
    "command": "input=$(cat); rem=$(echo \"$input\" | jq -r '.context_window.remaining_percentage // empty'); [ -n \"$rem\" ] && { eff=$(echo \"$rem\" | awk '{v=100-((($1-16.5)/(100-16.5))*100); if(v<0)v=0; if(v>100)v=100; printf \"%.0f\", v}'); echo \"Context: ${eff}% (raw $((100-rem))%)\"; } || echo \"Context: waiting...\"",
    "type": "command"
  }
}
```

Подробнее изучить настройки можно [здесь](https://code.claude.com/docs/en/settings).

Рекомендую определиться с обязательными для проекта настройками и закоммитить их в репозиторий.
