---
categories: [Technical]
math: true
---

# Understanding MCP Servers, Skills, Commands, and Knowledge Work Plugins

A reference guide to the layered architecture behind how Claude connects to external tools and uses domain expertise to accomplish tasks.
---

## 1. MCP Servers: The Bare-Bones Connection

### The Big Picture

Think of MCP as a **universal adapter**. Just like a USB-C cable lets your laptop talk to any monitor without knowing the monitor's internals, MCP lets Claude talk to any app without knowing that app's API.

```
Claude  ←→  MCP Server  ←→  Slack
         (the adapter)     (the app)
```

Someone else already built the Slack MCP server (the adapter). Your job is just to tell Claude: *"here's where the adapter lives, use it."* That's what the `.mcp.json` file does.

### How Claude Talks to an MCP Server

An important clarification: Claude does **not** send natural language to the MCP server. The communication is entirely structured.

**Step 1 — The handshake.** When Claude first connects to an MCP server, the server declares its capabilities by sending back a list of structured tools — essentially function signatures with typed parameters:

```
Tool: search_messages
  Parameters:
    - channel (string)
    - query (string)
    - after (date)

Tool: post_message
  Parameters:
    - channel (string)
    - text (string)

Tool: list_channels
  Parameters: none
```

No natural language involved — just typed inputs and outputs.

**Step 2 — Claude does the translation.** Claude itself is the translator between your natural language and structured tool calls. When you say "what did my team discuss today?", Claude reasons:

```
User wants recent Slack messages.
I have a tool called search_messages.
I need to call it with after = today's date.
But which channel? I should call list_channels first.
```

Then Claude makes a structured call — not a sentence, but something like:

```json
{
  "tool": "search_messages",
  "parameters": {
    "channel": "engineering",
    "after": "2026-04-04"
  }
}
```

**Step 3 — The MCP server is just a thin wrapper.** It receives that structured call and translates it to the app's actual API:

```
Incoming from Claude:
  search_messages(channel="engineering", after="2026-04-04")

Outgoing to Slack's API:
  POST https://slack.com/api/conversations.history
  Headers: Authorization: Bearer xoxb-...
  Body: { channel: "C04ABC123", oldest: "1743724800" }
```

The MCP server handles the boring stuff: authentication tokens, converting channel names to Slack's internal IDs, formatting dates into Unix timestamps, pagination. But it's not doing any AI or language understanding — it's just plumbing.

**The full accurate picture:**

```
You: "What did my team discuss today?"
    ↓
Claude (the brain): understands intent, picks the right tools
    ↓
Claude → MCP Server: structured call, like a function call
    ↓
MCP Server → Slack API: HTTP request with auth tokens
    ↓
Slack API → MCP Server: raw JSON response
    ↓
MCP Server → Claude: cleaned-up results
    ↓
Claude (the brain): summarizes, formats, applies your skill rules
    ↓
You get a clean digest
```

The intelligence is all on Claude's side. The MCP server is closer to a **database driver** than an AI — it just knows how to speak Slack's API so Claude doesn't have to. That's why you don't need to understand MCP internals to use it. Someone else wrote the driver. You just plug it in.

### What's Behind the Adapter

An MCP (Model Context Protocol) server is how an AI agent issues API calls to gather information from an external service. It is the lowest level of the stack — pure plumbing with no opinions about how the data should be used.

For example, Slack's official MCP server is hosted at `mcp.slack.com/mcp`. When Claude connects to it, it gains access to a set of **tools** — concrete API operations like:

- `slack_search_public` — search messages across public channels
- `slack_search_public_and_private` — search across all channels including private and DMs
- `slack_read_channel` — read message history from a specific channel
- `slack_read_thread` — read an entire thread and its replies
- `slack_send_message` — send a message to a channel or user

Each tool has a defined name, input parameters, and output format. Claude calls these tools by name when it needs to interact with Slack. The MCP server handles authentication, rate limiting, and API translation — Claude just sees a clean interface.

### Connectors vs MCP Servers

On **claude.ai** (the web interface), Anthropic wraps MCP servers in a managed layer called **Connectors**. You toggle them on/off in Settings → Connectors. The OAuth token is stored at the account level, which means if multiple people share a Claude account, they all share the same Slack identity. This is a privacy risk in shared environments.

On **Claude Code** (the CLI), you connect to MCP servers directly. The configuration and OAuth token live in your personal `~/.claude.json`, scoped to your Linux user account. Each person on a shared server has their own isolated connection.

The underlying protocol is the same — MCP is an open standard. The difference is just who manages the connection lifecycle.

### Adding the Slack MCP Server to Claude Code

The easiest way is using Slack's official plugin:

```bash
git clone https://github.com/slackapi/slack-mcp-plugin.git
cd slack-mcp-plugin
claude --plugin-dir ./
```

The MCP server connection is automatically configured when the plugin loads. Authentication happens lazily — the first time Claude actually tries to call a Slack tool, you'll be prompted to authenticate via OAuth.
You can also do it by running the command below in Claude code.

```bash
/mcp
```

Alternatively, you can skip the plugin and add the connection manually:

```bash
claude mcp add-json slack \
  '{"type":"http","url":"https://mcp.slack.com/mcp","oauth":{"clientId":"1601185624273.8899143856786","callbackPort":3118}}' \
  --scope user
```

*(Configuration from [Slack's official MCP plugin docs](https://docs.slack.dev/ai/slack-mcp-server/connect-to-claude/))*

The `--scope user` flag ensures the configuration is stored per-user in `~/.claude.json`, not per-project. On a remote server accessed via SSH, you may need to forward port 3118 for the OAuth callback:

```bash
ssh -L 3118:localhost:3118 your-server
```

---

## 2. Skills: Domain Knowledge That Claude Draws On

A **skill** is a markdown file that encodes domain expertise — best practices, step-by-step workflows, terminology, and guidance for how to approach a specific kind of task.

Skills live in a `skills/` directory and have a metadata header with a `name` and `description`:

```markdown
---
name: product-brainstorming
description: Brainstorm product ideas, explore problem spaces, and challenge assumptions...
---

# Product Brainstorming Skill

You are a sharp product thinking partner...
```

The key property of skills is that they **auto-activate** — Claude draws on them automatically when the conversation topic is relevant, without the user explicitly requesting it.

However, skills can **also be invoked explicitly** by name, functioning much like a command. The line between skills and commands is blurring; Anthropic's own Cowork UI now presents commands and skills as a single "Skills" concept.

### Skills Can Reference Other Skills

Skills can refer to more specialized skills, enabling composition:

- `design/research-synthesis` → *"See the **user-research** skill for research methods and analysis frameworks"*
- `engineering/architecture` → *"See the **system-design** skill for detailed frameworks on scalability analysis"*
- `data/analyze` → *"use the **sql-queries** skill for dialect-specific best practices"*
- `operations/status-report` → *"See the **risk-assessment** skill for risk matrix frameworks"*

This allows domain knowledge to be modular — a high-level skill handles the workflow while delegating specialized concerns to focused sub-skills.

### Skills Reference MCP Tool Names Directly

Skills are aware of the exact tools exposed by MCP servers. For example, the Slack search skill from the knowledge-work-plugins repository instructs Claude to:

- *"Use `slack_read_thread` to get the full thread context for any threaded message"*
- *"Use `slack_read_channel` with oldest/latest timestamps to read surrounding messages"*
- *"Use `slack_read_user_profile` to identify who a user is when their ID appears"*

This is what makes skills powerful — they are essentially a prompt engineering layer that teaches Claude *how* and *when* to use specific MCP tools effectively.

---

## 3. Commands: Explicit Orchestration

A **command** is a markdown file in a `commands/` directory that defines an explicitly invocable workflow, triggered by a slash command like `/brainstorm` or `/enterprise-search:search`.

Commands serve as **orchestrators** — they define a step-by-step sequence that coordinates multiple skills and MCP tools:

1. Understand the user's input
2. Pull context from connected tools (Slack, email, analytics, etc.)
3. Apply domain knowledge by referencing one or more skills
4. Generate output
5. Offer follow-up actions (which may invoke other commands)

For example, the `brainstorm` command in the product-management plugin:
- Step 1–2: Understands the starting point and pulls context from `~~chat`, `~~knowledge base`, `~~product analytics` via MCP
- Step 3: Delegates to the `product-brainstorming` skill for frameworks and session structure
- Step 5: Offers follow-ups like `/write-spec`, `/competitive-brief`, `/synthesize-research`

### Commands vs Skills: The Distinction

| | Skills | Commands |
|---|---|---|
| Trigger | Auto-fire when relevant **and** invocable by name | Only fire on explicit invocation |
| Purpose | Encode domain knowledge and best practices | Orchestrate a workflow across tools and skills |
| MCP awareness | Reference specific tool names and other skills | Reference specific tool names and other skills |
| Location | `skills/` directory | `commands/` directory |

In practice, most plugins in Anthropic's repository use **only skills** and skip the `commands/` directory entirely. Commands are an optional structure for when you want a dedicated orchestration file that is clearly separated from the underlying domain knowledge.

---

## 4. Knowledge Work Plugins: The Full Bundle

A **Knowledge Work Plugin** packages everything together — MCP server connections, skills, and commands — into a single installable unit for a specific job function.

```
plugin-name/
├── .claude-plugin/plugin.json   # Manifest (name, version, description)
├── .mcp.json                    # MCP server connections
├── commands/                    # Slash commands (optional)
└── skills/                      # Domain knowledge
```

The plugin's `.mcp.json` declares which MCP servers the plugin needs. The skills then reference the exact tool names exposed by those servers, teaching Claude how to use them effectively for the plugin's domain.

### Plugins Can Connect to Multiple MCP Servers

A single plugin often needs data from multiple sources. For example, the **enterprise-search** plugin connects to Slack, email, cloud storage, wikis, and project trackers simultaneously. When you ask *"What did we decide about the API redesign?"*, the plugin:

1. Searches `~~chat` (Slack) for relevant threads
2. Searches `~~email` for follow-up messages
3. Searches `~~cloud storage` for design documents
4. Synthesizes results from all sources into a single coherent answer with attribution

Each of those sources is a separate MCP server connection declared in `.mcp.json`, and the plugin includes skills that know how to query each one and combine the results. In practice, wiring up multiple adapters is just adding entries:

```json
{
  "mcpServers": {
    "slack":  { "type": "url", "url": "https://mcp.slack.com/mcp", ... },
    "gmail":  { "type": "url", "url": "https://gmail.mcp.claude.com/mcp", ... },
    "notion": { "type": "url", "url": "https://notion-mcp.example.com/sse", ... }
  }
}
```

### The Layered Architecture

```
┌─────────────────────────────────────────────────┐
│              Knowledge Work Plugin              │
│  (e.g., enterprise-search, sales, productivity) │
├─────────────────────────────────────────────────┤
│  Commands (optional)                            │
│  Explicit orchestration workflows               │
│  e.g., /enterprise-search:search                │
│  e.g., /enterprise-search:digest                │
├─────────────────────────────────────────────────┤
│  Skills                                         │
│  Domain knowledge, best practices, workflows    │
│  Auto-fire on relevance + invocable by name     │
│  Can reference other skills for composition     │
│  Reference MCP tool names directly              │
├─────────────────────────────────────────────────┤
│  MCP Servers (.mcp.json)                        │
│  Bare API connections to external services      │
│  e.g., Slack, Gmail, Notion, Jira, Snowflake    │
│  Each exposes named tools Claude can call       │
└─────────────────────────────────────────────────┘
```

### Installing a Plugin in Claude Code

```bash
# Add the marketplace
claude plugin marketplace add anthropics/knowledge-work-plugins

# Install a specific plugin
claude plugin install enterprise-search@knowledge-work-plugins
```

Once installed, skills fire automatically when relevant, and slash commands (whether defined in `commands/` or via skill names) are available in your session.

### Customizing Plugins

Everything is file-based — markdown and JSON, no code. You can:

- **Swap connectors** — edit `.mcp.json` to point at your tool stack
- **Add company context** — drop your terminology and processes into skill files
- **Adjust workflows** — modify skill instructions to match how your team works
- **Build new plugins** — follow the same directory structure

---

*This document was compiled from a conversation exploring the Slack MCP server, Claude Code configuration, and Anthropic's knowledge-work-plugins repository (github.com/anthropics/knowledge-work-plugins).*
