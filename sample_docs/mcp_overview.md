# MCP overview

The Model Context Protocol (MCP) is an open standard that lets AI applications connect to external tools and data.

- **Tools** are functions a model can call, such as reading a file.
- **Resources** are read-only data a server exposes for context.
- **Prompts** are reusable templates a server offers to clients.

An MCP client talks to a server over a transport. Local servers usually use stdio; remote servers use HTTP.
