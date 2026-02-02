Here’s a clean write-up of what you’re trying to do, based on your setup and the security boundary you want.

---

## Goal: Separate MCP servers by capability and trust boundary

You want a local “tool bridge” setup for Claude Code where **internet access and code generation are split into different MCP servers**, so you can audit and control them independently.

### Why

You don’t want “provider creep” where a project that is meant to be **Gemini-only web search** quietly grows an OpenAI path, API key, or codepath that could handle internet requests. You want:

* **Explicit control** over which vendor does what
* **Clear auditability** (easy to prove “OpenAI never touches web search”)
* **Minimized blast radius** (if one server is misconfigured, it can’t impact other capabilities)
* **Different policy enforcement** per capability (web ≠ code)

---

## Desired architecture

### 1) `gemini-web` MCP server — web search only

**Purpose:** Controlled, auditable web access via Gemini + Google Search grounding.

* Tool exposed: `web_search`
* Provider: Gemini only
* Inputs: user search queries
* Outputs: grounded summaries + source URLs, always treated as **untrusted**
* Guardrails:

  * “Explicit intent only” trigger (hooks)
  * Network-blocking hook (no curl/wget/etc)
  * Recency enforcement (no “latest” claims without sources)
  * Response sanitization + `UNTRUSTED` wrapping
  * Rate limiting + caching

**Hard requirement:** This server must *never* call OpenAI or contain a usable OpenAI provider.

---

### 2) `openai-code` MCP server — code generation only (later)

**Purpose:** A separate MCP server you can enable later for code generation/refactoring, **without any web search responsibility**.

* Tools exposed (examples):

  * `generate_code`
  * `refactor_code`
  * `explain_code`
  * `write_tests`
* Provider: OpenAI (or other coding model), but **no browsing / no external search**
* Inputs: tasks + optionally local code snippets passed by Claude Code
* Outputs: code patches / diffs / suggested edits

**Guardrails should be different from web search**, e.g.:

* Output should be **diff-only** or patch format (preferred)
* File/path allowlists (only certain directories)
* Explicit user intent phrases (“generate code for…”, “refactor…”, etc.)
* No network access (same Bash network hook applies)
* Token/context limits to prevent dumping whole repos

**Hard requirement:** This server should not implement web search and should not be able to access the internet.

---

## Directory layout you want

```
~/mcp/
  gemini-web/
    server.mjs
    start.sh
    providers/
      index.mjs
      gemini-provider.mjs
      base-provider.mjs
    lib/
      cache.mjs
      logger.mjs

  openai-code/
    server.mjs
    start.sh
    providers/
      index.mjs
      openai-code-provider.mjs
      base-provider.mjs
    policies/
      diff-only.mjs
      allowlist.mjs
```

Optional: a shared utilities folder if you want to reuse logging/schema helpers:

```
~/mcp/shared/
  logger.mjs
  schema.mjs
```

---

## How Claude Code uses it

You register both servers separately:

* `gemini-web` → used only when the user explicitly requests web access
* `openai-code` → used only when the user explicitly requests code generation/refactoring

That means Claude Code ends up with two clean tools:

* `web_search` (Gemini)
* `generate_code` / `refactor_code` (OpenAI)

And your hooks/policies enforce:

* Web access is routed only through `gemini-web`
* Code generation is routed only through `openai-code`
* Neither server can “accidentally” do the other’s job

---

## The core principle you’re enforcing

**One MCP server = one capability + one trust boundary.**

So:

* Web search is isolated because it ingests **untrusted internet**
* Code generation is isolated because it produces **executable output**
* Providers are chosen based on the capability, not mixed inside a single server

---

If you want, I can turn this into a `README.md`/`SETUP.md` section you can drop into your repo (same tone/style as the guide you pasted), including recommended environment variables and a “hardening checklist” for each server.

