# Credyt Claude Plugin

Set up and integrate [Credyt](https://credyt.ai) — real-time billing infrastructure for AI products — directly from Claude Code. The plugin bundles an MCP server and three skills that guide you from first configuration through production integration.

## What's included

| Type | Name | What it does |
|------|------|--------------|
| Command | `/credyt:init` | Gets you connected to Credyt. Creates an account, configures the API key, and verifies the MCP connection. Run this first. |
| Skill | `/credyt:setup` | Discovers your billing model through a guided conversation, then configures products, assets, and pricing in Credyt via MCP. Runs a full end-to-end billing cycle verification automatically. |
| Skill | `/credyt:verify` | Tests the full billing cycle for a specific product — creates a test customer, funds their wallet, sends a usage event, and confirms the fee was charged correctly. Use this after making changes or to troubleshoot. |
| Skill | `/credyt:integrate` | Wires Credyt into your application code. Adds customer creation at signup, usage event tracking, balance checks, cost tracking, billing portal links, and balance display. |

The plugin also connects to the **Credyt MCP server** (`mcp.credyt.ai`), which exposes the Credyt API as tools Claude can call directly — creating products, sending events, checking wallets, and more.

## Installation

### 1. Install the plugin

In your project directory, run:

```bash
claude mcp add --from-plugin https://github.com/credyt/claude-plugin
```

Or clone this repo and install locally:

```bash
git clone https://github.com/credyt/claude-plugin
claude mcp add --from-plugin ./claude-plugin
```

### 2. Get your API key

1. Go to [app.credyt.ai/api/sign-up](https://app.credyt.ai/api/sign-up) and create an account
2. Open the **Developers** section in the dashboard
3. Copy your API key

### 3. Set the API key

**Claude Code** — add to your shell config (`.zshrc`, `.bashrc`, etc.):

```bash
export CREDYT_API_KEY="Bearer sk_your_api_key_here"
```

Then restart Claude Code to pick it up.

**Claude Desktop** — open Settings → Developer → Edit config and update the Credyt entry:

```json
{
  "mcpServers": {
    "credyt": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://mcp.credyt.ai",
        "--header",
        "Authorization:${CREDYT_API_KEY}"
      ],
      "env": {
        "CREDYT_API_KEY": "Bearer sk_your_api_key_here"
      }
    }
  }
}
```

Restart Claude Desktop after saving.

### 4. Verify the connection

Run `/credyt:init` in Claude Code. It will confirm the MCP is connected and walk you through any remaining setup.

## Usage

Start here if you're new:

```
/credyt:init
```

Once connected, configure your billing:

```
/credyt:setup
```

Test that billing works correctly:

```
/credyt:verify
```

Wire billing into your app code:

```
/credyt:integrate
```

## Resources

- **Docs**: [docs.credyt.ai](https://docs.credyt.ai)
- **Integration guide**: [docs.credyt.ai/ai-integration.md](https://docs.credyt.ai/ai-integration.md)
- **Examples**: [github.com/credyt/learn](https://github.com/credyt/learn)
- **Dashboard**: [app.credyt.ai](https://app.credyt.ai)
