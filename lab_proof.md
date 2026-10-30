# Lab proof: Agents that speak MCP fluently

Entry point for review. Code: [`mcp_langchain.ipynb`](./mcp_langchain.ipynb). Source material: [`sample_docs/mcp_overview.md`](./sample_docs/mcp_overview.md).

## 1. Workflow

```text
user query
  -> LangChain agent (create_agent + ChatOpenAI gpt-4o-mini, temperature 0)
       -> MCP tool calls   (filesystem server, 14 tools, scoped to sample_docs/)
       -> MCP resource     (everything server, features.md, loaded into the system prompt)
  -> final answer
```

| Piece | What I used |
|---|---|
| Client | `langchain-mcp-adapters` 0.3.2, `MultiServerMCPClient`, stdio transport |
| Server 1 (tools) | `@modelcontextprotocol/server-filesystem`, allowed directory: `sample_docs/` only |
| Server 2 (resources) | `@modelcontextprotocol/server-everything`, 7 static resources + 2 dynamic templates |
| Agent | `langchain.agents.create_agent(llm, tools, system_prompt=...)`, 30 tools from both servers |

Steps covered: 1 (setup), 2 (connect), 3 (tools -> LangChain), 4 (agent), 5 (resources), 6 (multi-server agent).

Findings along the way:
- The filesystem server exposes **no resources**: `list_resources()` raises `McpError: Method not found`. Step 5 therefore uses a second server.
- In this version the tool loader is `client.get_tools()` and the resource loader is `client.get_resources()`; the lab text's `get_langchain_tools()` does not exist.
- `MultiServerMCPClient` has no `close()`; sessions are opened per tool call and closed automatically.

## 2. Query

```text
Read mcp_overview.md and tell me which transports it mentions for local and remote servers.
Then tell me which transports the Everything server serves according to its features doc.
```

This needs both MCP paths: a file read through a **tool**, and the features doc supplied as an MCP **resource**.

## 3. Retrieved and tool evidence

**Tool call** (filesystem server):

```text
[TOOL CALL] read_text_file({'path': '.../sample_docs/mcp_overview.md'})
```

Returned text (relevant line):

> An MCP client talks to a server over a transport. Local servers usually use stdio; remote servers use HTTP.

**Resource** (everything server), `demo://resource/static/document/features.md`, 16,839 characters, supplied in the system prompt. Relevant lines:

> The server serves two protocol eras from one process, over stdio and over Streamable HTTP.

> The HTTP+SSE transport serves this era only. *(the 2025-era "legacy" protocol)*

## 4. Final output

> The `mcp_overview.md` file mentions the following transports for local and remote servers:
> - **Local servers** usually use **stdio**.
> - **Remote servers** use **HTTP**.
>
> According to the features document of the Everything server, it serves two protocol eras over the following transports:
> - **2026-07-28** (`modern`): over **Streamable HTTP**.
> - **2025-era** (`legacy`): over **HTTP+SSE**.
>
> Thus, the Everything server supports both Streamable HTTP and HTTP+SSE for its respective protocol eras.

## 5. Verification: tying the answer to the sources

| Claim in the answer | Supported by | Verdict |
|---|---|---|
| Local servers use stdio | `mcp_overview.md` via `read_text_file` | Supported |
| Remote servers use HTTP | `mcp_overview.md` via `read_text_file` | Supported |
| Everything server serves Streamable HTTP | `features.md` resource | Supported |
| Legacy era uses HTTP+SSE | `features.md` resource | Supported |
| Everything server serves stdio | `features.md` resource | **Missing from the answer** |

## 6. Failure and limitation

**Plausible but incomplete answer.** The final answer reads fluently and every claim in it is supported, but it **omits stdio**. The features doc says the server runs "over stdio and over Streamable HTTP", and the model's summary dropped stdio while adding the HTTP+SSE detail. Nothing in the answer signals that something was left out, so only a check against the source catches it.

**Resource mistaken for a file.** In the first run of this same query the agent treated "the features doc" as a file on disk. It called `read_text_file` on `sample_docs/features.md` (`ENOENT`), then `search_files` ("No matches found"), and gave up on the second half of the question. The system prompt had not said the resource is already loaded and is not a file. Adding that line fixed it, which shows the answer depends on prompt wording and not only on the tools.

**Other grounding risks:**
- The whole 17k-character resource goes into the prompt. That works for one document but does not scale, and the model can quote it with no check on whether it is current.
- The agent guesses file names without listing them first (its first search was for `*.txt`), which costs calls and can miss files.
- The filesystem server includes `write_file`, `edit_file` and `move_file`. The directory scope limits the damage, but the agent is only told by its prompt not to write.
