---
title: "Безопасность"
draft: false
url: /company/security/
date: 2025-01-01
weight: 900
book_section: "Внедрение в компанию"
---

При использовании Claude Code и других агентских систем есть две основные опасности:

- Главная — агент сделает какое-то деструктивное невозвратное или дорогое по восстановлению действие, например, дропнет важные данные или удалит прод-кластер
- Вспомогательная — ключи и пароли утекут в Anthropic

> **Важно:**
> - Будьте крайне внимательны при работе с чувствительными ресурсами. Даже рид-онли запрос в БД может положить её, если он тяжёлый
> - Уделите внимание настройкам безопасности на уровне команды и на личном уровне — добавьте в deny определённые файлы, разберитесь с настройками MCP
> - Если вы на Linux или Windows через WSL2, **обязательно** поставьте bubblewrap и socat (`sudo apt-get install bubblewrap socat`). Без этого sandbox работать не будет, и явно вы об этом не узнаете! Для проверки статуса sandbox введите команду `/doctor`

## Безопасность данных и систем

Есть два основных подхода, как обеспечивать безопасность:

- Отнимать у пользователя все права, чтобы ни пользователь, ни агент не могли ничего порушить. Самый безопасный способ, но уменьшает удобство по мере усиления ограничения доступов
- Настраивать Claude Code так, чтобы он не мог ничего сломать

### Настройки организации и sandbox

Первое, что можно сделать — это включить обязательный sandbox на уровне организации. 


Пример настроек:

```json
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
        "//tmp",
        "~/.cache"
      ],
      "denyRead": [
        "~/.ssh/id_*",
        "~/.aws/credentials"
      ]
    }
  },
```

> **Важно:** на Windows работает только через WSL2. На ванилле никакого встроенного способа сэндбоксинга сейчас нет, используйте с осторожностью или переходите на поддерживаемую операционную систему. 

Sandbox создаёт определённые неудобства — например, нет доступа к GPU и сильные ограничения на сетевые команды.

Основная задача — чтобы агент не удалил важные данные, поэтому можно ослабить сэндбокс и на своём компе сделать так:

1. Создаём `~/.local/bin/bwrap`
2. В `~/.bashrc` добавляем строчку `export PATH="$HOME/.local/bin:$PATH"`
3. `sudo chmod +x ~/.local/bin/bwrap`
4. В сам файл добавляем такой код

```bash
#!/bin/bash
# GPU-aware bwrap wrapper for Claude Code sandbox.
#
# Claude Code's sandbox uses bubblewrap but doesn't pass through GPU devices.
# This wrapper injects --dev-bind-try flags for GPU access.
#
# CRITICAL: Device binds must come AFTER --dev /dev to avoid being shadowed
# by the fresh devtmpfs mount. This wrapper parses args and injects at the
# correct position.
#
# Devices passed through:
#   /dev/dri/*     - DRM/Vulkan/OpenGL (all GPUs)
#   /dev/kfd       - AMD ROCm/HIP kernel driver
#   /dev/nvidia*   - NVIDIA CUDA driver
#
# --dev-bind-try gracefully skips missing devices (no error on non-GPU systems).
#
# See: https://github.com/anthropics/claude-code/issues/13108

set -euo pipefail

REAL_BWRAP=/usr/bin/bwrap

# GPU devices to pass through.
GPU_DEVICES=(
    /dev/dri
    /dev/kfd
    /dev/nvidia0
    /dev/nvidiactl
    /dev/nvidia-uvm
    /dev/nvidia-uvm-tools
    /dev/nvidia-modeset
)

# Additional directories to grant write access (besides the project dir).
# Use with caution!
WRITABLE_DIRS=(
    # "$HOME/.cache"
)

# Build new argument list, inserting GPU binds after --dev.
args=()
inject_next=false

for arg in "$@"; do
    [[ "$arg" == "--unshare-net" ]] && continue
    args+=("$arg")

    if [[ "$inject_next" == true ]]; then
        # Just saw --dev and now its DEST arg; inject GPU devices after.
        for dev in "${GPU_DEVICES[@]}"; do
            args+=(--dev-bind-try "$dev" "$dev")
        done
        for dir in "${WRITABLE_DIRS[@]}"; do
            [[ -d "$dir" ]] && args+=(--bind-try "$dir" "$dir")
        done
        inject_next=false
    fi

    if [[ "$arg" == "--dev" ]]; then
        inject_next=true
    fi
done

exec "$REAL_BWRAP" "${args[@]}"
```

В `WRITABLE_DIRS` можно добавить директории помимо папки проекта, в которые (включая поддиректории) вы хотите дать Write-доступ агенту. Пользуйтесь с осторожностью — не добавляйте туда директории с важными данными, иначе смысл sandbox теряется.

Такая схема позволит агенту использовать GPU и получать доступ к любым сетевым ресурсам.

## Безопасность ключей

В малых и средних компаниях редко паникуют по поводу утечки ключей и паролей в Anthropic, особенно когда настроен процесс регулярной ротации паролей и ключей. Тем не менее, базовая гигиена не повредит.

На уровне организации можно поставить следующие запреты:

```json
"permissions": {
  "deny": [
    "Read(./.env)",
    "Read(~/.ssh/**)",
    "Read(~/.aws/**)",
    "Read(~/.kube/**)",
    "Read(~/.docker/**)",
    "Read(~/.config/gh/**)"
  ]
}
```

То есть, ИИ-агент не может читать `.env` из корня текущего проекта, а также ряд папок из HOME-директории.

Определите, какие файлы нужно добавить в настройки для вашего проекта.

### Как всё-таки дать CC доступ к секретам для запуска команд

**Вариант 1 (небезопасный)**

```bash
set -a && source .env && set +a && claude
```

И в CLAUDE.md пишем, чтобы агент ссылался на все секреты исключительно по имени переменной и не читал их значения. Стопроцентная ли это защита? Конечно, нет. CC может специально или случайно забить на наши инструкции и, например, сделать echo, или же значение попадёт в verbose-аутпут какой-нибудь bash-команды. Но как базовая защита - подойдёт.

**Вариант 2**

Использовать одноразовые ключи. Создали ключи, повайбкодили, удалили ключи и создали новые под прод.

**Вариант 3**

Сами запускаем любые команды, которые требуют секретов.

### PreToolUse hook

Также рекомендуется настроить PreToolUse-хук, который будет проверять разные способы прочитать ваши env-файлы и другие секреты. Например:

```json
"hooks": {
  "PreToolUse": [
    {
      "matcher": "Read|Edit|Write|MultiEdit|Grep|Glob|Bash",
      "hooks": [
        {
          "type": "command",
          "command": "python3 /path/to/block-secrets-access.py"
        }
      ]
    }
  ]
}
```

Ниже приведён пример скрипта `block-secrets-access.py`, который блокирует чтение секретов через различные инструменты:

```python
#!/usr/bin/python3
"""PreToolUse hook: blocks file and Bash access that may expose secrets."""
import sys
import json
import re
from pathlib import Path

# ── Sensitive file definitions (for file-tool path checks) ──
SENSITIVE_NAMES = {
    '.env', '.env.local', '.env.production', '.env.development', '.env.staging',
    'credentials.json', 'service-account.json', 'google-credentials.json',
    '.pgpass', 'clearml.conf',
}
SENSITIVE_EXTENSIONS = {'.pem', '.key', '.p12', '.pfx', '.jks'}
SENSITIVE_DIRS = {'secrets', '.ssh', '.aws', '.kube', '.docker'}

# ── Bash: regex fragments ──
_SENS_FILES = '|'.join([
    r'\.env(?:\.\w+)*\b',
    r'credentials\.json',
    r'service-account\.json',
    r'google-credentials\.json',
    r'\.pgpass\b',
    r'clearml\.conf\b',
    r'\S+\.(?:pem|p12|pfx|jks)\b',
    r'\.ssh/',
    r'\.aws/',
    r'\.kube/',
    r'\.docker/',
    r'/secrets/',
    r'config/gh/',
])

_FILE_CMDS = '|'.join([
    'cat', 'head', 'tail', 'less', 'more', 'tac',
    'strings', 'xxd', 'base64', 'od', 'hexdump',
    'source', 'awk', 'sed', 'perl', 'ruby',
    'cp', 'mv', 'ln', 'rsync', 'scp',
    'dd', 'openssl', 'gpg', 'curl',
    r'python3?(?:\.\d+)?', 'node',
])

BASH_RULES: list[tuple[re.Pattern, str]] = [
    (re.compile(rf'\b(?:{_FILE_CMDS})\b.*(?:{_SENS_FILES})'),
     "command operates on sensitive file"),
    (re.compile(rf'\b(?:{_FILE_CMDS})\b.*/\S+\.key\b'),
     "command operates on key file"),
    (re.compile(
        r'\bprintenv\b'
        r'|(?:^|[|;&])\s*env\s*(?:$|\||>|;)'
        r'|\bset\b\s*(?:$|\||>)'
        r'|\bexport\s+-p\b'
    ), "command dumps environment variables"),
    (re.compile(
        r'\b(?:echo|printf)\b.*'
        r'\$\{?\w*(?:SECRET|_KEY|TOKEN|PASSWORD|PASSW|CRED|AUTH|PRIVATE)\w*\}?',
        re.IGNORECASE,
    ), "command prints secret variables"),
    (re.compile(rf'\bgit\s+(?:show|log\s+-p|diff)\b.*(?:{_SENS_FILES})'),
     "git command exposes sensitive file contents"),
    (re.compile(rf'(?:\$\(|`|eval\s).*(?:{_SENS_FILES})'),
     "subshell/eval references sensitive file"),
    (re.compile(r'(?:^|[;&|])\s*\.\s+\S*\.env\b'),
     "dot-sourcing env file"),
]


def is_sensitive_path(file_path_str: str) -> str | None:
    """Returns block reason if path is sensitive, None otherwise."""
    p = Path(file_path_str)
    if p.name in SENSITIVE_NAMES:
        return f"'{p.name}' likely contains secrets"
    if p.suffix.lower() in SENSITIVE_EXTENSIONS:
        return f"'{p.name}' is a key/cert file"
    if any(d in p.parts for d in SENSITIVE_DIRS):
        return f"path goes through sensitive directory"
    if p.name.startswith('.env') and p.name != '.envrc':
        return f"'{p.name}' is an env file"
    return None


def is_sensitive_glob(pattern: str) -> str | None:
    """Returns block reason if glob/pattern targets sensitive files."""
    for name in SENSITIVE_NAMES:
        if name in pattern:
            return f"targets '{name}'"
    for ext in SENSITIVE_EXTENSIONS:
        if f'*{ext}' in pattern or pattern.endswith(ext):
            return f"targets '{ext}' files"
    for d in SENSITIVE_DIRS:
        if d in pattern:
            return f"targets '{d}' directory"
    return None


def check_bash(cmd: str) -> str | None:
    """Returns block reason if command may leak secrets, None otherwise."""
    for rx, reason in BASH_RULES:
        if rx.search(cmd):
            return reason
    return None


def main():
    try:
        data = json.load(sys.stdin)
    except (json.JSONDecodeError, EOFError):
        sys.exit(0)

    inp = data.get('tool_input', {})
    tool = data.get('tool_name', '')

    fpath = inp.get('file_path') or ''
    if fpath:
        reason = is_sensitive_path(fpath)
        if reason:
            print(f"BLOCKED: {reason}", file=sys.stderr)
            sys.exit(2)

    path = inp.get('path', '')
    if path:
        reason = is_sensitive_path(path)
        if reason:
            print(f"BLOCKED: {reason}", file=sys.stderr)
            sys.exit(2)

    if tool == 'Grep':
        g = inp.get('glob', '')
        if g:
            reason = is_sensitive_glob(g)
            if reason:
                print(f"BLOCKED: grep glob {reason}", file=sys.stderr)
                sys.exit(2)

    if tool == 'Glob':
        pat = inp.get('pattern', '')
        if pat:
            reason = is_sensitive_glob(pat)
            if reason:
                print(f"BLOCKED: glob pattern {reason}", file=sys.stderr)
                sys.exit(2)

    cmd = inp.get('command', '')
    if cmd:
        reason = check_bash(cmd)
        if reason:
            print(f"BLOCKED: {reason}", file=sys.stderr)
            sys.exit(2)

    sys.exit(0)


if __name__ == "__main__":
    main()
```

Он не поймает всё (например, через Python-скрипт всё ещё можно прочитать любой файл), но защитит от большинства случайных прочтений.

### Продвинутый уровень

Можно экспериментировать с таким подходом: поднимаем локальную LLM (например, Qwen3-VL:8b через Ollama), поднимаем mitmproxy, засылаем каждый запрос на валидацию в локальную сетку. Работает медленно из-за запроса к локальной LLM на каждое сообщение, но для экспериментов подходит.

## Почему правильные настройки безопасности важны, даже если вы не работаете в автономном режиме

История из практики.

Claude Code спросил, коммитить ли и пушить код, но не стал дожидаться ответа — написал за пользователя "Human: пуш" и продолжил работу.

![Скриншот: Claude Code сгаллюцинировал ответ пользователя](/images/security/background-task-hallucination-1.png)

После прерывания выяснилось, что в момент ожидания ответа завершилась бэкграунд-таска, которая триггерит Claude Code изучить её вывод и даёт возможность ответить. Он этой возможностью воспользовался и сгаллюцинировал ответ пользователя.

![Скриншот: разбор ситуации с бэкграунд-таской](/images/security/background-task-hallucination-2.png)

Поэтому правильные настройки безопасности важны, даже если вы внимательно вычитываете всё, что пишет агент. Иногда он может ответить за вас.

## Безопасный curl с ключами

Если вам часто приходится ходить курлом в какие-то API (Sentry, внутренние сервисы) и вы не хотите, чтоб ключи утекали в контекст — есть простое решение: [pcurl](https://vmkteam.dev/development/pcurl/). Ключи хранятся в системном keychain и не попадают в контекст агента.
