---
title: "Настройки"
draft: false
url: /company/settings/
date: 2025-01-01
weight: 800
book_section: "Внедрение в компанию"
---

## Уровни настроек

- `.claude/settings.json` — project-level настройки, они должны быть закоммичены в репу
- `.claude/settings.local.json` — личные добавления к проектным настройкам, добавьте в .gitignore
- `~/.claude/settings.json` — user-level настройки в HOME-директории
- Организационный `managed-settings.json` — дополняет локальные настройки. Скажем, если у вас есть свой deny-лист, то организационный добавится к нему. Пользователь не может снять ограничения, установленные на уровне организации
- `~/.claude.json` — личные настройки MCP, общие и проектные
- `.mcp.json` — проектные настройки MCP, они должны быть закоммичены в репу

---

## Пример базовых настроек организации

Что они включают из важного:

- SETTINGS_CHECK — для проверки, что настройки применились, у пользователя выскочит предупреждение, что ему пытаются прокинуть неизвестный энв
- sandbox — сэндбокс включён по умолчанию (Linux, Mac, WSL2), не позволяет удалить чего-то лишнего вне папки проекта

```json
{
  "availableModels": [
    "sonnet",
    "opus"
  ],
  "cleanupPeriodDays": 180,
  "env": {
    "SETTINGS_CHECK": "test",
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

Рекомендуем определиться с обязательными для компании и проекта настройками и закоммитить их в репозиторий.

> **Совет:** В версии 2.1.77 появилась команда `/update-config`, которая вызывает скилл, помогающий настраивать разные части по желанию пользователя — хуки, MCP, разрешения и многое другое.

## Полезные настройки

- Можно убрать надпись [Co-Authored by Claude Code](https://code.claude.com/docs/en/settings#attribution-settings), если вы не хотите её использовать для дальнейшей аналитики использования ИИ
- cleanupPeriodDays - можно дольше хранить сессии (скажем, 180 дней)
- Постоянные переменные можно прокинуть в раздел env, например:
  - CLAUDE_CODE_SUBAGENT_MODEL - можно поменять модель субагента на Opus, если у вас хорошая подписка
  - ENABLE_EXPERIMENTAL_MCP_CLI - экономит кучу контекста, Claude Code читает описания тулзов из MCP только по требованию
  - CLAUDE_CODE_NEW_INIT - новая интерактивная версия команды /init
- spinnerVerbs - можно поставить [прикольные фразочки](https://github.com/i1kazantsev/claude-code-spinner) на раздумья Клода
