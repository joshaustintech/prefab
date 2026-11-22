# MCP protocol state and Rust MCP options

Research for issue #3 (child of map #1). Checked 2026-11-21. Every claim cites the primary source that owns it.

## Short answer

- **Current spec:** MCP **`2026-07-28`**. This is a breaking revision: the protocol is now **stateless**. The `initialize` handshake and sessions are gone, and every request carries its protocol version and client capabilities in `_meta`. [versioning], [changelog]
- **Transport for a local CLI:** **stdio**, which is newline-delimited JSON-RPC 2.0 over the stdin and stdout of a subprocess. Streamable HTTP is for remote servers and is not needed for v1. [stdio], [transports]
- **Minimal compliant server:** answer `server/discover` (MUST), plus `tools/list` and `tools/call` if it declares `tools`. `resources/*` is optional. Every result needs `resultType`, and list results need `ttlMs` and `cacheScope`. [discover], [tools], [basic]
- **Compatibility catch:** a client that speaks only the old protocol (`2025-11-25` and earlier) **cannot talk to a server that speaks only the new one**. To work with both kinds of client, a server must also answer `initialize`. [versioning]
- **Rust options:** `rmcp` 3.11.0 is the official SDK. It needs tokio and pulls in about 70 crates, and it supports both protocol eras. `rust-mcp-schema` 2.0.0 provides only the message types: it needs serde and serde_json (about 12 crates) and no runtime. A stdlib-only build is about 1.0–1.5k lines of Rust, and roughly half of that is a JSON parser, because the stdlib has none.

## 1. Spec version and what changed

- The current version is `2026-07-28`. Version IDs follow the `YYYY-MM-DD` format and change only for backwards-incompatible changes. [versioning-root]
- Major changes since `2025-11-25` [changelog]:
  - The `initialize` / `notifications/initialized` handshake is removed. Every request sets `io.modelcontextprotocol/protocolVersion` and `io.modelcontextprotocol/clientCapabilities` in `_meta`.
  - `server/discover` is new, and servers MUST implement it.
  - Protocol sessions and `Mcp-Session-Id` are removed. List endpoints must not vary per connection.
  - `ping` and `logging/setLevel` are removed. The log level is now set per request through `_meta`.
  - Servers can no longer send requests to the client. Sampling, elicitation and roots now use Multi Round-Trip Requests: the server returns `resultType: "input_required"` and the client retries.
  - Every result carries a required `resultType` (`"complete"` or `"input_required"`).
  - `tools/list`, `resources/list` and `resources/read` results require `ttlMs` and `cacheScope`.
  - The resource-not-found code changes from `-32002` to `-32602`. MCP-specific codes now live in the range `-32020..-32099`.
  - Roots, Sampling and Logging are deprecated. Servers should log to stderr instead.
- Statelessness rules: servers MUST NOT rely on earlier requests on the same connection, and a stdio process is not a session. State that spans calls uses explicit handles passed as tool arguments. [basic], [tools § Stateful Tools]

## 2. Transports for a local CLI

- **stdio** [stdio]:
  - The client launches the server as a subprocess. Messages are UTF-8 JSON-RPC, one per line, with no embedded newlines.
  - The server MUST NOT write anything to stdout except valid MCP messages. It MAY log to stderr.
  - The server MUST NOT send JSON-RPC requests. It sends only responses and notifications.
  - Cancellation uses `notifications/cancelled`. The server SHOULD stop the work and MUST NOT send further messages for that request.
  - Shutdown: the client closes stdin, and the server SHOULD exit on EOF.
  - Authorization: stdio servers SHOULD NOT use the OAuth framework. They take credentials from the environment. [basic § Auth]
- **Streamable HTTP**: each message is a POST to one endpoint, and the reply is JSON or a per-request SSE stream. It requires `MCP-Protocol-Version`, `Mcp-Method` and `Mcp-Name` headers, and OAuth for remote use. Prefab's MCP front end is local, so v1 does not need it. [transports], [changelog minor 4]
- HTTP+SSE is deprecated. [changelog]

## 3. What a minimal compliant server implements (2026-07-28)

| Item | Requirement | Source |
|---|---|---|
| JSON-RPC 2.0 framing; request `id` is a string or integer, never null | MUST | [basic] |
| Reject requests missing `_meta` `protocolVersion` or `clientCapabilities` with `-32602` | MUST | [basic § _meta] |
| Reject an unsupported version with `-32022` and `data: {supported, requested}` | MUST | [versioning] |
| `server/discover` returns `supportedVersions`, `capabilities`, `_meta.serverInfo` (SHOULD), `instructions` (optional) and cache hints | MUST | [discover] |
| `resultType: "complete"` on every result | MUST | [basic] |
| `tools/list` returns `{resultType, tools[], nextCursor?, ttlMs, cacheScope}` in a deterministic order (SHOULD) | MUST if `tools` is declared | [tools], [changelog minor 5] |
| `tools/call` returns `{resultType, content[], structuredContent?, isError?}` | MUST if `tools` is declared | [tools] |
| `resources/list`, `resources/read` and `resources/templates/list`; not-found returns `-32602` | Only if `resources` is declared | [resources] |
| `_meta.io.modelcontextprotocol/serverInfo` on results | SHOULD | [basic] |

**Error shapes** [basic § Error Codes], [tools § Error Handling], [resources § Error Handling]:

- These are standard JSON-RPC codes:
  - `-32700`: parse error.
  - `-32600`: invalid request.
  - `-32601`: method not found.
  - `-32602`: invalid params. This also covers an unknown tool, missing `_meta` and a missing resource.
  - `-32603`: internal error.
- MCP adds three codes:
  - `-32020`: HeaderMismatch (HTTP only).
  - `-32021`: MissingRequiredClientCapability.
  - `-32022`: UnsupportedProtocolVersion.
- **Tool execution failures are not JSON-RPC errors.** They return a normal result with `isError: true` and text the model can act on. Protocol errors cover only an unknown tool, a malformed request or a server fault.
- New implementations SHOULD NOT use `-32000..-32019`. Application-defined codes go outside `-32768..-32000`.

**Tool input schemas** default to JSON Schema 2020-12. A tool with no arguments should use `{"type":"object","additionalProperties":false}`. Servers MUST validate tool inputs, but the spec does not require a general-purpose JSON Schema validator; validation inside each tool is enough. [basic § JSON Schema Usage], [tools]

### Legacy (pre-2026-07-28) clients

- A modern-only server rejects `initialize`. A legacy client cannot move forward to the new protocol, so **that pairing fails**. [versioning § Compatibility Matrix]
- A server that supports both eras MAY answer `initialize` and serve that stdio process under legacy semantics. A modern-only server SHOULD list its supported versions in the error it returns to `initialize`. [versioning]
- Major SDKs still maintain legacy lines, which suggests legacy clients are still common:
  - The TypeScript SDK ships 2.x alongside a maintained 1.32.x line and a `server-legacy` package. [ts-sdk releases]
  - rust-mcp-sdk keeps 1.x as LTS for `2025-11-25`. [rust-mcp-sdk README]
- Legacy support for tools only (`2025-11-25`) adds:
  - `initialize`, which returns `protocolVersion`, `capabilities` and `serverInfo`.
  - Accepting `notifications/initialized`.
  - `ping`.
  - Results without `resultType` or cache fields.

## 4. Rust options

Footprints were measured with `cargo tree -e normal` on cargo 1.99.1 against the crates.io index on 2026-11-21. The counts are unique crates, excluding the root. For reference, Tera 2.4.0 (already approved) adds 3 crates: `tera`, `serde` and `serde_core`. It does **not** pull in serde_json or tokio.

### Candidate A: `rmcp` 3.11.0 (official SDK)

- Repository: `modelcontextprotocol/rust-sdk`. 3.11.0 was released 2026-11-17, MSRV is 1.88, and recent downloads are about 31M. Releases come out weekly. [crates.io rmcp], [rust-sdk releases]
- Implements the stable `2026-07-28` spec and stays compatible with `2025-11-25` and earlier. A `serve()`-based server answers a legacy `initialize`, so it **supports both eras**. [rmcp README]
- **Requires the tokio async runtime** ("with tokio async runtime"). The stdio transport is `(tokio::io::stdin(), tokio::io::stdout())` behind the `transport-io` feature. [rmcp README], [rmcp Cargo.toml]
- Footprint with `default-features = false` and features `server,macros,transport-io` plus tokio `rt,macros,io-std`: **70 crates**. They include tokio, futures, tokio-util, tracing, schemars plus derive, serde_json, chrono, uuid, sha2, indexmap, syn/quote/darling and unicode-normalization. With default features it is 71. [measured]
- Macros (`#[tool]`, `#[tool_router]`) derive `inputSchema` from a `schemars::JsonSchema` struct. [rmcp README]
- Trade-offs:
  - Pro: official, tracks the spec quickly, handles both eras, and Streamable HTTP is available later behind a feature.
  - Con: the largest footprint. It brings an async runtime into a CLI that is otherwise synchronous.
  - Con: the 3.x API churns; there is a migration guide and minor releases come out weekly.
  - Con: the macro style leans on traits, async and `Send`. That is harder to square with the owned-values, minimal-lifetimes, minimal-`Arc` style, although handler types can still be plain `Clone` structs.

### Candidate B: `rust-mcp-schema` 2.0.0 (types only) with a hand-written stdio loop

- Repository: `rust-mcp-stack/rust-mcp-schema`. 2.0.0 was released 2026-08-22, with about 335k recent downloads. It provides type-safe structs generated from the official schema. Each protocol version is a Cargo feature, and `2026-07-28` (the default) and `2025-11-25` can be enabled together. [rust-mcp-schema README]
- Dependencies: only `serde` and `serde_json`. **No runtime and no async.** [rust-mcp-schema Cargo.toml]
- Footprint: **12 crates**. Together with Tera the total is 13, because serde is shared. [measured]
- Prefab still writes the blocking stdin loop, dispatch, `_meta` checks and the legacy `initialize` path itself, roughly 400–600 lines (estimate, see section 5 minus the JSON layer).
- Trade-offs:
  - Pro: small and synchronous, with correct wire types and serde_json for free.
  - Con: a third-party crate (about 79 GitHub stars) rather than the official one, so updating after spec changes is up to us.
  - Con: spec compliance is our responsibility, though the official conformance suite can check it.
- A related option is the full `rust-mcp-sdk` 2.0.0 from the same maintainers. It is modern-only, with 1.x LTS for legacy, needs tokio, and pulls in 68 crates with `server,macros,stdio`. It is heavier than rmcp without rmcp's dual-era support, so it is not recommended. [rust-mcp-sdk README], [measured]

### Option C: stdlib only

The work, estimated from the requirements in section 3:

| Piece | Approx. lines | Notes |
|---|---|---|
| JSON value type, parser and serializer (RFC 8259: strings with escapes and surrogates, numbers, nesting) | 400–700 | The stdlib has no JSON. This is the bulk of the work and of the risk. |
| stdio loop (`std::io::stdin().lock().lines()`, write a line and flush stdout, exit on EOF) | 50–100 | Synchronous and single-threaded. Requests are processed in order, and `notifications/cancelled` can be ignored for short tools, since stopping is only a SHOULD. |
| JSON-RPC envelope and error helpers | 100–150 | Covers `-32700`, `-32600`, `-32601`, `-32602`, `-32603` and `-32022`. |
| `server/discover`, `tools/list`, `tools/call`, `_meta` validation | 200–300 | Tool schemas are written as JSON literals. |
| Legacy `initialize` / `ping` path (optional) | 80–150 | Needed so pre-2026 clients can connect. |
| `resources/*` (optional) | 100–200 | |
| **Total** | **~1.0–1.5k plus tests** | Can be checked with `modelcontextprotocol/conformance`. [conformance] |

- Trade-offs:
  - Pro: zero new crates and fully synchronous. It fits the owned-values style naturally, and nothing upstream changes under us.
  - Con: we own a JSON parser, which needs fuzz and edge-case tests, and we own spec tracking.
  - Con: tool input schemas are hand-written rather than derived.

### Summary

| | rmcp 3.11.0 | rust-mcp-schema 2.0.0 + own loop | stdlib only |
|---|---|---|---|
| New crates (excl. Tera) | ~70 | ~12 (serde_json) | 0 |
| Async runtime | tokio required | none | none |
| Handles both protocol eras | yes, built in | types for both; we write the dispatch | we write it |
| Prefab code to write | smallest | ~400–600 lines | ~1.0–1.5k lines |
| Spec tracking | upstream (official) | upstream types, our behavior | ours |
| Fit with the Rust style | weakest (async, traits) | good | best |

Note on the hybrid: if serde_json is approved for other reasons (for example, Universal Description Spec parsing in another ticket), Option C's biggest cost goes away. Option C would then be very close to Option B without the schema crate.

## Open questions surfaced

1. **Which protocol eras must v1 support?** This depends on which MCP clients the maintainer targets and whether those clients speak `2026-07-28` yet. A modern-only server will fail with legacy clients.
2. **Is serde_json, or some JSON crate, going to be approved anyway** for the Universal Description Spec (JSON or YAML parsing)? That changes the cost of Options B and C.
3. Is Streamable HTTP ever in scope for v1? If it is, rmcp's HTTP server features matter. If not, stdio alone is enough.
4. Will any MCP tool run long (generating a full Service)? If so, either progress notifications, cancellation (which needs a reader thread in Options B and C), or the Tasks extension come into play.

## Sources

- [versioning-root]: https://modelcontextprotocol.io/specification/versioning
- [versioning]: https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning
- [changelog]: https://modelcontextprotocol.io/specification/2026-07-28/changelog
- [basic]: https://modelcontextprotocol.io/specification/2026-07-28/basic
- [transports]: https://modelcontextprotocol.io/specification/2026-07-28/basic/transports
- [stdio]: https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/stdio
- [discover]: https://modelcontextprotocol.io/specification/2026-07-28/server/discover
- [tools]: https://modelcontextprotocol.io/specification/2026-07-28/server/tools
- [resources]: https://modelcontextprotocol.io/specification/2026-07-28/server/resources
- [rmcp README], [rust-sdk releases], [rmcp Cargo.toml]: https://github.com/modelcontextprotocol/rust-sdk (README.md, releases, crates/rmcp/Cargo.toml)
- [crates.io rmcp]: https://crates.io/crates/rmcp
- [rust-mcp-schema README], [rust-mcp-schema Cargo.toml]: https://github.com/rust-mcp-stack/rust-mcp-schema
- [rust-mcp-sdk README]: https://github.com/rust-mcp-stack/rust-mcp-sdk
- [ts-sdk releases]: https://github.com/modelcontextprotocol/typescript-sdk/releases
- [conformance]: https://github.com/modelcontextprotocol/conformance
- [measured]: `cargo tree -e normal` in scratch crates: rmcp 3.11.0, rust-mcp-schema 2.0.0, rust-mcp-sdk 2.0.0, tera 2.4.0; cargo 1.99.1; 2026-11-21.
