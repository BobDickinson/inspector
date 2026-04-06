# mcpc integration design

This document tracks the design for integrating **[mcp-cli](https://github.com/apify/mcp-cli)** (npm **`@apify/mcpc`**, binaries **`mcpc`**, **`mcpc-bridge`**) into the MCP Inspector monorepo.

**Product goal:** **Apify donates the source code** to the project; we **adapt** it as described here (built on **`@modelcontextprotocol/inspector-core`**, consistent with the TUI and web clients) and **replace the existing Inspector CLI** (`clients/cli`) with the resulting implementation. There should be **one** supported first-party CLI surface area going forward, not parallel “minimal CLI” and “mcpc” products.

**Working branch:** `mcpc` (based on **`v1.5/main`**, kept in sync with **`upstream/v1.5/main`** on the maintainer fork).

For the as-built shared client architecture, see [shared-code-architecture.md](shared-code-architecture.md), [environment-isolation.md](environment-isolation.md), and [protocol-and-state-managers-architecture.md](protocol-and-state-managers-architecture.md).

---

## 1. Source project (what we are porting)

| Aspect           | Detail                                                                                                      |
| ---------------- | ----------------------------------------------------------------------------------------------------------- |
| **Repository**   | `github.com/apify/mcp-cli`                                                                                  |
| **Package**      | `@apify/mcpc` (binaries `mcpc`, `mcpc-bridge`)                                                              |
| **License**      | Apache-2.0 today (donation may require **license / attribution** alignment with the Inspector repo; see §7) |
| **Contribution** | **Donation** of codebase by Apify for adaptation under MCP Inspector governance (see §7, §9)                |

**Major subsystems today:**

- **Core:** `McpClient` wraps the MCP SDK `Client`; `createMcpClient` + `transports.ts` (stdio, streamable HTTP, proxy-aware fetch, OAuth provider injection, custom fetch for x402, MCP session id, timeouts).
- **Bridge:** Long-lived process, Unix-socket IPC, keepalives, OAuth + x402 wiring, task tracking, optional **proxy MCP server** (SDK **server** side).
- **CLI:** Commander commands (tools, resources, prompts, tasks, auth, sessions, grep, x402, shell, etc.).
- **Tests:** Jest unit tests + shell e2e; e2e uses SDK **server** (`Server`, `StreamableHTTPServerTransport`).

**SDK touchpoints today:** client (stdio, streamable HTTP, auth), types, `shared/transport` (`FetchLike`), and **server** (bridge proxy + e2e).

---

## 2. Goals

1. **Replace the Inspector CLI** — Retire **`clients/cli`** as the standalone “`--method` + JSON” implementation; ship **one** CLI (adapted mcpc) as the supported command-line experience, including integration with the **launcher** (`inspector --cli` or successor flags) as documented after migration.
2. **Use core as the primary MCP client layer** — `InspectorClient` and Node transport wiring (`createTransportNode` / custom `CreateTransport`), not a parallel hand-rolled SDK wrapper inside the donated code.
3. **Stay as thin as practical** — Sessions, bridge IPC, x402, and CLI UX live in the CLI workspace; protocol patterns align with TUI and web.
4. **Preserve donated product behavior** — Same capabilities as upstream mcpc unless we explicitly defer scope (phased migration); add **compatibility** for today’s Inspector CLI **single-shot `--method`** workflows where required (see §3.1).
5. **Do not pretend to remove the SDK** — core is built on `@modelcontextprotocol/sdk`; the CLI will still import SDK types, notification schemas, and **server** APIs where needed (see §5).
6. **One OAuth implementation and one credential store (product requirement)** — All first-party clients must use the **same OAuth code path** in core (**`OAuthManager`** + **`environment.oauth`** / **`options.oauth`**, not a forked token stack only inside the ported CLI) and the **same persisted credentials** where technically possible: **Node** (CLI, TUI, bridge, dev server) shares **one** default **`OAuthStorage`** in **`core/auth/node/`** (today file; target keychain-first, §5.1). **Browser** must use **`RemoteOAuthStorage`** via the Inspector API’s **`/api/storage/oauth`** so tokens match that Node store on the API host (**not** a separate sessionStorage-only world—see [environment-isolation.md](environment-isolation.md); the web factory switch is **pending** in code). A user who authenticates in one client should not re-authenticate in another when connecting to the same server profile.

---

## 3. Target shape in this repo

### 3.1 Workspace layout and CLI replacement

- **Replace `clients/cli`** with the adapted codebase (either by **renaming** the donated tree into `clients/cli` or by using **`clients/mcpc`** as the canonical workspace and **removing** the old `clients/cli` package while moving the **`@modelcontextprotocol/inspector-cli`** name and launcher wiring to the new implementation). Exact layout is an implementation choice; the outcome is **one** CLI package consumed by the root/launcher.
- Root **`package.json`**: workspace entry, `bin` / launcher targets, build and test scripts updated so CI and `npx` flows invoke the new CLI.
- **Backward compatibility:** Today’s **`inspector --cli … --method tools/list`** (and related flags) are relied on in docs and automation. The replacement must either:
  - **Re-implement** that surface as a thin mode on top of shared core (same flags, same JSON shape), or
  - **Document a breaking change** with a migration guide and, if needed, a **transitional shim** or major version bump.

The comparison in §3.3 is written against the **current** Inspector CLI so we preserve or explicitly migrate each use case.

### 3.2 Dependencies

- **`@modelcontextprotocol/inspector-core`** (workspace): `InspectorClient`, `createTransportNode`, types, logging helpers as appropriate.
- **`@modelcontextprotocol/sdk`**: peer/direct dependency for types, Zod schemas (e.g. logging notifications), **`Server` / `StreamableHTTPServerTransport`** for bridge proxy and e2e test servers.

Align **Node `engines`** with the monorepo: root **`package.json`** requires **`node >= 22.7.5`**; donated mcpc upstream allowed **Node 20+**, so the merged CLI must bump accordingly.

### 3.3 Legacy Inspector CLI behavior: coverage and migration mapping

The **current** **Inspector CLI** (`clients/cli`), to be **replaced**, is a **single-shot** tool: each run builds an `InspectorClient`, connects, performs **one** MCP method (`--method`), prints **JSON**, and exits. It is normally launched through the **launcher** as `inspector --cli …` (see `clients/cli/README.md` and [mcp-server-configuration.md](mcp-server-configuration.md) for server selection flags).

**Can mcpc duplicate that functionality?** **Yes, for the MCP operations themselves.** Listing and calling tools, listing resources (and templates), reading resources, listing and getting prompts, and setting server log level are all within mcpc’s command set. mcpc can emit **JSON** for scripting (`--json`), similar to the Inspector CLI’s always-JSON output.

**Where they differ (by design):**

| Aspect               | Inspector CLI                                                                                                                                                                                                                                                                                                        | mcpc (upstream)                                                                                              |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| **Process model**    | One Node process; connect → one RPC → disconnect                                                                                                                                                                                                                                                                     | Typical path uses a **named session** (`@name`) talking to a **bridge** process with a long-lived connection |
| **Entrypoint**       | `npx @modelcontextprotocol/inspector --cli …`                                                                                                                                                                                                                                                                        | Standalone `mcpc` binary                                                                                     |
| **Remote transport** | **`resolveServerConfigs`** (`core/mcp/node/config.ts`): if `--transport` is set, use **`sse`** or **`http`** (mapped to streamable HTTP). If omitted, URL path must end with **`/sse`** → SSE or **`/mcp`** → streamable HTTP; **otherwise resolution throws** (bare host URLs need `--transport` or a path suffix). | mcpc uses **streamable HTTP** for HTTPS URL targets in its transport factory (stdio for local `command`).    |
| **Scope**            | Narrow, automation-focused surface                                                                                                                                                                                                                                                                                   | Sessions, OAuth/keychain, proxy MCP server, tasks, interactive shell, x402, grep, schema validation, etc.    |

So mcpc is a **strict superset** for MCP-facing features: anything the Inspector CLI does as an MCP client is representable in mcpc, but the **CLI ergonomics** are not identical (session + bridge vs one-shot). For **CI-style one-liners**, scripts today usually need **`mcpc connect … @session`** before **`mcpc --json @session …`**; an explicit **ephemeral mode** (§3.4) is the intended way to get **connect → one command → exit** without a bridge. **Optional implicit session** (§9.1) is a separate UX shortcut for users who still want a persistent session.

**Sample use cases: Inspector CLI → mcpc**

Assume a session already exists (`mcpc connect … @myserver`) where mcpc requires `@name` for tool/resource/prompt commands (unless an optional default session is implemented per §9.1). Use `--json` when you want machine-readable output comparable to the Inspector CLI.

| Inspector CLI pattern                                                  | mcpc equivalent (illustrative)                                                                                                                                                                                                                                                                                                                                                         |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--cli node build/index.js --method tools/list`                        | `mcpc --json @myserver tools-list` (after connecting that session to the stdio server defined in config or connect args)                                                                                                                                                                                                                                                               |
| `--cli --config mcp.json --server myserver --method tools/list`        | `mcpc connect /path/to/mcp.json:myserver @myserver` then `mcpc --json @myserver tools-list`                                                                                                                                                                                                                                                                                            |
| `--method tools/call --tool-name X --tool-arg k=v`                     | `mcpc --json @myserver tools-call X k:=v` (mcpc uses `name:=value` / `name=value` style args; see upstream mcpc docs)                                                                                                                                                                                                                                                                  |
| `--method resources/list`                                              | `mcpc --json @myserver resources-list`                                                                                                                                                                                                                                                                                                                                                 |
| `--method resources/templates/list`                                    | `mcpc --json @myserver resources-templates-list`                                                                                                                                                                                                                                                                                                                                       |
| `--method resources/read --uri <uri>`                                  | `mcpc --json @myserver resources-read "<uri>"`                                                                                                                                                                                                                                                                                                                                         |
| `--method prompts/list`                                                | `mcpc --json @myserver prompts-list`                                                                                                                                                                                                                                                                                                                                                   |
| `--method prompts/get --prompt-name X …`                               | `mcpc --json @myserver prompts-get X …`                                                                                                                                                                                                                                                                                                                                                |
| `--method logging/setLevel` / `--log-level debug`                      | `mcpc --json @myserver logging-set-level debug` (upstream command name)                                                                                                                                                                                                                                                                                                                |
| Remote URL + `--transport http --method tools/list`                    | `mcpc connect https://host @s` then `mcpc --json @s tools-list` (streamable HTTP is mcpc’s default for HTTPS URLs)                                                                                                                                                                                                                                                                     |
| Remote URL + SSE when path ends with **`/sse`** (or `--transport sse`) | Same path/heuristic in core; mcpc remains streamable-HTTP-first for typical HTTPS MCP endpoints—**SSE-only** servers must use an explicit SSE mode in the integrated product if we expose it. **Note:** `clients/cli/README.md` examples that use a bare origin URL without `/mcp` or `/sse` may **fail** resolution unless `--transport` is added; fix README when consolidating CLI. |

**Takeaway for this integration:** After donation and port, **one** CLI product covers both **session-oriented** workflows (connect, `@session`, bridge, OAuth, tasks, etc.) and—where we preserve it—the **legacy single-shot `--method`** path for scripts and launcher-based workflows. README, root docs, and release notes should describe a **single** CLI story and any flag migrations. **Ephemeral mode** (§3.4) and optional implicit session (§9.1) address different ergonomics gaps.

### 3.4 Ephemeral (“one-shot”) vs persistent session modes

The integrated CLI can support **two complementary lifecycles** without two MCP implementations—both use **`InspectorClient`** and the same config → **`MCPServerConfig`** mapping; the difference is **process topology** and **whether a bridge is involved**.

| Mode                       | User model                                                                                                                                        | Typical process shape                                                                                                                                         | Bridge                                                                          |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| **Ephemeral**              | Pass **server config** (or config ref) **and** a **command** in one invocation; run **one** MCP operation (or a small fixed sequence), then exit. | **Single** Node process: construct **`InspectorClient`**, **`connect()`**, dispatch the command (e.g. tools list/call), **`disconnect()`**, exit with status. | **Not required**—same pattern as today’s **`inspector --cli`** / `clients/cli`. |
| **Session (current mcpc)** | **`mcpc connect … @name`** leaves a **named** connection alive; later commands target **`@name`** via IPC.                                        | **Bridge** process holds **`InspectorClient`**; short-lived CLI processes send requests over the socket/pipe.                                                 | **Required** for this workflow.                                                 |

**Why both make sense**

- **Ephemeral** fits **automation, CI, launcher one-liners**, and **Inspector CLI parity** (`--method` style): no session file to clean up, no orphan bridge if the user forgets to disconnect.
- **Session** fits **interactive** use: many commands against one server, shell/tasks, proxy, and flows where paying **connect once** amortizes cost.

**Implementation notes (straightforward in principle)**

- **Shared core path:** Ephemeral handlers are a **thin layer** on **`InspectorClient`**—parse argv + optional `mcp.json` / server id, build config, run the same RPC helpers the bridge uses (or will use) after port.
- **OAuth:** Ephemeral runs still use **`OAuthManager`** and unified **`OAuthStorage`** (§2 goal 6, §5); first-time auth may open a browser or callback URL, then the process completes the flow and runs the command—no architectural conflict, only UX (timeouts, non-interactive flags).
- **stdio / secrets:** Ephemeral mode avoids putting server config on a **bridge** `argv` (§4.11), but the **CLI process** that spawns stdio servers must still avoid logging sensitive **`env`/`args`**; same hygiene as any local spawn.
- **HTTP session teardown / reconnect:** Ephemeral mode should use the same **`InspectorClient`** behavior as other clients (e.g. **`terminateSession`** policy in §4.7)—prefer fixes in **core** so one-shot and session modes stay consistent.

**Product / CLI design (open)**

- **Surface:** e.g. a dedicated subcommand (`mcpc run …`), a **`--ephemeral`** / **`--once`** flag, or preservation/extension of **`inspector --cli … --method`** as a compatibility entrypoint—choose one primary story for docs.
- **Relationship to §9.1:** Implicit **`@session`** improves **session** mode for single-server users; **ephemeral** mode solves **“no session at all”** for scripts. They can coexist.

---

## 4. Architectural mapping

### 4.0 Principle: one protocol object — extend core, do not bypass it

The **bridge process** owns **exactly one `InspectorClient` per session** (plus the same injected transports / `environment` seams as other Node clients). **There is no second “MCP client” layer** in the bridge to work around missing core behavior.

- If donated code needs a capability **`InspectorClient` does not expose yet** (e.g. **`notifications/message`** / server logging today), **add it in core**—typically **`setNotificationHandler` inside `connect()`** plus a **`dispatchTypedEvent`**—and have the bridge **subscribe to `InspectorClient`** (or use the new API) for IPC fan-out.
- **Do not** introduce **intermediate wrappers** whose purpose is to call **`getAppRendererClient()`** or the raw SDK **`Client`** to patch gaps. That defeats shared architecture; **extend `InspectorClient`** instead.

**IPC is not a second stack:** **`SessionClient` + `BridgeClient`** remain **only** the wire protocol between short-lived CLI processes and the long-lived bridge. They forward **requests/results**; they are **not** an alternate MCP implementation. Inside the bridge, **`handleMcpRequest`** maps IPC method names to **`InspectorClient`** methods (and shared helpers), not to a parallel `McpClient` class.

### 4.1 Replace donated `McpClient` with `InspectorClient` in the bridge

- **Bridge:** **`connect()` / `disconnect()`** on **`InspectorClient`** with **`createTransportNode`** (or a thin wrapped **`CreateTransport`** for auth / x402 / proxy fetch only—see §4.2). Remove **`McpClient`** from the bridge path.
- **CLI process:** Keep **`SessionClient`** as the **IPC façade** implementing whatever typed surface the CLI commands need (`listTools`, `callTool` by name, etc.). That façade **sends JSON over the socket**; it does **not** implement MCP. Optionally rename or narrow types so **`IMcpClient`** is clearly **“IPC command surface”**, not “another MCP client.”
- **Direct / one-shot modes** (e.g. legacy **`inspector --cli`**): **`InspectorClient`** in the same process—no bridge, no extra wrapper beyond what the command handler needs for argv parsing.

### 4.2 Transport: wrap `createTransportNode`, do not fork core

`InspectorClient` attaches `authProvider` from **`OAuthManager`** when HTTP + OAuth are configured (see `InspectorClient` constructor and `connect()`). **Donated** mcpc’s bridge uses a **custom `OAuthClientProvider`** (token manager + keychain); that is **legacy upstream shape**, not the **target** (§2 goal 6, §5): steady state is **`OAuthManager`** + unified storage, **without** injecting a parallel provider over core’s auth.

**During port:** a wrapped **`CreateTransport`** may still merge **x402 / proxy** into `createTransportNode` options. **`authProvider` overrides** are acceptable only as a **temporary** bridge to **`OAuthManager`** parity—same rule as Path **B** in §5.

**Fetch:** x402 and proxy behavior via **`environment.fetch`** on `InspectorClient` so the same base `fetch` is used consistently with core’s transport and auth tracking.

This matches the **environment isolation** model documented in [environment-isolation.md](environment-isolation.md).

### 4.3 Config mapping

| mcpc `ServerConfig`         | Core `MCPServerConfig`                                                                               |
| --------------------------- | ---------------------------------------------------------------------------------------------------- |
| `command` + `args` + `env`  | `StdioServerConfig` (`type` optional; defaults to stdio)                                             |
| `url` + `headers` + timeout | `{ type: "streamable-http", url, headers?, requestInit? }` (match mcpc’s default of streamable HTTP) |

**Timeouts:** mcpc uses **seconds** on server config and may put **AbortSignal** on `requestInit`; core’s `InspectorClient` uses **`options.timeout` in milliseconds** for SDK request options — map explicitly.

### 4.4 `InspectorClient` options for mcpc-like behavior

Likely defaults for bridge/CLI (tune per command):

- **`sample: false`**, **`elicit: false`** (unless we want parity with interactive sampling/elicitation).
- **`progress`:** align with current mcpc UX.
- **`pipeStderr: false`** for stdio unless stderr should surface in logs.
- **List-changed:** core dispatches **`toolsListChanged`**, **`resourcesListChanged`**, **`promptsListChanged`**, etc. Replace SDK `listChanged` callbacks with **`addEventListener`** on those events (and refresh caches / IPC broadcast in bridge).

### 4.5 RPCs and `callTool` shape

**Gap:** `InspectorClient.callTool(tool, args, …)` expects a **`Tool`** object (schema-aware args, task-required tools). Donated mcpc IPC uses **tool name + arguments** (no `Tool` instance on the wire).

**Resolution (core-first, no raw-Client bypass):**

1. **Preferred:** Add a **`InspectorClient`** API that accepts **name + args** (e.g. **`callToolByName`**) that **resolves** the tool via **`listTools` / cache**, then delegates to existing **`callTool`** / **`callToolStream`** (including task-required tools). Implement once in core; bridge and CLI benefit.
2. **Alternatively:** Bridge keeps a **tool list cache** updated on **`toolsListChanged`**, resolves **name → `Tool`**, then calls **`inspectorClient.callTool(tool, args, …)`**—still **only `InspectorClient`**, no intermediate MCP object.

Task-augmented calls: use **`InspectorClient.callToolStream`** and existing task helpers in core; IPC **`callTool`** with **`useTask`** maps to those methods.

### 4.6 Server logging (`notifications/message`)

**Today:** Donated bridge registers **`LoggingMessageNotificationSchema`** on the SDK **`Client`** and forwards over IPC.

**Target:** **`InspectorClient.connect()`** registers the same notification and **`dispatchTypedEvent`**s a stable event (e.g. **`loggingMessage`**). Bridge **`addEventListener`**s on **`InspectorClient`** and forwards to IPC clients—**no `setNotificationHandler` in bridge code**, no **`getAppRendererClient()`**.

### 4.7 MCP session id and HTTP session teardown

- **Session id (read after connect):** **`MessageTrackingTransport`** (`core/mcp/messageTrackingTransport.ts`) defines **`get sessionId()`** and returns **`this.baseTransport.sessionId`**, so the wrapped streamable HTTP transport’s server-assigned id is visible on the **same** object passed to **`Client.connect()`**. **`InspectorClient`** does **not** expose `baseTransport` or session id on its **public** API today (`baseTransport` is private); code that needs the id (e.g. to persist **`mcp-session-id`** for resume) must **retain a reference** to the transport returned from the custom **`CreateTransport`** / factory, or add a small **`InspectorClient`** accessor in core.
- **Session id (resume before connect):** The MCP SDK’s **`StreamableHTTPClientTransport`** constructor accepts **`sessionId`** in its options object and sends **`mcp-session-id`** on requests when set (`@modelcontextprotocol/sdk` `client/streamableHttp.js`). **`StreamableHttpServerConfig`** in core (`core/mcp/types.ts`) has **no** `sessionId` field, and **`createTransportNode`** does not pass one—so resume-via-config requires extending the config + factory or building the transport outside **`createTransportNode`**.
- **Teardown:** Donated **`McpClient.close()`** calls **`transport.terminateSession()`** (SDK HTTP DELETE) before **`client.close()`**. **`InspectorClient.disconnect()`** (`core/mcp/inspectorClient.ts`) calls **`this.client.close()`** and clears transport references but **does not** call **`terminateSession()`** on the underlying streamable HTTP transport.

**Resolution (pick one or combine):**

- Extend **core** `disconnect()` to call **`terminateSession`** on the base transport when `typeof (transport as { terminateSession?: () => Promise<void> }).terminateSession === 'function'` (benefits all clients), **or**
- In the CLI/bridge only, keep a handle to the **base** transport from the transport factory and call **`terminateSession()`** before **`inspector.disconnect()`**.

### 4.8 Streamable HTTP `sessionId` (resume) on config

**Current core types:** **`StreamableHttpServerConfig`** includes **`type`**, **`url`**, **`headers`**, **`requestInit`** only—**no** `sessionId` (`core/mcp/types.ts`). **`createTransportNode`** instantiates **`StreamableHTTPClientTransport`** with **`authProvider`**, **`requestInit`**, and **`fetch`** only (`core/mcp/node/transport.ts`).

**Resolution:** extend **`StreamableHttpServerConfig`** + **`createTransportNode`**, **or** construct streamable HTTP transports in a CLI-specific factory that mirrors core’s fetch-tracking behavior.

### 4.9 Proxy / reconnection behavior

- **Fetch / proxy:** **`createTransportNode`** uses **`options.fetchFn ?? globalThis.fetch`** as the base for the streamable HTTP transport (`core/mcp/node/transport.ts`). It does **not** wire **Node `HTTP_PROXY` / `HTTPS_PROXY`** the way mcpc’s **`proxyFetch`** does unless **`fetchFn`** is supplied (e.g. **`InspectorClient`** **`environment.fetch`** or a wrapped **`CreateTransport`**). Donated **`createStreamableHttpTransport`** defaults to **`proxyFetch`** when no custom fetch is passed (`mcp-cli` `src/core/transports.ts`).
- **Reconnection:** The SDK’s **`StreamableHTTPClientTransport`** applies **default** **`reconnectionOptions`** when omitted: **`initialReconnectionDelay: 1000`**, **`maxReconnectionDelay: 30000`**, **`reconnectionDelayGrowFactor: 1.5`**, **`maxRetries: 2`** (verified in **`@modelcontextprotocol/sdk`** `dist/esm/client/streamableHttp.js`). **`createTransportNode`** does **not** pass **`reconnectionOptions`**, so those SDK defaults apply. mcpc overrides with **longer retry** (**`maxRetries: 10`**, **`reconnectionDelayGrowFactor: 2`**, same 1s / 30s bounds) in **`createStreamableHttpTransport`**. Expect **fewer automatic reconnects** in core unless the factory passes mcpc-equivalent options.

### 4.10 Proxy MCP server (bridge)

**`ProxyServer`** implements an MCP **server** forwarding to the upstream client. **Core does not provide a server abstraction.** Keep **`@modelcontextprotocol/sdk` server imports** in mcpc (bridge + e2e) unless we later extract shared server helpers into core (out of scope for a minimal port).

### 4.11 Bridge process `argv`: hiding stdio secrets and observability

**Donated behavior:** The bridge is spawned with **`sessionName`** and a **JSON `ServerConfig`** on the command line. **`headers`** are **removed** from that JSON so **`ps`** does not show HTTP auth material; **OAuth** and real headers are sent **after** the socket exists via **`set-auth-credentials`** IPC.

**Gap:** For **stdio** servers, secrets can live in **`args`** or **`env`** as easily as in **`headers`**. Those fields remain inside the stringified JSON on **`argv`**, so they are still visible to **`ps`**, crash reporters, and log aggregation—**the same class of leak** headers were meant to avoid.

**Directions for the integrated CLI:**

1. **Extend the “strip from `argv`” idea** — also **omit or redact** **`env`** / **`args`** entries known to be sensitive (fragile without a schema; easy to miss keys), or **store** stdio secrets in the **keychain** / sidecar and pass **references** only.

2. **Prefer: minimal `argv` + full config over IPC** — pass only a **stable identifier** on the command line (e.g. **session name**, **`--verbose`**) so **`ps`** can still **correlate a PID to a session**, and send the **complete `ServerConfig`** (including **`command`**, **`args`**, **`env`**, **`url`**, **`headers`**) in one or more **IPC messages** after the socket is listening, **before** `connectToMcp()`. This matches the existing pattern for auth and avoids putting **any** transport secrets on **`argv`**.

Choose explicitly during implementation; **(2)** is the stronger default if stdio secrets are in scope.

---

## 5. OAuth

**Target end state (aligned with §2 goal 6):** **Path A** — **`OAuthManager`** inside **`InspectorClient`** drives the flow for the integrated CLI/bridge the same way it does for TUI and (once the web factory is updated) the browser. **No** long-lived parallel **`OAuthTokenManager`** / custom **`OAuthClientProvider`** that bypasses **`OAuthManager`** while other clients use core. **Credential persistence** is **not** a separate product choice: **one** Node **`OAuthStorage`** implementation in core (§5.1) and **`RemoteOAuthStorage`** for the web app against the same **`oauth`** store.

**Migration paths (implementation sequencing only):**

| Path                                                        | Description                                                                                                                                                                                                                                  |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **A. Converge on core `OAuthManager` (required end state)** | Map mcpc profiles + keychain + bridge IPC timing into **`environment.oauth`** + **`options.oauth`**; bridge **`InspectorClient`** uses the same events and storage as TUI/CLI.                                                               |
| **B. Temporary spike / incremental port**                   | Optionally keep **`OAuthTokenManager` + `OAuthProvider`** injected via a wrapped **`createTransportNode`** **only** for early wiring or tests—**must not** ship as the steady state, or creds diverge from TUI/web (**violates §2 goal 6**). |

**Sequencing note:** It may be practical to land **storage** unification (§5.1) before or in parallel with **OAuthManager** wiring, but both are required for the stated product goal. If **B** is used at all, treat it as a **short-lived** bridge to **A**, not an alternative architecture.

### 5.1 Unified credential storage (keychain-first on Node)

**Problem:** Today, **Node clients** in core use **`NodeOAuthStorage`**, backed by **`getOAuthStore()`** → **`createFileStorageAdapter`** on **`getStoreFilePath(getDefaultStorageDir(), "oauth")`**. On Unix that resolves to **`~/.mcp-inspector/storage/oauth.json`** (see **`core/storage/store-io.ts`** **`getDefaultStorageDir`** and **`core/auth/node/storage-node.ts`**; the file is written with mode **`0600`** via **`writeStoreFile`**). The Zustand store holds OAuth client info, tokens, PKCE verifiers, scopes, etc.—**secrets on disk** in one JSON document.

**Donated mcpc** uses **`@napi-rs/keyring`** for **OS keychain**, with file fallback **`${getMcpcHome()}/credentials.json`** (typically **`~/.mcpc/credentials.json`**, mode **`0600`**, **`withFileLock`**) when the keychain is unavailable (`mcp-cli` `src/lib/auth/keychain.ts`). **Profile metadata** lives in **`~/.mcpc/profiles.json`**; **sessions** in **`~/.mcpc/sessions.json`** (`src/lib/utils.ts`). **Service name** for keychain entries is the string **`mcpc`** (same file).

**Strategic fit:** **All Node consumers** (CLI, TUI, dev server, bridge) must use **one** credential persistence implementation **in core**—keychain-first with file fallback—not a second storage stack living only in the ported CLI package.

**By environment (this is not a “pick one” table):** **`InspectorClient`** already selects storage via **`environment.oauth.storage`**. **Node** uses a **Node** `OAuthStorage` implementation (today file-based; target keychain-capable, **defined in `core/auth/node/`**). **Browser (Inspector web app)** should ultimately use **`RemoteOAuthStorage`** against the Inspector API’s **`/api/storage/oauth`** so tokens match **`NodeOAuthStorage`** on the server host; **`createWebEnvironment()`** still uses **`BrowserOAuthStorage`** (sessionStorage) until that switch is implemented—see [environment-isolation.md](environment-isolation.md). **Those adapters do not use the OS keychain**—there is no keyring in the browser bundle. That is expected and unrelated to the Node work below.

**Node implementation options (both are core-only; pick one for how much moves off disk):**

| Option                               | Summary                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Adopt (mcpc-style split in core)** | Lift or rewrite **keychain + file fallback** in **`core/auth/node/`** as an **`OAuthStorage`** (or layered secret store + JSON metadata). Align metadata files with something like mcpc’s **profiles** vs **secrets** split.                                                                                                                                      |
| **Hybrid (minimal churn)**           | New **`NodeOAuthKeychainStorage`** (or evolve **`NodeOAuthStorage`**) in **core**: **tokens** and **`client_secret`** in keychain; **non-secret** OAuth fields (e.g. PKCE verifiers, non-sensitive client registration metadata) stay in the existing **single JSON store** until a later migration. **Same class**, still **one** default for every Node client. |

**Not acceptable:** Keeping **`NodeOAuthStorage`** as **file-only secrets** in core while the ported CLI alone talks to the keychain. That duplicates persistence and breaks **shared credentials across Node clients**—it is the **pre-port anti-pattern**, not a third legitimate option.

**Engineering considerations**

- **Native dependency:** **`@napi-rs/keyring`** is a **native addon**; it must stay **out of browser bundles** and load only in **Node** entrypoints (lazy `import()` as mcpc does is a good pattern).
- **Service name:** Today mcpc uses a fixed **keychain service name** (e.g. `mcpc`). After donation, standardize on an **Inspector-owned** service string (e.g. `modelcontextprotocol.inspector`) and define **account key** format (server URL + profile + purpose) so CLI, TUI, and launcher do not collide with unrelated apps.
- **Migration:** Existing users with **`~/.mcp-inspector/storage/oauth.json`** need a **one-time migration** (read old file, write secrets to keychain, optionally retain non-secrets in JSON) or a **dual-read** period.
- **mcpc `profiles.json` vs core store:** mcpc separates **profiles** from keychain entries; core today folds more into one file. Align **on-disk schema** (what is metadata vs secret) so **all Node clients** share the same logical config. **Web:** after **`RemoteOAuthStorage`** is wired in **`createWebEnvironment()`**, OAuth state is the **same** Zustand-backed blob as Node for that host—not a second semantic world.
- **Tests / CI:** Keychain is often **absent** in CI; tests must use **file fallback** or inject a custom **`OAuthStorage`** via **`InspectorClientOptions.environment.oauth.storage`** (see **`InspectorClientEnvironment.oauth.storage`** in **`core/mcp/types.ts`**).
- **Headless Linux:** Same operational notes as mcpc (e.g. **libsecret**, **dbus**); document in README.

**Assessment:** **Yes, it is reasonable—and aligned with product goals—to move Node credential storage in core toward the mcpc model** (keychain with file fallback), implemented **inside core** so **CLI, TUI, and any Node `InspectorClient`** user share credentials without duplicating modules. Exact API (full **`OAuthStorage`** replacement vs. internal secret backend) should be decided during implementation; prefer **`OAuthStorage`** compliance so **`OAuthManager`** and future adapters stay unchanged at the boundary.

---

## 6. Testing and CI

- Donated upstream uses **Jest**; the monorepo largely uses **Vitest**. Choose per workspace: migrate tests to Vitest for consistency, or keep Jest isolated in the new CLI package directory.
- Preserve e2e coverage; update paths if the repo layout differs from upstream mcp-cli.

---

## 7. License, naming, donation, and publishing

- **Donation:** Apify contributes the **mcp-cli / mcpc** source; the MCP Inspector maintainers **adapt** it and **replace** the existing **`inspector-cli`** implementation. Legal review should cover **copyright assignment or inbound license**, **third-party notices**, and **trademark** (e.g. `mcpc` name and any Apify-specific branding in UX copy).
- **License:** Upstream mcpc is **Apache-2.0**; the Inspector repo uses a different root license. Decide whether donated files keep **Apache-2.0** headers, are **relicensed** under the repo license with permission, or use **dual-license** / **NOTICE** aggregation—**before** merging the donation.
- **Package naming:** The **primary** published CLI for Inspector will likely remain **`@modelcontextprotocol/inspector-cli`** (or the workspace name that replaces it), not a separate **`@apify/mcpc`** consumer package, unless we deliberately publish **both** for backward compatibility during a transition. Global **`mcpc`** / **`mcpc-bridge`** binaries may be **renamed** or **aliased** to match Inspector branding; decide explicitly.

---

## 8. Fork and branch workflow (maintainer fork)

- **`main`** on the fork should track **`upstream/main`** when “sync main” is requested.
- **`v1.5/main`** must be synced explicitly from **`upstream/v1.5/main`**; it does **not** update when only `main` is merged.
- Feature work for this integration: branch **`mcpc`** from updated **`v1.5/main`**, push to **`origin/mcpc`**.

---

## 9. Open questions

1. **Donation mechanics:** Inbound contribution agreement, **CLA / DCO**, and whether Apify **archives** or **redirects** the original repo after donation.
2. **CLI compatibility:** **Strict** preservation of `inspector --cli` + `--method` vs. **documented breaking** migration; timeline for removing any shim. Overlaps **ephemeral mode** (§3.4): legacy flags may map to that path.
3. **Binary and package names:** Keep **`mcpc`** / **`mcpc-bridge`** as user-facing names vs. **`inspector`** subcommands only; npm package naming (single **`inspector-cli`** vs. transitional dual publish).
4. **OAuth + storage implementation detail:** **End state is decided** (§2 goal 6, §5): **`OAuthManager`** + unified **`OAuthStorage`** / **`RemoteOAuthStorage`**. Remaining work is **how** to sequence migration from donated code and **web factory** change, not whether to converge.
5. **Credential storage mechanics:** **Keychain-first `OAuthStorage` for Node** (§5.1): **service/account naming**, **migration** from file-based `oauth.json`, **profiles** layout vs. monolithic state file—implementation choices under the unified-store requirement.
6. **Core contributions:** Which transport gaps (**`sessionId` on HTTP config**, **`terminateSession` in disconnect**, reconnection defaults) should be fixed in **core** vs. CLI-only shims?
7. **Phasing:** Single big PR vs. phases (e.g. land donation behind flag, swap launcher default, remove old `clients/cli`; storage migration may be its own release).

### 9.1 Open issue: optional session target (default / last-used)

**Idea:** Allow MCP subcommands to **omit** the `@session` argument when the user has a single obvious session—e.g. resolve to the **last-used** (or **default**) session, updated whenever a connection succeeds or a command runs against a named session. That matches the mental model “I only use one server” and moves mcpc closer to **Inspector CLI** ergonomics (`mcpc --json tools-list` after one `connect`, instead of repeating `@myserver` on every line).

**Assessment**

| Topic             | Notes                                                                                                                                                                                                                                                                                                                                           |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **User value**    | Less repetition for the dominant **single-session** case; scripts and docs become shorter; mental overlap with “current directory” or “current kubectl context”.                                                                                                                                                                                |
| **Compatibility** | **Additive** if `@name` remains supported and behavior when omitted is well-defined. Existing scripts that pass `@session` are unchanged.                                                                                                                                                                                                       |
| **Semantics**     | Define **one** rule and document it: e.g. **`lastActiveSession`** (session name last successfully used for any MCP command), set on `connect` and on each explicit `@name` use. Alternatives: “most recently created” only (weaker) or explicit **`mcpc use @name`** to pin default (clearer for multi-session users).                          |
| **Multi-session** | With several active sessions, implicit default can surprise users (wrong server). Mitigations: print which session is implicit in **`--verbose`** or human mode; **`mcpc`** with no args lists sessions and marks **default**; require explicit **`@name`** when default is ambiguous (e.g. none set, or policy “never implicit if ≥2 active”). |
| **Safety / CI**   | Pipelines should prefer **explicit `@name` or `--session <name>`** so runs do not depend on local machine state. Consider **`MCPC_SESSION`** env var for CI instead of implicit file state, or document that implicit default is **interactive-first**.                                                                                         |
| **Parsing**       | Upstream grammar is **`mcpc @session tools-list`**. Omitting `@session` works if subcommand names (`tools-list`, `connect`, …) are reserved and cannot collide with session names (session names already require `@` prefix today, so **`mcpc tools-list`** is syntactically free).                                                             |
| **Persistence**   | Store default in **`sessions.json`** (or adjacent metadata): `lastActiveSession`, optional `pinnedDefaultSession` if `use` is added. On **`disconnect`** / session delete, clear or recompute default.                                                                                                                                          |
| **Failure modes** | No default and no `@session`: error message with **`mcpc connect … @name`** hint. Default session **crashed / expired**: existing bridge recovery paths apply; consider auto-clearing stale default after hard failure.                                                                                                                         |
| **Scope**         | Applies to commands that currently use **`withMcpClient(@name, …)`**; **`connect`**, **`login`**, and global listing likely keep current argument patterns.                                                                                                                                                                                     |

**Recommendation:** Treat as a **post-port UX enhancement** unless we want it in the first integration milestone; design **`mcpc use @name`** (explicit pin) plus **`lastActiveSession`** fallback for the lowest surprise factor.

---

## 10. Appendix: sources verified (this pass)

The following were read or grepped in the **Inspector `v1.5`-line workspace** and parallel **`mcp-cli`** checkout to replace hand-wavy claims:

| Topic                                                                                 | Where                                                           |
| ------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| OAuth file path + mode                                                                | `core/auth/node/storage-node.ts`, `core/storage/store-io.ts`    |
| `InspectorClientEnvironment.oauth.storage`                                            | `core/mcp/types.ts`                                             |
| `resolveServerConfigs` URL / transport rules                                          | `core/mcp/node/config.ts`                                       |
| `createTransportNode` (streamable HTTP args)                                          | `core/mcp/node/transport.ts`                                    |
| `StreamableHttpServerConfig` fields                                                   | `core/mcp/types.ts`                                             |
| `MessageTrackingTransport.sessionId`                                                  | `core/mcp/messageTrackingTransport.ts`                          |
| `InspectorClient.disconnect` / private `baseTransport`                                | `core/mcp/inspectorClient.ts`                                   |
| SDK streamable HTTP defaults (`reconnectionOptions`, `sessionId`, `terminateSession`) | `@modelcontextprotocol/sdk` `dist/esm/client/streamableHttp.js` |
| mcpc proxy fetch + reconnection constants                                             | `mcp-cli/src/core/transports.ts`                                |
| mcpc keychain service name + paths                                                    | `mcp-cli/src/lib/auth/keychain.ts`, `mcp-cli/src/lib/utils.ts`  |
| Inspector CLI supported `--method` values                                             | `clients/cli/src/cli.ts`                                        |
| Node engines                                                                          | Root `package.json`                                             |

**Core JSDoc:** `NodeOAuthStorage` default path in **`core/auth/node/storage-node.ts`** was corrected to match **`~/.mcp-inspector/storage/oauth.json`** (same research pass).

---

## 11. Revision history

| Date       | Note                                                                                                                                                                                                          |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-04-04 | Initial design doc from porting assessment; branch `mcpc` on `v1.5/main`; fork sync caveat documented.                                                                                                        |
| 2026-04-04 | §3.3: Inspector CLI parity, differences, sample use-case mapping.                                                                                                                                             |
| 2026-04-04 | §9.1: Open issue — optional session / last-used default for single-session UX and CLI parity.                                                                                                                 |
| 2026-04-04 | Goal: Apify donates source; adapt on core; **replace** existing Inspector CLI (§1–3, §7, §9).                                                                                                                 |
| 2026-04-04 | §5.1: Unified Node credential storage — keychain-first (mcpc model) in core; §9 expanded.                                                                                                                     |
| 2026-04-04 | Research pass: §3.2–3.3 transport rules, §4.7–4.9 teardown/sessionId/reconnect/proxy, §5.1 paths, §10 appendix + core JSDoc caveat.                                                                           |
| 2026-04-04 | §4.0–4.6: Bridge uses **`InspectorClient` only**; extend core for gaps (logging, `callTool` by name); no intermediate MCP wrapper / **`getAppRendererClient`** hacks.                                         |
| 2026-04-04 | §4.11: Bridge **`argv`** — extend header-style stripping to **`args`/`env`**, or minimal **`argv` + full config over IPC** (session name for **`ps`**).                                                       |
| 2026-04-04 | §5.1: Clarify Node vs browser storage; remove misleading **status quo** table row; **not acceptable** = CLI-only keychain.                                                                                    |
| 2026-04-04 | §5.1: Browser — target **`RemoteOAuthStorage`** via Inspector API; **`createWebEnvironment()`** still **`BrowserOAuthStorage`** until implemented ([environment-isolation.md](environment-isolation.md)).     |
| 2026-04-04 | §2 goal 6 + §5: Product requirement — **same OAuth code path** (`OAuthManager`) and **same cred storage** (Node core store + web **`RemoteOAuthStorage`**); Path B only as temporary spike, not steady state. |
| 2026-04-04 | §3.4: **Ephemeral** (one-shot, no bridge) vs **session** (bridge) CLI modes — rationale, implementation notes, open UX surface.                                                                               |
