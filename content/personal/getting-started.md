---
title: "Начало работы с Claude Code"
url: /personal/getting-started/
date: 2025-01-01
weight: 10
book_section: "Подходы к использованию"
---

За прошлый год я потестил достаточно большое количество ИИ-инструментов для разработки (Cursor, Claude Code, Codex, Cline, Aider, RooCode, Kilo, Augment) и даже написал [обзорный пост](https://crazyfrogspb.github.io/2025/09/22/%D0%B8%D0%B8-%D0%B8%D0%BD%D1%81%D1%82%D1%80%D1%83%D0%BC%D0%B5%D0%BD%D1%82%D1%8B-%D0%B2-%D1%80%D0%B0%D0%B7%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B5-opinionated-guide/). Какие-то отбросил почти сразу, какие-то прям поюзал и даже посравнивал на одних и тех же задачах. В итоге сейчас я отменил все подписки кроме Claude Code Max и в качестве IDE использую Zed, в котором есть нативная интеграция через [Agent Client Protocol](https://agentclientprotocol.com/get-started/introduction). Почему Claude Code?

- Главная причина - на моих субъективных ненаучных сравнениях качество намного лучше, чем у конкурентов
- Богатая эко-система плагинов и в целом это популярный развивающийся инструмент
- Адекватная цена - за 100/200 долларов я получаю практически безлимитные запросы и не парюсь о тратах

Если вы никогда не использовали CC или подобные инструменты, то рекомендую забить на гайды в стиле "запускаем 5-10 агентов параллельно" и поработать сначала в максимально простом режиме - **без плагинов**, с **одной открытой вкладкой**, можно через плагин для VS Code или в Zed. Так вы лучше поймёте сильные и слабые стороны, как лучше формулировать свои запросы и не пропустите момент, когда его нужно останавливать.

![Классика жизни](/images/0fa52e37.png)

## Если вы в России

Я использую такую схему - VPN настроен через Shadowsocks, сверху поднят [Privoxy](https://www.privoxy.org/), так как CC не заработал у меня с SOCKS5-прокси.

Схема простая, ставим Privoxy (`sudo apt-get install privoxy`), в конфиг `/etc/privoxy/config` добавляем строчку

`forward-socks5t / 127.0.0.1:1080 .`

И перезапускаем: `sudo systemctl restart privoxy`.

Для удобства у меня в `~/.bash_aliases` есть такие команды:

```bash
alias setproxy='export http_proxy=http://127.0.0.1:8118; export https_proxy=http://127.0.0.1:8118; export no_proxy=localhost,127.0.0.1; export HTTP_PROXY=$http_proxy; export HTTPS_PROXY=$https_proxy; export NO_PROXY=$no_proxy'

alias unsetproxy='unset HTTP_PROXY; unset HTTPS_PROXY; unset NO_PROXY; unset http_proxy; unset https_proxy; unset no_proxy'
```

В no_proxy можно добавить любые адреса, к которым CC должен обращаться не через VPN.

Теперь в терминале достаточно набрать setproxy и потом уже claude.

## Работа с CC в IDE

Есть плагины для популярных IDE (VSCode, PyCharm). Они позволяют удобно изучать изменения, которые сделал агент, прямо в IDE.

Ещё один вариант — скачать [Zed](https://zed.dev/download), симпатичный IDE с нативной интеграцией с CC через [ACP](https://agentclientprotocol.com/get-started/introduction).

## Что ещё почитать

- [Детальный свежий гайд](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/)
- [Безумный пайплайн создателя Claude Code](https://x.com/bcherny/status/2007179832300581177?lang=en)
- [Как CC пользуются в Anthropic](https://x.com/bcherny/status/2017742741636321619?t=rtE5PtnphMf785Z5dn4JqA&s=19)
- [Официальная документация](https://code.claude.com/docs/en/overview)
- [The Longform Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2014040193557471352)
