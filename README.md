# Agent Skills

Reusable, drop-in skills for AI coding agents — Kimi Code, Claude Code, OpenCode, Codex, and others.

## What are skills?

Skills are self-contained instruction files (`.md`) that give agents new capabilities. Drop them into your agent's skills directory and they're immediately available.

## Installation

```bash
# Clone the repo
git clone https://github.com/arrrrny/agent-skills.git ~/.agents/skills-public

# Symlink the skills you want into your agent's skill directory
ln -s ~/.agents/skills-public/skills/commit-and-pr ~/.agents/skills/commit-and-pr
ln -s ~/.agents/skills-public/skills/triage ~/.agents/skills/triage
# ... etc
```

Or copy individual skill directories:

```bash
cp -r skills/commit-and-pr ~/.agents/skills/
cp -r skills/triage ~/.agents/skills/
```

## Available Skills

| Skill | Purpose |
|-------|---------|
| `apply-fixes` | Apply PR review comment fixes automatically |
| `changelog` | Update changelogs in SEM format |
| `changelog-manager` | Manage CHANGELOG.md files across projects |
| `coderabbit` | CodeRabbit-style PR review |
| `commit-and-pr` | Commit, open PR, merge, pull — full cycle |
| `docs` | Generate project documentation |
| `find-skills` | Discover and install new agent skills |
| `forklift` | Multi-feature batch orchestration |
| `git-commit` | Conventional commit from diff |
| `git-patch` | Generate git patches from PRs |
| `insights` | Extract and persist session insights |
| `lean-request` | Strip HTTP requests to minimal reproducible form |
| `palletjack` | Autonomous single-feature delivery |
| `patch-upstream` | Fork + upstream sync automation |
| `path-dependencies` | Update pubspec to local path deps |
| `pr-from-patch` | Apply external patches as PRs |
| `publish-manager` | Orchestrate release workflows |
| `readme-generator` | Generate professional READMEs |
| `skill-to-bash` | Convert skill logic to bash scripts |
| `spec-renumber` | Renumber feature spec directories |
| `speckit-workflow` | Self-bootstrapping SDD workflow |
| `triage` | Classify incoming work |
| `uncle-bob` | Code quality enforcement |
| `update-thread-title` | Fix thread titles |
| `zread` | Wiki-style codebase knowledge base |

## Not included (private)

These skills contain API keys, private infrastructure URLs, or business-sensitive logic and are not published:

- `daytona*` — Daytona sandbox delegation (contains API keys)
- `flutter-mock-app-ux` — Flutter mock app testing (contains SSH credentials)
- `handoff` — Session handoff (designed to capture credentials)
- `jumpstart` — Session jumpstart (contains API keys)
- `package-upgrades` — Package updates (references private registry)
- `straddle` — Business strategy (contains revenue targets)
- `update-daytona` — Daytona updates (contains API keys)

## Contributing

1. Fork this repo
2. Add your skill to `skills/<skill-name>/SKILL.md`
3. Ensure no secrets, personal paths, or private URLs
4. Open a PR

## License

MIT
