# MCP Agent Chat

A terminal chat assistant that can **browse the web, search Airbnb listings, and run web searches**. It connects a Groq LLM to several [Model Context Protocol (MCP)](https://modelcontextprotocol.io) servers using [`mcp-use`](https://github.com/mcp-use/mcp-use) and LangChain.

The agent remembers the conversation, so you can ask follow-up questions.

```
You: open google.com and then open makemytrip.com, tell me the page titles

Assistant:
- Google: "Google" (https://www.google.com)
- MakeMyTrip: "MakeMyTrip - #1 Travel Website ..." (https://www.makemytrip.com)
```

## How it works

```
 you ──► app.py (MCPAgent + Groq LLM)
             │  picks a tool, calls it, reads the result, repeats
             ▼
        MCPClient  ── reads browser_mcp.json, starts each server with npx
             │
   ┌─────────┼──────────────────┐
   ▼         ▼                  ▼
Playwright  Airbnb          DuckDuckGo
(real browser) (listing search)  (web search)
```

Each MCP server is a small local program that `npx` starts. It talks to `app.py` over stdin/stdout. No web server or UI is involved, except that Playwright opens a visible Chromium window that it controls.

| Server | Package | What the agent can do |
|---|---|---|
| `playwright` | `@playwright/mcp` | Open pages, click, type, fill forms, take screenshots |
| `airbnb` | `@openbnb/mcp-server-airbnb` | Search listings and get listing details |
| `duckduckgo-search` | `duckduckgo-mcp-server` | Run web searches |

## Requirements

- Python 3.13+
- [uv](https://docs.astral.sh/uv/)
- Node.js 18+ (for `npx`)
- A free [Groq API key](https://console.groq.com/keys)

## Setup

```bash
git clone <your-repo-url>
cd <repo-folder>

uv sync                     # install Python dependencies
cp .env.example .env        # then put your Groq key in .env
```

`.env`:

```
GROQ_API_KEY=your_groq_api_key_here
```

If Playwright asks for a browser the first time it runs, install one with:

```bash
npx playwright install chromium
```

## Run

```bash
uv run app.py
```

In the chat:

- Type a request, for example `find Airbnb listings in Paris for 2 adults next weekend`
- Type `clear` to reset the conversation memory
- Type `exit` or `quit` to stop

The first run takes longer because `npx` downloads the MCP server packages.

## Configuration

**Model.** The model is set in `app.py`:

```python
llm = ChatGroq(model="openai/gpt-oss-120b", timeout=60, max_retries=2)
```

Use a Groq model that supports tool calling. To list the models your key can use:

```bash
uv run python -c "from dotenv import load_dotenv; load_dotenv(); from groq import Groq; print([m.id for m in Groq().models.list().data])"
```

**MCP servers.** To add or remove servers, edit `browser_mcp.json`. It uses the standard `mcpServers` format:

```json
{
  "mcpServers": {
    "playwright": { "command": "npx", "args": ["@playwright/mcp@latest"] }
  }
}
```

**Agent steps.** `max_steps=15` in `app.py` caps how many tool calls the agent can make for one request.

## Using the same servers in an editor chat

The repo also includes configs so the same MCP servers work in editor chat panels, with no Python or Groq key needed:

| File | Used by |
|---|---|
| `.vscode/mcp.json` | GitHub Copilot Chat (Agent mode) in VS Code |
| `.mcp.json` | Claude Code |

## Troubleshooting

| Problem | Fix |
|---|---|
| `model_not_found` (404) | Groq retired the model. Pick one from the model list command above. |
| Answers take a long time | The Groq free tier has a low tokens-per-minute limit, and every request sends all of the tool descriptions. Remove servers you don't need from `browser_mcp.json`. |
| `DDG detected an anomaly` | DuckDuckGo is rate-limiting you. Wait a bit, or remove the DuckDuckGo server. |
| `Recursion limit ... reached` | The agent ran out of steps, usually because a tool kept failing. Rephrase the request or raise `max_steps`. |

## Project structure

```
app.py             # interactive chat loop (entry point)
browser_mcp.json   # MCP servers used by app.py
.env.example       # template for your API key
.vscode/mcp.json   # MCP config for VS Code Copilot Chat
.mcp.json          # MCP config for Claude Code
pyproject.toml     # dependencies (managed with uv)
```
