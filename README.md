# Product Forge ⚙️

Autonomous product builder for Claude Code. Submit a requirement, get a verified product.

**Pipeline:** Claude Opus (Planner) → Codex gpt-5.4 (Generator) → Claude Sonnet (Evaluator)

- Trigger from Telegram, terminal, or any agent
- Automatic retry loop (max 3 rounds) with structured critique injection
- Deploys to Vercel (frontend/fullstack) or reports test results (backend/CLI)
- Supports: frontend, fullstack, backend, CLI, infra, agent

## Requirements

- Claude Code
- [codex-plugin-cc](https://github.com/openai/codex-plugin-cc) (for Codex Generator)
- ChatGPT subscription or OpenAI API key (for Codex)
- Vercel CLI (optional, for frontend deploy)
- [OpenClaw](https://openclaw.ai) (optional, for Telegram trigger)

## Install

### 1. Install this plugin

```
/plugin marketplace add muling-dev/product-forge
/plugin install product-forge@muling-dev
/reload-plugins
```

### 2. Install codex-plugin-cc (Generator)

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

### 3. (Optional) OpenClaw Telegram trigger

```bash
cp -r openclaw/skills/product-forge ~/.openclaw/workspace/skills/product-forge
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
```

## Usage

### From Claude Code terminal

```
/forge build a personal portfolio site with dark mode and a blog section
/forge create a REST API for a todo app with SQLite storage
/forge build a CLI tool that converts markdown to a formatted PDF
```

### From Telegram (via OpenClaw)

```
forge build a SaaS landing page with pricing table and waitlist signup
```

### Check status

```
/forge:status
```

## How It Works

1. **Phase 1 — Needs Analysis**: Claude identifies project type, writes `spec.md`
2. **Phase 2 — Planning**: Claude writes implementation plan with tech stack
3. **Sprint Loop** (max 3 rounds):
   - **Generator**: Codex implements features from plan + previous critique
   - **Evaluator**: Claude Sonnet runs tests, checks build, produces structured critique
4. **Phase 5 — Delivery**: Deploy to Vercel (or local), git tag, Telegram notification

All projects land in `~/projects/YYYY-MM-DD-<name>/` with git history and `.product-forge/` metadata.

## Project Structure

Each built project contains:
```
~/projects/2026-04-02-my-project/
├── .git/                    version history
├── .product-forge/
│   ├── spec.md              product spec
│   ├── plan.md              implementation plan
│   ├── contract.md          last sprint's acceptance criteria
│   ├── critique_*.md        evaluator reports
│   ├── summary.md           delivery summary
│   └── screenshots/         (frontend projects)
└── [your project code]
```

## License

MIT
