# CS Study Project

Personal computer science knowledge vault with Obsidian integration, AI-powered research, and structured note-taking with code and diagrams.

## Project Structure

```
/
├── CS/                        # CS study notes (kebab-case filenames)
├── .opencode/
│   ├── agents/               # Subagent definitions
│   │   ├── topic-researcher.md
│   │   └── topic-formatter.md
│   └── skills/               # Skills
│       ├── cs-study/         # Orchestrator — topic capture, research, formatting
│       ├── obsidian-cli/     # Vault operations via CLI
│       ├── obsidian-markdown/ # Wikilinks, callouts, Mermaid diagrams
│       ├── obsidian-bases/   # Dashboard views
│       ├── tavily-search/    # Quick web search
│       └── tavily-research/  # Deep cited research
├── .obsidian/                # Obsidian vault configuration
└── opencode.json             # OpenCode configuration
```

## Required Environment Variables

This project requires a **Tavily API key** for web research. The key must be manually exported in your shell before starting OpenCode:

```bash
export TAVILY_API_KEY=tvly-your-api-key-here
```

Get your API key from: https://tavily.com

## Skills & Subagents

### CS Study Skill

Located at: `.opencode/skills/cs-study/SKILL.md`

Helps study CS topics through:
- Topic categorization (16 CS domains)
- Web research via Tavily + Wikipedia (via `topic-researcher` subagent)
- Note creation with code snippets and Mermaid diagrams (via `topic-formatter` subagent)
- Bidirectional wikilink connections between related topics

### Subagents

1. **topic-researcher** — Searches Tavily + Wikipedia + official docs for explanations, code, and sources
2. **topic-formatter** — Formats research into structured notes with template, code, diagrams, and wikilinks

## Note Folder

All CS notes go in the `CS/` folder. Use the note template from the `cs-study` skill.

## Quick Usage

```
User: I want to learn about hash tables

cs-study:
  1. Checks CS/ for existing notes
  2. Researches via topic-researcher
  3. Creates CS/hash-tables.md with code, Mermaid diagram, and wikilinks
  4. Updates prerequisites with backlinks
```
