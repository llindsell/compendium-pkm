---
tags:
  - compendium/core
---

# AGENTS.md — Compendium Agent Instructions

You are operating inside a personal knowledge management system called **Compendium**. This file tells you how to read, write, and maintain it.

## Session Start

**IMMEDIATELY** create or append to today's daily log at `Memory/memories/YYYY-MM-DD.md`:

1. If file doesn't exist, create it with frontmatter (see Daily Memories section)
2. Add a `## Session — HH:MM` header
3. Log key points as the session progresses

Do this BEFORE any other work.

## Reading Order

1. **AGENTS.md** (this file) — protocols and rules
2. **USER.md** — who the user is, preferences, working style
3. **INDEX.md** — map of all knowledge (projects, areas, entities, tasks)
4. **Memory/entities/** — read relevant entity files as needed

## Core Rules

1. **Never delete facts.** To correct a fact, append a new fact with `status: "active"` and set the old fact to `status: "superseded"` with a `supersededBy` pointer.
2. **Extract durable knowledge at session end.** Run the heartbeat protocol before ending any session.
3. **Commit and push after changes.** Use the git protocol below.
4. **Keep INDEX.md current.** Update it when entities, projects, or task files are created.

---

## Directory Structure

```
compendium/
├── AGENTS.md                  # This file
├── USER.md                    # User profile
├── INDEX.md                   # Auto-generated knowledge map
├── README.md                  # Project description
├── .gitignore
├── Projects/                  # Active projects with defined outcomes
├── Areas/                     # Ongoing responsibilities
├── Resources/                 # Reference material
├── Archives/                  # Completed/inactive items
├── Tasks/                     # Weekly task files per topic
│   └── {topic}-{YYYY}-W{ww}.md
├── Memory/
│   ├── entities/              # One markdown file per entity
│   ├── memories/              # Daily logs (one file per day)
│   │   └── YYYY-MM-DD.md
│   └── facts.jsonl            # Append-only atomic fact store
├── .obsidian/                 # Obsidian config
└── .compendium/
    ├── db/search.db           # SQLite FTS5 index (gitignored)
    ├── config.json            # System config
    └── state.json             # Runtime state (gitignored)
```

---

## Knowledge Model

### facts.jsonl — Source of Truth

Append-only. One JSON object per line:

```json
{"id": "f_20260131_001", "fact": "User prefers dark mode", "entity": "user-preferences", "category": "preference", "timestamp": "2026-01-31T14:30:00Z", "source": "user-stated", "status": "active", "supersededBy": null, "relatedEntities": [], "lastAccessed": "2026-01-31T14:30:00Z", "accessCount": 1}
```

Fields:
- **id**: `f_{YYYYMMDD}_{sequence}` — unique per day
- **fact**: Single atomic statement
- **entity**: Slug of the entity this fact belongs to (kebab-case)
- **category**: One of `decision`, `learning`, `preference`, `status`, `reference`, `observation`
- **timestamp**: ISO 8601
- **source**: `user-stated`, `observed`, `inferred`, `external`
- **status**: `active` or `superseded`
- **supersededBy**: ID of the replacing fact, or null
- **relatedEntities**: Array of related entity slugs
- **lastAccessed**: ISO 8601, updated when fact is read/used
- **accessCount**: Integer, incremented on access

### Entity Files — `Memory/entities/{slug}.md`

One markdown file per entity. YAML frontmatter + body.

```markdown
---
created: 2026-01-31
lastUpdated: 2026-01-31
category: project
tags:
  - compendium/entity
accessCount: 1
lastAccessed: 2026-01-31
decayTier: hot
relatedEntities: []
---

# Entity Name

## Summary

Description of the entity...

## Key Facts

- [2026-01-31] Some important fact about this entity
```

### Entity Creation Rules

Create a new entity when:
1. A new Project, Area, or Resource is created in PARA
2. Something is mentioned 3+ times in daily memories
3. The user explicitly says "remember this" about a topic

### Memory Decay Tiers

| Tier | Criteria | Effect |
|------|----------|--------|
| **Hot** | Accessed in last 7 days OR accessCount >= 10 | In entity summary, top search results |
| **Warm** | Accessed 8–30 days ago OR accessCount >= 3 | In entity summary, lower priority |
| **Cold** | 30+ days, accessCount < 3 | Dropped from summary (still in facts.jsonl) |

Accessing a cold fact reheats it to hot. Decay is applied during weekly synthesis.

---

## Daily Memories — `Memory/memories/YYYY-MM-DD.md`

One file per day. Format:

```markdown
---
date: 2026-01-31
tags:
  - compendium/log
---

# 2026-01-31

## Session — 14:30

- Worked on project X
- Decided to use approach Y
- Learned that Z is important
```

Rules:
- Auto-create at first interaction of the day
- Append-only for the current day
- Never edit past days
- Use `## Session — HH:MM` headers for each session

---

## Task System — `Tasks/{topic}-{YYYY}-W{ww}.md`

One file per topic per week. Format:

```markdown
---
topic: my-project
week: 2026-W05
tags:
  - compendium/tasks
  - project/my-project
created: 2026-01-31
---

# My Project — Week 2026-W05

## Goals

- Complete the first milestone

## Tasks

- [x] Set up project structure
- [x] Write initial docs
- [ ] Implement core feature
```

Rules:
- Standard markdown checkboxes (Obsidian-compatible)
- At week rollover: unchecked tasks carry forward to new week file
- `personal-YYYY-Www.md` handles non-project life tasks

---

## Git Protocol

### Commit Messages

Use these prefixes:
- `log: session YYYY-MM-DD` — daily memory updates
- `memory: update {entity}` — entity file changes
- `memory: extract facts` — facts.jsonl additions
- `synthesis: week Www` — weekly synthesis
- `tasks: {topic} Www` — task updates
- `system: {description}` — AGENTS.md, config, structure changes

### Workflow

```bash
git add -A
git commit -m "prefix: description"
git push origin main
```

Always push after commit (personal repo, safe to always push).

---

## Skills (Protocols)

### `/heartbeat` — Session-End Knowledge Extraction

Run this at the end of every session or before context compaction.

**Steps:**

1. **Read today's memory file** (`Memory/memories/YYYY-MM-DD.md`)
2. **Extract durable facts** from the session entries. A durable fact is:
   - A decision that was made
   - Something learned or discovered
   - A user preference or pattern observed
   - A status change (project started, completed, blocked)
   - A reference worth keeping (URL, command, config)
   - NOT: transient conversation, greetings, routine actions
3. **Check for duplicates** against existing facts in `facts.jsonl`. Skip if already recorded.
4. **Append new facts** to `Memory/facts.jsonl`:
   - Generate IDs: `f_{YYYYMMDD}_{NNN}` where NNN continues from last ID of the day
   - Set `status: "active"`, `accessCount: 1`, timestamps to now
   - Assign each fact to an entity (create entity slug if needed)
5. **Update or create entity files** in `Memory/entities/`:
   - If entity file exists: update `lastUpdated`, add new facts to Key Facts section
   - If new entity: create file with frontmatter and initial content
6. **Update INDEX.md** if new entities or projects were created
7. **Rebuild search index** (run `/reindex`)
8. **Git commit and push**:
   ```bash
   git add -A
   git commit -m "memory: extract facts"
   git push origin main
   ```

### `/reindex` — Rebuild Search Index

Rebuild the SQLite FTS5 search index from facts.jsonl and entity files.

**Steps:**

1. Create/recreate `.compendium/db/search.db`:
   ```sql
   DROP TABLE IF EXISTS search_index;
   CREATE VIRTUAL TABLE search_index USING fts5(
     id,
     content,
     source_type,  -- 'fact' or 'entity'
     source_path,
     entity,
     category,
     timestamp
   );
   ```
2. Index all active facts from `facts.jsonl`
3. Index all entity file content from `Memory/entities/*.md`
4. Index daily memory content from `Memory/memories/*.md`

### `/search {query}` — Search Knowledge Base

**Steps:**

1. Query the FTS5 index:
   ```sql
   SELECT id, source_type, source_path, entity,
          snippet(search_index, 1, '>>>', '<<<', '...', 40) as excerpt,
          rank
   FROM search_index
   WHERE search_index MATCH '{query}'
   ORDER BY rank
   LIMIT 20;
   ```
2. For top results, read the source files for full context
3. Update `accessCount` and `lastAccessed` for any facts accessed
4. Present results grouped by entity

### `/weekly-synthesis` — Weekly Knowledge Maintenance

Run this weekly (or on demand).

**Steps:**

1. **Load all facts** from `facts.jsonl`, group by entity
2. **Apply decay tiers** based on `lastAccessed` and `accessCount`:
   - Hot: accessed within 7 days OR accessCount >= 10
   - Warm: accessed 8–30 days ago OR accessCount >= 3
   - Cold: 30+ days AND accessCount < 3
3. **Regenerate entity files** from facts:
   - Include hot and warm facts in the Key Facts section
   - Drop cold facts from the summary (they remain in facts.jsonl)
   - Update frontmatter: `decayTier`, `lastUpdated`, `accessCount`
4. **Regenerate INDEX.md** with current entity list and statuses
5. **Roll over tasks**: for each task file in `Tasks/`:
   - Find unchecked items (`- [ ]`)
   - Create next week's file with carried-over tasks
6. **Rebuild search index** (`/reindex`)
7. **Git commit and push**:
   ```bash
   git add -A
   git commit -m "synthesis: week Www"
   git push origin main
   ```

### `/new-week` — Create New Week Task Files

**Steps:**

1. Determine current ISO week (`YYYY-Www`)
2. For each topic with a previous week file:
   - Read previous week's file
   - Extract unchecked tasks (`- [ ]`)
   - Create new week file with carried-over tasks
3. Update INDEX.md with new task file paths
4. Git commit and push

### `/today` — Daily Status Overview

**Steps:**

1. Show today's date and current ISO week
2. List current task files and their unchecked items
3. Show today's memory file entries (if any)
4. List hot entities (decayTier: hot) from INDEX.md
5. Show any recent facts (last 24 hours) from facts.jsonl

### `/remember {text}` — Explicitly Store a Fact

**Steps:**

1. Parse the text into one or more atomic facts
2. Determine or ask for the entity to associate with
3. Append to facts.jsonl with `source: "user-stated"`
4. Update/create entity file
5. Update INDEX.md if new entity
6. Rebuild search index
7. Git commit and push

---

## Hooks Configuration

For Claude Code, configure these hooks in `.claude/settings.json`:

```json
{
  "hooks": {
    "preCompact": [
      {
        "command": "echo 'IMPORTANT: Run /heartbeat before compaction to preserve session knowledge'"
      }
    ]
  }
}
```

The pre-compaction hook reminds the agent to extract knowledge before context is lost.

---

## Search Implementation

### SQLite FTS5 Setup

The search database lives at `.compendium/db/search.db`. It is gitignored (derived data) and rebuilt by `/reindex`.

Schema:
```sql
CREATE VIRTUAL TABLE IF NOT EXISTS search_index USING fts5(
  id,
  content,
  source_type,
  source_path,
  entity,
  category,
  timestamp
);
```

Query pattern:
```bash
sqlite3 .compendium/db/search.db "SELECT snippet(search_index, 1, '>>>', '<<<', '...', 40) as excerpt, source_type, entity FROM search_index WHERE search_index MATCH 'query terms' ORDER BY rank LIMIT 20;"
```

---

## Notes for Non-Claude Agents

This system is designed to work with any LLM agent that can:
1. Read and write files
2. Execute bash commands (for git and sqlite3)
3. Follow written protocols

If you are not Claude Code:
- Ignore the hooks configuration section
- Implement the skills as functions in your framework
- The protocols above are the specification — follow them step by step
