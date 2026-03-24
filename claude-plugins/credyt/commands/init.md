---
description: Set up your Credyt account and verify the MCP connection. Run this first before using other Credyt commands.
---

# Credyt Init

Get the user connected to Credyt so they can use `/credyt:setup`, `/credyt:verify`, and `/credyt:integrate`.

## Step 1: Check for API key in the environment

Check if `CREDYT_API_KEY` is set in the current environment by running:

```bash
test -n "$CREDYT_API_KEY" && echo "set" || echo "not set"
```

**Never** echo or print the actual API key value to the terminal.

If it **is** set, skip directly to **Step 3: Verify the MCP connection**.

If it **is not** set, continue to Step 2.

## Step 2: Configure the API key

### 2a: Collect the API key

Ask the user for their Credyt API key. It's in the **Developers** section of the Credyt dashboard. If they don't have an account, direct them to sign up at https://app.credyt.ai/api/sign-up.

**Never** echo or print the API key value to the terminal.

### 2b: Ask where to save

Ask the user where they'd like to save the key:

1. **Global** (`~/.claude/settings.json`) — applies to all your projects
2. **Project** (`.claude/settings.local.json`) — this project only (gitignored)

Accept `1` or `2`. Map `1` → `global`, `2` → `project`.

### 2c: Run the configuration script

Once you have both values, run:

```bash
./scripts/configure-api-key.sh --key "<key>" --target <global|project>
```

If the script reports an existing key (`"status": "exists"`), ask the user whether they want to overwrite it. If yes, rerun with `--force`.

**If the script exits with code 0** — the key was written or retained. Proceed to step 2d.

**If the script exits with code 2** — something went wrong (jq not installed, invalid JSON, etc.). Share the script's output with the user and help them resolve it before retrying.

### 2d: Tell the user to restart

> "Your API key has been saved to `<target from script output>`. **Please restart Claude Code** for the environment variable to take effect, then run `/credyt:init` again to complete setup."

**Stop here.** The env var won't be available until restart, so do not proceed to MCP verification.

## Step 3: Verify the MCP connection

Try calling `credyt:list_assets`. If it works, the MCP is connected and authenticated.

If connected, tell the user:

> "You're connected to Credyt. You can run `/credyt:setup` to configure your products and pricing."

And stop here — they're done.

## Step 4: Troubleshoot a failed connection

If the MCP call failed or the tool isn't available, help the user troubleshoot:

> "The MCP connection failed. A few things to check:
>
> - Have you restarted Claude Code since setting the variable?
> - Check that `CREDYT_API_KEY` is present in your settings file (`~/.claude/settings.json` or `.claude/settings.local.json`) under the `env` block.
> - The value should be in the format `Bearer key_...` — the `Bearer ` prefix is required.
>
> Would you like me to check the settings files for you, or would you prefer to re-enter your API key?"

**Never** echo or print the API key value to the terminal. If the user wants to re-enter their key, go back to **Step 2a**. If they want you to check the files, read the relevant settings file and confirm whether `CREDYT_API_KEY` is present and correctly formatted (without outputting the full key — just confirm the format looks right, e.g. "starts with `Bearer key_` and is N characters long").
