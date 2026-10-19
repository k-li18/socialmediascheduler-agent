# Social Media Scheduler for AI agents

Everything an agent needs to publish and schedule posts through [Social Media Scheduler](https://socialmediascheduler.cc): a hosted MCP server, a skill that teaches the posting workflow, and the files directories and registries read.

| File | For |
|---|---|
| `skills/social-media-scheduler/SKILL.md` | The skill: when and how to post, platform rules, errors. Works with `npx skills add` and Claude Code plugins. |
| `.mcp.json` | MCP server config (Claude Code plugin, Cursor, and anything that reads `mcpServers`). |
| `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` | Claude Code plugin and marketplace manifest. |
| `server.json` | Entry for the official MCP Registry. |

## Install

**Claude Code (plugin: MCP server + skill)**

```
/plugin marketplace add k-li18/socialmediascheduler-agent
/plugin install social-media-scheduler@social-media-scheduler
```

**Any agent that supports skills** (Claude Code, Cursor, Codex, …)

```bash
npx skills add k-li18/socialmediascheduler-agent
```

**Just the MCP server**

```bash
claude mcp add --transport http social-media-scheduler https://socialmediascheduler.cc/api/mcp
```

Cursor: add this to `~/.cursor/mcp.json`:

```json
{ "mcpServers": { "social-media-scheduler": { "url": "https://socialmediascheduler.cc/api/mcp" } } }
```

Claude.ai and ChatGPT: add a custom connector with the URL `https://socialmediascheduler.cc/api/mcp` and sign in.

The server signs in with OAuth, so no key is needed in clients that support it. Clients that don't can send `Authorization: Bearer sms_…` with a key from Dashboard → Agents & API keys.

**Terminal**

```bash
npx socialmediascheduler setup --key sms_…
npx socialmediascheduler post "Hello" --platforms linkedin,x
```

More: https://socialmediascheduler.cc/developers
