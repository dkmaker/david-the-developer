# David the Developer 🤖

> A Claude Code plugin marketplace — AI-optimized architecture skills for non-developers, by Christian Pedersen ([@dkmaker](https://github.com/dkmaker)).

**David** is a collection of Claude Code plugins designed for people who rely on AI agents to write and maintain their code. The skills here give Claude precise, opinionated guidance so it produces consistent, simple, and maintainable output — without you needing to be a developer to get great results.

---

## 🚀 Quick Start

Add the marketplace and install the skill in one go:

```shell
/plugin marketplace add dkmaker/david-the-developer
/plugin install vanilla-spa-architecture@david-the-developer
```

That's it. Claude will now automatically apply the right architecture patterns when you ask it to build or modify your app.

---

## 📦 Available Plugins

### `vanilla-spa-architecture`

**The problem:** Next.js is a powerful framework — but it's designed for developers who understand server components, file-based routing, SSR caching, and framework-specific auth. When an AI agent maintains Next.js code, it has to track all those conventions perfectly on every single edit. One wrong `"use client"`, one missing `force-dynamic`, and things break silently.

**The solution:** This skill switches Claude's default architecture to a **React SPA + standard HTTP API** — the same app, but without the framework complexity:

- Every component is just a React component. No server/client distinction.
- Every API endpoint is a standard HTTP handler. No Next.js route format.
- Background jobs are separate timer functions. No coupling to the web process.
- The frontend is a static build deployed to a CDN. No server needed.
- Total monthly cost: **~$0** (Azure Static Web Apps free + Azure Functions free).

**When Claude activates this skill:**
- When you ask it to build a new dashboard, admin tool, or internal app
- When you ask it to refactor or migrate away from Next.js
- When you ask about architecture for an authenticated app
- When you ask it to add a new page, feature, or API endpoint

**What changes for you (the non-developer):**
- You describe what you want. Claude writes clean, consistent code.
- No need to know what "SSR" or "server components" mean.
- Claude stops needing to ask "is this server-side or client-side?"
- Deployments are simpler and faster.

---

## 🗂️ Repository Structure

```
david-the-developer/
├── .claude-plugin/
│   ├── marketplace.json        ← Marketplace catalog
│   └── plugin.json             ← Plugin manifest
├── skills/
│   └── vanilla-spa-architecture/
│       └── SKILL.md            ← The skill instructions Claude reads
├── README.md
└── LICENSE
```

---

## 🛠️ For Developers: How It Works

This repo is both the **marketplace** and the **plugin** — the marketplace entry points to `./` (the repo root) as the plugin source.

When a user installs `vanilla-spa-architecture@david-the-developer`, Claude Code:
1. Clones this repo
2. Reads `.claude-plugin/plugin.json` to discover the skill
3. Loads `skills/vanilla-spa-architecture/SKILL.md` when the user asks something architecture-related
4. Follows the DO/DON'T guidance in the skill body to produce AI-optimized code

### Contributing

Pull requests welcome. If you add a new skill:
- Create `skills/<skill-name>/SKILL.md` with proper frontmatter (`name` must match folder)
- Add the skill to the plugin listing in `.claude-plugin/marketplace.json`
- Bump the version in both `plugin.json` and `marketplace.json`

---

## 📋 Install from CLI (non-interactive)

```bash
# Add the marketplace
claude plugin marketplace add dkmaker/david-the-developer

# Install the skill
claude plugin install vanilla-spa-architecture@david-the-developer

# Reload without restarting
/reload-plugins
```

---

## 📄 License

MIT — see [LICENSE](./LICENSE)
