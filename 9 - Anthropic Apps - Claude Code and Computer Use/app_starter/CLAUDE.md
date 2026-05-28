# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Dependency management and execution go through `uv`:

```bash
uv venv                       # create virtualenv (.venv)
source .venv/bin/activate     # POSIX; on Windows PowerShell: .venv\Scripts\Activate.ps1
uv pip install -e .           # editable install of the `app` package

uv run main.py                # start the MCP server (stdio transport via FastMCP)
uv run pytest                 # run the full test suite
uv run pytest tests/test_document.py::TestBinaryDocumentToMarkdown::test_binary_document_to_markdown_with_pdf  # single test
```

Python `>=3.10` is required (see `pyproject.toml`).

## Architecture

This repo is a **FastMCP server** that exposes Python functions as MCP tools to AI assistants. The big-picture shape is intentionally small:

- `main.py` constructs a single `FastMCP("docs")` instance, registers tools onto it via `mcp.tool()(fn)`, and calls `mcp.run()`. This is the only entry point — there is no plugin discovery or auto-registration. **Every new tool must be explicitly registered in `main.py`**, otherwise it will not be exposed to MCP clients.
- `tools/` holds plain Python functions, one logical tool per module (`tools/math.py`, `tools/document.py`). Modules here are decoupled from MCP — they don't import `mcp` and don't use `@mcp.tool()`. Registration happens in `main.py` so the same functions stay unit-testable in isolation.
- `tests/` uses `pytest` and imports the tool functions directly (e.g. `from tools.document import binary_document_to_markdown`). Binary fixtures live in `tests/fixtures/` (`mcp_docs.docx`, `mcp_docs.pdf`) and are loaded as raw bytes — mirror this pattern when adding tools that take binary input.
- Document conversion is delegated to `markitdown` (`markitdown[docx,pdf]`), driven by a `StreamInfo(extension=...)` hint so the same function handles multiple formats from an in-memory `BytesIO`.

## MCP tool authoring conventions

The two existing tools (`tools/math.py::add`, `tools/document.py::binary_document_to_markdown`) are the templates. Follow them precisely — MCP clients use the docstring and `Field` metadata verbatim when deciding whether and how to invoke a tool.

**Function signature.** Use `pydantic.Field` as the default value for every parameter, with a `description=` that an LLM can read cold:

```python
from pydantic import Field

def my_tool(
    param1: str = Field(description="Detailed description of this parameter"),
    param2: int = Field(description="Explain what this parameter does"),
) -> ReturnType:
    ...
```

Type hints are required on every parameter and the return — FastMCP derives the JSON schema from them.

**Docstring.** This is the tool's prompt-visible contract. Structure it as:

1. One-line summary (first line of the docstring).
2. A longer paragraph explaining what the tool does and what kinds of input it accepts.
3. A `When to use:` bulleted list of triggers/intents.
4. (Optional but encouraged) A `When not to use:` list to keep the model from over-firing.
5. `Examples:` with `>>>`-style input/output so the LLM can pattern-match.

`tools/math.py::add` shows the minimal version of this shape.

**Registration.** After writing the function, register it in `main.py`:

```python
from tools.mymodule import my_tool
mcp.tool()(my_tool)
```

Keep one `mcp.tool()(...)` line per tool — do not batch-register via loops; explicit registration is what makes the surface area visible at a glance.

**Binary I/O.** Tools that consume binary data should accept `bytes` plus a discriminator (see `binary_document_to_markdown(binary_data, file_type)`) rather than file paths. The MCP boundary passes bytes, not filesystem paths, so wrap with `BytesIO` internally if a library needs a file-like object.

**Testing.** Import the underlying function (not the MCP-wrapped version) and test it as a normal Python callable. There is no need to spin up the MCP server in tests.
