---
name: agent-cli-design
description: Design principles and patterns for building CLIs that work well for both AI agents and humans. Covers machine-readable output, schema introspection, input hardening, context window discipline, safety rails, and multi-surface architecture. Use this skill whenever the user is building, designing, or improving a CLI tool — especially if it will be used by AI agents, LLMs, or automation. Also trigger when discussing CLI output formats, agent-friendly APIs, MCP server design, or making existing tools machine-readable.
---

# Agent-First CLI Design

Principles for building CLIs that AI agents can use reliably. Adapted from [Justin Poehnelt's "Rewrite your CLI for AI Agents"](https://justin.poehnelt.com/posts/rewrite-your-cli-for-ai-agents/) and [Cursor's "CLI for Agents" skill](https://x.com/ericzakariasson/status/2036762680401223946), based on patterns from the Google Workspace CLI and the Cursor CLI.

**Core tension:** Human DX optimizes for discoverability and forgiveness. Agent DX optimizes for predictability and defense-in-depth. Great CLIs serve both.

---

## 1. Machine-Readable Output is Table Stakes

Every command must support structured output. Agents cannot reliably parse tables, colored text, or free-form prose.

**Requirements:**
- Support `--output json` (or make JSON the default)
- Use an environment variable fallback: `OUTPUT_FORMAT=json`
- Auto-detect: default to JSON when stdout is not a TTY
- Wrap responses in a stable envelope with consistent fields
- Use deterministic key ordering for reproducible output

**Envelope pattern:**
```json
{
  "status": "ok",
  "data": { ... },
  "metadata": { "cached": false, "request_id": "abc123" }
}
```

**Error envelope (same structure, always):**
```json
{
  "status": "error",
  "error": { "code": "INVALID_INPUT", "message": "..." },
  "metadata": {}
}
```

Agents must be able to distinguish success from failure programmatically. Never change the envelope shape based on flags like `--results-only` for errors.

---

## 2. Accept Structured Input for Complex Operations

Human-friendly flags can't express nested structures without layers of custom abstractions. Support both paths:

**Human path (convenience flags):**
```
mycli users create --name "Jane" --role admin --notify
```

**Agent path (raw JSON):**
```
mycli users create --json '{"name": "Jane", "role": "admin", "notify": true, "permissions": ["read", "write"]}'
```

When a command wraps a structured API, let agents pass the full payload as JSON. This eliminates translation loss and lets agents generate requests directly from API schemas.

---

## 3. Schema Introspection Replaces Documentation

Static docs in system prompts are expensive (tokens) and go stale. Make the CLI self-describing at runtime:

```
cli schema <command>
cli <command> --describe
```

Should return machine-readable JSON with:
- Parameter names, types, required/optional
- Valid enum values and defaults
- Request/response structure
- Auth requirements (scopes, API keys)

This lets agents discover capabilities without pre-loaded documentation.

**Practical alternative for simpler CLIs:** A `schema` command that dumps all commands with their flags, types, and descriptions as a single JSON document.

---

## 4. Examples Over Documentation in Help Text

Agents pattern-match from examples more effectively than they parse prose descriptions. A single working invocation teaches an agent the flag syntax, argument ordering, and expected values faster than a paragraph of explanation — it's few-shot learning from your `--help` output.

Every `--help` should include real, copy-pasteable invocations.

**Bad** — flags and types only, agent must infer usage:
```
Options:
  --env string    Target environment
  --tag string    Image tag
  --force         Skip confirmation
```

**Good** — includes working examples the agent can adapt directly:
```
Options:
  --env string    Target environment
  --tag string    Image tag (default: latest)
  --force         Skip confirmation

Examples:
  mycli deploy --env staging
  mycli deploy --env production --tag v1.2.3
  mycli deploy --env staging --force
```

**What makes a good example set:**
- Start with the simplest common case (minimal required flags)
- Show a realistic full invocation (multiple flags, real-looking values)
- Include edge cases agents are likely to hit (`--dry-run`, `--json` input, piped stdin)
- Use concrete values, not placeholders — `--env staging` teaches more than `--env <ENV>`

This complements schema introspection (§3): schemas give agents the structure, examples give them the pattern to follow. Both matter — schema for validation, examples for generation.

---

## 5. Context Window Discipline

API responses consume agent token budget. Humans scroll; agents pay per token.

**Field selection** - Let agents request only the fields they need:
```
mycli users list --select "id,name,role"
mycli users list --fields "id,name,role"
```

**Pagination controls** - Bounded, agent-friendly:
```
cli list --limit 10 --offset 0
```

**NDJSON streaming** - For large result sets, emit one JSON object per line instead of buffering a top-level array. Agents can process incrementally:
```
cli list --page-all --output ndjson
```

**Guidance for agent skill files:** "ALWAYS use field selection when listing resources to minimize token usage."

---

## 6. Input Hardening Against Hallucinations

Agents hallucinate in predictable patterns. Validate defensively:

| Hallucination pattern | Defense |
|---|---|
| Path traversal (`../../.ssh`) | Canonicalize and sandbox to working directory |
| Embedded query params in IDs (`file123?fields=name`) | Reject `?` and `#` in resource identifiers |
| Double-encoded strings (`%2F` passed to URL encoder) | Reject `%` in raw inputs; encode at HTTP layer |
| Control characters (invisible bytes) | Reject anything below ASCII 0x20 |
| Invented flag names | Strict flag parsing; reject unknown flags with clear errors |
| Wrong types (string where int expected) | Type-check all inputs before processing |

**Core principle:** The agent is not a trusted operator. Validate agent input the same way you validate untrusted user input in a web API.

---

## 7. Safety Rails for Mutations

Mutations (create, update, delete) are where hallucinated parameters cause real damage.

**`--dry-run`** - Validate the request locally without executing:
```
mycli users delete --id 42 --dry-run
# Returns: what would happen, validated, no side effects
```

**Confirmation prompts** - For destructive operations, require explicit confirmation or a `--yes`/`--confirm` flag.

**Bounded defaults** - Prefer safe defaults that agents must explicitly override:
- Bounded approvals over unlimited approvals
- Read-only operations by default
- Require `--force` for destructive overrides

**Response sanitization** - API responses may contain prompt injection attempts (e.g., a malicious email body saying "Ignore previous instructions"). Sanitize or flag untrusted content in responses before returning to agents. Consider a `--sanitize <TEMPLATE>` flag that pipes responses through an injection defense layer before the agent sees them.

---

## 8. Stable Exit Codes

Agents parse exit codes to decide next actions. Define and document them:

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | General error |
| 2 | Usage/input error |
| 3 | Auth error |
| 4 | Not found |
| 5 | Rate limited / temporary |

Never change exit code semantics across versions. Map internal error types to exit codes deterministically.

---

## 9. Headless Authentication

Agents cannot do browser-based OAuth flows. Support headless auth paths:

- **Environment variables** for tokens: `CLI_TOKEN`, `CLI_API_KEY`
- **Credential files**: `CLI_CREDENTIALS_FILE`
- **Service accounts** where the platform supports them
- **Key files** with auto-discovery: `~/.config/cli/key.hex`

Document which commands need which auth. Make metadata-only commands (version, schema, provider list) work without any credentials.

---

## 10. Ship Agent Context, Not Just Commands

Agents learn from injected context, not trial and error. Ship structured guidance:

**Skill files / CONTEXT.md** encode invariants agents can't intuit:
- "Always use `--dry-run` before mutating operations"
- "Always add `--select` to list calls to limit output size"
- "Dates must be ISO 8601 format"
- "IDs are UUIDs, not sequential integers"

**Non-obvious behaviors** that break agent assumptions:
- Which commands require API keys
- Which flags change envelope structure
- Cache behavior and TTL semantics
- Provider-specific quirks and limitations

A skill file is cheaper than a hallucination.

---

## 11. Multi-Surface Architecture

Expose the same capabilities through multiple interfaces from a single source of truth:

```
                  Source of Truth
                  (core library)
                        |
         +------+------+------+--------+
         |      |      |      |        |
        CLI    MCP    SDK   Env Vars  Native Ext
      (human) (agent) (dev) (headless) (plugins)
```

**MCP (Model Context Protocol)** - Expose commands as typed JSON-RPC tools over stdio. Eliminates shell escaping, argument parsing ambiguity, and output parsing. High-value for API-backed CLIs.

**CLI** - Human-friendly surface with help text, colors, tables.

**Environment variables** - Headless configuration and credential injection.

**Native extensions** - Platform-specific plugin systems (e.g., Gemini CLI Extensions, VS Code extensions) that wrap the core library for deeper integration.

All surfaces should produce identical results for identical inputs.

---

## Implementation Priority

Retrofit existing CLIs incrementally in this order:

1. **`--output json`** - Machine-readable output with stable envelope
2. **Input validation** - Reject control chars, path traversals, unknown flags
3. **Stable exit codes** - Map errors to documented codes
4. **Examples in `--help`** - Copy-pasteable invocations agents can pattern-match from
5. **Schema/describe command** - Runtime introspection
6. **Field selection (`--select`/`--fields`)** - Protect context window
7. **`--dry-run`** - Validate before mutating
8. **Agent skill files** - Encode invariants and non-obvious behaviors
9. **MCP surface** - Typed tool invocation (if wrapping structured APIs)

---

## Anti-Patterns to Avoid

- **Interactive prompts without `--yes` bypass** - Agents can't answer "Are you sure? [y/N]"
- **Color codes / spinners in stdout** - Breaks JSON parsing; use stderr for human decoration
- **Changing output shape between versions** - Treat JSON output as a contract
- **Implicit defaults that vary by context** - Be explicit about what happens when flags are omitted
- **Mixing human text with structured data** - Separate data (stdout) from messages (stderr)
- **Unbounded output** - Always support `--limit`; never dump thousands of records by default
- **Swallowing errors into generic messages** - Return typed, specific errors with context

---

## Checklist for Agent-Ready CLIs

- [ ] All commands support `--output json` with stable envelope
- [ ] Error responses use the same envelope as success responses
- [ ] Exit codes are documented and deterministic
- [ ] Unknown flags are rejected (no silent ignoring)
- [ ] Inputs are validated against hallucination patterns
- [ ] Mutations support `--dry-run`
- [ ] Field selection available on list/read commands
- [ ] Auth works via environment variables (no browser required)
- [ ] `--help` includes copy-pasteable examples for every subcommand
- [ ] Schema or describe command exists for runtime introspection
- [ ] Agent skill files document invariants and non-obvious behaviors
- [ ] Output pagination is bounded (`--limit`)
- [ ] Human decoration (color, spinners) goes to stderr, not stdout
