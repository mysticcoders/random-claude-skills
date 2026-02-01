# Random Claude Skills

A collection of community-shared skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## What are Skills?

Skills are markdown files that extend Claude Code's capabilities with specialized workflows, tool permissions, and domain knowledge. They're invoked with slash commands like `/transcript` or `/news-briefing`.

## Installation

### Quick Install (Recommended)

Use [OpenSkills](https://github.com/anthropics/openskills) to install directly from GitHub:

```bash
npx openskills install mysticcoders/random-claude-skills:transcript
```

### Manual Install

Clone and copy the skill folder:

```bash
git clone https://github.com/mysticcoders/random-claude-skills.git
cp -r random-claude-skills/transcript ~/.claude/skills/
```

### Usage

After installation, restart Claude Code or start a new session, then invoke the skill:

```
/transcript https://youtube.com/watch?v=example
```

## Available Skills

| Skill | Command | Description |
|-------|---------|-------------|
| [transcript](./transcript/) | `/transcript <URL>` | Transcribe video/audio URLs to markdown notes using yt-dlp and parakeet-mlx. Generates summaries, key insights, and wiki-links to related notes. |

## Skill Structure

Each skill lives in its own folder with a `SKILL.md` file:

```
skill-name/
└── SKILL.md
```

The `SKILL.md` file contains:
- **Frontmatter**: name, description, and allowed-tools
- **Instructions**: Workflow, configuration, and usage details

Example frontmatter:
```yaml
---
name: my-skill
description: |
  What this skill does.
  MANDATORY TRIGGERS: "keyword1", "keyword2", "/my-skill"
allowed-tools: Bash(specific-command *), Read, Write
---
```

## Contributing

Have a useful skill to share? Contributions are welcome.

1. Fork this repository
2. Create a folder for your skill with a `SKILL.md` file
3. Follow the skill structure above
4. Submit a pull request

### Guidelines

- Keep skills focused on a single purpose
- Document prerequisites and installation steps
- Include usage examples
- Handle errors gracefully with helpful messages
- Use `AskUserQuestion` for first-run configuration when needed

## License

MIT
