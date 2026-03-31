---
name: sync-notion-playbook
description: Interactive review of Notion vs repo differences — walk through each change and decide together whether to sync to repo or update Notion
---

# Sync Notion <-> Playbook Repository

Interactive review of differences between the Notion knowledge base "AI-кодинг" and this Hugo repo. Walk through each change one by one, decide together what to do.

## Page Mapping

| Notion page ID | Notion title | Repo file |
|---|---|---|
| `322237f3-82b8-80d3-ad44-e0297372b6ca` | AI-friendly документация | `content/team/ai-friendly-docs.md` |
| `322237f3-82b8-80fd-ae64-ecfd1259e852` | Скиллы, плагины, субагенты | `content/advanced/skills-plugins-subagents.md` |
| `322237f3-82b8-802f-8312-d72437718368` | MCP и CLI | `content/advanced/mcp-vs-cli.md` |
| `322237f3-82b8-80cc-92d8-f0d0cc7d57eb` | Безопасность | `content/company/security.md` |
| `322237f3-82b8-8076-9e7b-db2bbde71c83` | Настройки | `content/company/settings.md` |
| `322237f3-82b8-8072-9d4e-d6f940f41141` | Продвинутые фишки | `content/advanced/advanced-features.md` |
| `322237f3-82b8-80fd-9be1-d7bbbf98d9b5` | Командные процессы | `content/team/team-processes.md` |
| `325237f3-82b8-8011-9bfb-ce849199f242` | Метрики внедрения | `content/company/adoption-metrics.md` |
| `326237f3-82b8-80dc-a6fe-de9e927c6a28` | Как включить сбор логов | `content/company/logging.md` |

Notion-only pages (no repo file yet):

| Notion page ID | Title |
|---|---|
| `326237f3-82b8-8010-baaa-efaff189179d` | С чего начать лидам команд |
| `325237f3-82b8-80a4-b6fb-d66cca8fed2a` | Делимся опытом |
| `325237f3-82b8-80d8-9a77-d11443a78988` | Использование других ИИ-инструментов |
| `326237f3-82b8-804f-b8d8-d0785506d577` | FAQ |
| `326237f3-82b8-8081-a969-d9d7d2d3979d` | Субъективный гайд от Жеки |

Repo-only files (most of them are inspired by Notion page 326237f382b88081a969d9d7d2d3979d):
- `content/personal/choosing-tools.md` — Типы ИИ-инструментов
- `content/personal/getting-started.md` — Начало работы с Claude Code
- `content/personal/sessions-and-modes.md` — Сессии и режимы работы
- `content/personal/best-practices.md` — Лучшие практики
- `content/advanced/model-selection.md` — Выбор моделей и экономия
- `content/team/task-workflow.md` — Работа над задачами

## Procedure

### Step 1: Gather data

**Option A (preferred): Notion ZIP export**
Ask the user to export the Notion workspace as ZIP (or check if a ZIP file already exists in the project root). Unzip to `/tmp/claude/notion_export/`. Match files to repo by Notion page ID in the filename.

**Option B: Notion MCP API**
Launch subagents to fetch data concurrently:
- **Subagent A:** For each mapped page, call `mcp__notionApi__API-get-block-children` with the page ID. For blocks with `has_children: true`, also fetch nested children. Record `last_edited_time`.
- **Subagent B:** Read all corresponding repo markdown files.

Also fetch the root Notion page (`321237f3-82b8-8033-a9bb-d7594edb7d22`) children to detect any NEW pages added since last sync.

### Step 2: Build change list

For each page pair, identify all differences:
- **ADDED_IN_NOTION** — new sections, callouts, code blocks, links in Notion not in repo
- **UPDATED_IN_NOTION** — content that changed in Notion vs repo version
- **ADDED_IN_REPO** — content in repo not in Notion (user may want to push to Notion)
- **STRUCTURAL** — reordered sections, renamed headers, content restructured or moved to a different file

Before classifying something as ADDED_IN_REPO, grep the Notion export for key phrases — the content may exist on a different Notion page or in a restructured form. Similarly, before classifying as ADDED_IN_NOTION, grep the repo for key phrases — the content may already exist in a different repo file.

Also identify:
- New Notion pages with no repo counterpart
- Changes to Notion-only pages that may now warrant a repo file

### Step 3: Interactive review (THIS IS THE KEY STEP)

**CRITICAL: Present changes STRICTLY ONE BY ONE.** Do NOT batch or group multiple changes in one message. Show exactly one change, wait for user's decision, then show the next one.

For each change:

```
---
📄 Page: [title]
📝 Change #N: [ADDED_IN_NOTION / UPDATED_IN_NOTION / ADDED_IN_REPO]

**Notion says:**
> [quote the Notion content]

**Repo says:**
> [quote the repo content, or "— отсутствует —"]

**What do we do?**
1. ✅ Добавить в репу — I'll edit the repo file
2. ✅ Добавить в Notion — I'll update the Notion page via MCP
3. ⏭️ Пропустить — skip this change
4. 📝 Переписать — rewrite the content before adding (tell me how)
---
```

Wait for user's decision before proceeding to the next change.

**Important interaction rules:**
- Never batch-apply changes without explicit approval for each one
- If the user says "все остальные в репу" or similar, you may batch the rest
- Group minor changes (typo fixes, formatting) and present them together
- If there is company-specific content in Notion (Mattermost or repository links, internal team names, etc.), never include them in the public playbook. If you are unsure if the content is company-specific, ask user. For example, examples derived from the real work experience might be okay to include
- Show the actual markdown diff / edit you plan to make before applying
- Record all skipped changes to `.claude/sync_skipped.md` with page name and brief description, so they don't come up again in future syncs

### Step 4: Apply approved changes

For each approved change:
- **"Добавить в репу"**: Edit the repo file. Preserve Hugo frontmatter. Adapt Notion callouts to markdown blockquotes or Hugo shortcodes.
- **"Добавить в Notion"**: Use `mcp__notionApi__API-update-a-block` or `mcp__notionApi__API-patch-block-children` to push content to Notion.
- **"Переписать"**: Apply the user's rewrite, then push to the chosen destination.

### Step 5: Summary

After all changes are reviewed, output:

```
## Sync Summary [date]

Applied to repo: N changes
Applied to Notion: N changes
Skipped: N changes
New pages to create: [list]

Suggested follow-ups:
- [any remaining items]
```

## Important notes
- Neither source is automatically "right" — the user decides per change
- The repo has educational content (personal/, model-selection) not in Notion — this is intentional, don't flag it as missing
- Content is in Russian
- Preserve Hugo frontmatter when editing repo files
- Some Notion content is company-internal — flag for review before adding to repo
- When pushing to Notion, respect existing block structure and formatting
