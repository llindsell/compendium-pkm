# Compendium

A personal knowledge management system that learns while you work.

Talk to your AI agent. It remembers everything. No forms, no tagging, no filing.

---

## What is this?

Compendium is a plain-file knowledge system designed to be operated by AI agents (Claude Code, etc.) or browsed in [Obsidian](https://obsidian.md). It combines:

- **PARA organization** (Projects, Areas, Resources, Archives)
- **Atomic facts** stored in an append-only JSONL file
- **Memory decay** so recent knowledge stays prominent and old knowledge fades (but never disappears)
- **Full-text search** via SQLite FTS5
- **Automatic git versioning** after every knowledge extraction
- **Weekly task tracking** with automatic rollover

The system is **agent-portable** -- instructions live in `AGENTS.md`, not tied to any specific tool. Point any file-capable LLM agent at this repo and it knows how to operate.

## Principles

- **Never forget, never delete.** Facts are superseded, not erased. You can always trace back.
- **Lazy by design.** The system does the work. You just talk to your agent.
- **Plain files over databases.** Markdown and JSON. Readable by humans, agents, and Obsidian.
- **Small context, big recall.** Many small files beat one massive doc. Agents load only what's relevant.
- **The map is not the territory.** `INDEX.md` tells agents where to look. They don't scan everything.
- **Compound learning.** Every conversation makes the system smarter.
- **Memory decays like memory should.** Recent and frequent stuff stays hot. Old stuff fades but never disappears.
- **Git is the backup.** Every change committed and pushed. Version history for free.

## How it works

1. **You talk to your agent.** That's it.
2. **The agent writes to today's memory.** Timestamped entries in `Memory/memories/YYYY-MM-DD.md`.
3. **Heartbeat fires at session end.** The agent scans the log, extracts durable facts -- decisions, learnings, status changes, preferences.
4. **Facts get stored twice.** Appended to `facts.jsonl` (structured, searchable) and written into entity markdown files (human-readable).
5. **New entities are created automatically.** Mentioned 3+ times or explicitly important? Gets its own file.
6. **INDEX.md regenerates.** The map updates so the next session knows what exists.
7. **Git commits and pushes.** Every heartbeat ends with a commit.
8. **Weekly synthesis runs.** Entity summaries rewritten from active facts. Decay tiers applied. Unchecked tasks roll forward.
9. **Search finds what you forgot.** SQLite FTS5 indexes everything.
10. **The cycle repeats.** Every session, the system gets a little smarter.

## Quick start

### 1. Create your own copy

Click **"Use this template"** on GitHub to create your own repository. This gives you a clean starting point with your own git history.

```bash
git clone git@github.com:YOUR_USERNAME/YOUR_REPO_NAME.git
cd YOUR_REPO_NAME
```

Edit `USER.md` with your name, preferences, and working style. This helps the agent tailor its behavior to you.

> **Getting system updates:** To pull in future improvements to the compendium system, add the upstream remote and merge selectively:
> ```bash
> git remote add upstream git@github.com:llindsell/compendium-pkm.git
> git fetch upstream
> git merge upstream/main --allow-unrelated-histories
> ```
> Resolve any conflicts in your personal files (USER.md, etc.) and commit.

### 2. Point your agent at it

**Claude Code:**
Open the compendium folder as your working directory. Claude Code will read `AGENTS.md` automatically and follow the protocols.

**Other agents:**
Tell your agent to read `AGENTS.md` first. It contains all the rules, directory layout, and skill definitions the agent needs to operate the system.

### 3. Start working

Just have a conversation. The agent will:
- Log the session to today's daily memory file
- Extract durable facts at session end (`/heartbeat`)
- Commit and push changes to git

### 4. Use the built-in skills

| Skill | What it does |
|-------|-------------|
| `/heartbeat` | Extract facts from today's session, update entities, commit & push |
| `/search {query}` | Full-text search across all knowledge |
| `/reindex` | Rebuild the SQLite FTS5 search index |
| `/weekly-synthesis` | Rewrite entity summaries, apply memory decay, roll over tasks |
| `/new-week` | Create next week's task files with carried-over items |
| `/today` | Show daily status: tasks, recent memories, hot entities |
| `/remember {text}` | Explicitly store a fact |

### 5. Browse in Obsidian

Open the folder as an Obsidian vault. All files use standard markdown with YAML frontmatter. The `.obsidian/` directory is gitignored so each device can have its own Obsidian config.

## Directory structure

```
compendium/
├── AGENTS.md                  # Agent instructions and protocols
├── USER.md                    # Your profile and preferences
├── INDEX.md                   # Auto-generated knowledge map
├── README.md                  # This file
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
└── .compendium/
    ├── db/search.db           # SQLite FTS5 index (gitignored)
    ├── config.json            # System config
    └── state.json             # Runtime state (gitignored)
```

## Knowledge model

### Facts (facts.jsonl)

The source of truth. Append-only, one JSON object per line:

```json
{
  "id": "f_20260201_001",
  "fact": "User prefers dark mode in all editors",
  "entity": "user-preferences",
  "category": "preference",
  "timestamp": "2026-02-01T10:00:00Z",
  "source": "user-stated",
  "status": "active",
  "supersededBy": null,
  "relatedEntities": [],
  "lastAccessed": "2026-02-01T10:00:00Z",
  "accessCount": 1
}
```

Facts are never deleted. Old facts are marked `"status": "superseded"` with a pointer to the replacing fact.

### Entities (Memory/entities/)

One markdown file per important thing -- projects, people, topics, tools. Contains a summary and key facts section. Automatically created when something is mentioned 3+ times or explicitly flagged.

### Memory decay

| Tier | Criteria | Effect |
|------|----------|--------|
| Hot | Accessed in last 7 days OR accessCount >= 10 | In entity summary, top search results |
| Warm | Accessed 8-30 days ago OR accessCount >= 3 | In entity summary, lower priority |
| Cold | 30+ days, accessCount < 3 | Dropped from summary (still in facts.jsonl forever) |

Accessing a cold fact reheats it to hot.

## PARA method

Everything in your life fits one of four buckets:

- **Projects** -- Short-term efforts with a defined outcome (e.g., "Launch website", "File taxes")
- **Areas** -- Ongoing responsibilities with no end date (e.g., "Health", "Finances", "Career")
- **Resources** -- Topics of interest or reference material (e.g., "Machine learning", "Recipes")
- **Archives** -- Completed projects or inactive areas

Projects move to Archives when done. Areas persist. Resources accumulate.

## Requirements

- An LLM agent that can read/write files and run bash commands (Claude Code recommended)
- `git` installed and configured
- `sqlite3` available (for search -- comes pre-installed on macOS and most Linux)
- [Obsidian](https://obsidian.md) (optional, for browsing)

## License

MIT
