# Design: Alfresco REST / Web Script discovery, action invocation and Acosix Keycloak

Status: initial architecture analysis (no runtime implementation yet)
Target branch: `feature/alfresco-api-discovery-keycloak`

## Goal

Extend `python-alfresco-mcp-server` so an MCP client can:

1. discover available Alfresco standard REST endpoints and custom plugin endpoints,
2. inspect the exact method, URL, parameters, request body and response format of each operation,
3. invoke an allowed operation with validated arguments,
4. discover and run repository *actions* with their declared parameters,
5. authenticate against Alfresco Community installations configured with [Acosix alfresco-keycloak](https://github.com/Acosix/alfresco-keycloak), preserving Alfresco-side permissions.

Discovery, specification parsing, execution, and authentication must be separate components. Custom Web Scripts are **not inherently equivalent to OpenAPI endpoints**, and repository actions are **not the same as Web Scripts**.

## Existing code (reviewed)

- `alfresco_mcp_server/fastmcp_server.py`: FastMCP 3 entry point, currently exposes a fixed set of search/core tools and a repository-information resource.
- `alfresco_mcp_server/utils/connection.py`: `ClientFactory` from `python-alfresco-api`; supports `basic`, `ticket`, and OAuth2 bearer using `OAuth2AuthUtil`. Current master-client cache is global.
- `alfresco_mcp_server/config.py`: configuration fields for the same auth choices.
- `fastmcp_server.py`: optional JWT verification for HTTP/SSE **MCP transport**, distinct from authentication to the Alfresco back end.
- `pyproject.toml`: FastMCP 3, `python-alfresco-api`, `httpx`, Pydantic, PyYAML.

This means OAuth2 is **not a greenfield feature**. The important work is compatibility with the Acosix subsystem, correct user-token handling and authorization boundaries, not simply adding a new bearer-token configuration variable.

## Architecture proposal

```text
MCP client
    |
    +-- list_alfresco_operations(source?, query?, only_enabled?)
    +-- describe_alfresco_operation(operation_id)
    +-- invoke_alfresco_operation(operation_id, arguments)
    +-- list_alfresco_actions(node_id?)
    +-- execute_alfresco_action(action_id, node_id, parameters)
    |
Operation registry
    +-- standard OpenAPI source (configured URL / packaged spec)
    +-- explicit plugin manifests (OpenAPI or JSON/YAML descriptors)
    +-- optional Web Script index adapter (best effort, opt-in)
    +-- repository action-definition adapter (version-dependent)
    |
Schema validation + execution policy + HTTP executor
    |
Authentication provider (current-user bearer or explicit service identity)
    |
Alfresco REST API / custom Web Scripts / action execution
```

### 1. Canonical operation descriptor

The registry should normalize every source to a common model, with at least:

| Field | Meaning |
| --- | --- |
| `id` | Stable namespaced ID such as `rest:alfresco:nodes.get` or `webscript:my-plugin:run` |
| `source` / `origin` | OpenAPI, plugin manifest, Web Script index, action definition; include provenance and version |
| `method` / `path_template` | HTTP verb and relative, allowlisted URL template |
| `parameters` | Required/optional path, query and allowed header parameters, with JSON Schema types, defaults and constraints |
| `request_body` | JSON Schema and content type; optionally multipart/form-data |
| `response` | Expected response content type / schema when known |
| `auth_requirements` | Authentication/permission hints; server remains final authority |
| `execution_policy` | Read-only, mutating, sensitive, disabled; explicit allowlist |
| `enabled` | Enabled only if sufficiently described and permitted by configuration |

Unknown required parameters or unparseable descriptors must result in a **non-executable** catalog item, not guessed arguments.

### 2. Discovery providers

**Standard REST**: parse an administrator-supplied OpenAPI document, or a specifically configured trusted OpenAPI URL. Do not assume all Alfresco deployments publish a live complete OpenAPI specification. Confirm OpenAPI version, server URL, base path and compatibility.

**Custom Web Scripts**: support explicit plugin manifests, or import `*.desc.xml` descriptors from the customer's deployed extension/source packages. Web Script descriptors typically describe URI templates, HTTP method (including naming convention), authentication and output formats, but may not fully describe body/property parameters. Supply missing parameter schemas through an accompanying manifest. A server-side Web Script index can be offered as an optional discovery aid **only after checking its availability and access controls**; HTML/index listings should not be treated as a complete machine-readable contract. An API can be installed and routable even if it cannot be discovered automatically by the MCP server.

**Actions**: query the action-definition/parameter metadata exposed by the installed Alfresco version, if supported; confirm exact REST paths and capabilities against a target instance before implementation. Where no suitable API exists, an explicitly installed custom Web Script may expose action definitions/execution. Keep action parameters (types, mandatory status, allowed values, multi-value semantics) separate from ordinary HTTP query arguments.

**Conflict handling**: namespace operations, track source/manifest hash and version, use deterministic precedence for overrides, and do not silently replace one operation with another. Refresh discovery explicitly or with bounded caching.

### 3. Tool surface and parameters

Start with generic tools: `list_alfresco_operations`, `describe_alfresco_operation`, `invoke_alfresco_operation`, `list_alfresco_actions`, and `execute_alfresco_action`.

`describe_*` must return the full JSON Schema for arguments. `invoke_*` must validate provided parameters **before** sending requests. Where supported by the installed FastMCP SDK, consider registering strongly typed, generated MCP tools from a trusted descriptor catalog at server startup. A single generic tool with `arguments: dict` is useful as a fallback, but does not advertise each operation's parameter schema directly in `tools/list`. Evaluate dynamic tool registration and list-changed notifications before promising hot reload.

Invocation should:
- resolve the operation ID from the registry, not accept arbitrary URLs,
- substitute and percent-encode path parameters correctly,
- serialize typed query parameters (including arrays and booleans),
- validate JSON/multipart bodies, enforce size/time limits,
- use the same authenticated HTTP stack for both standard REST and custom Web Scripts,
- report JSON or bounded text errors cleanly without echoing credentials.

### 4. Acosix Keycloak: compatibility is distinct from native identity-service

The [Acosix extension](https://github.com/Acosix/alfresco-keycloak) supplies its **own** authentication subsystem, different from Alfresco's `identity-service`. It supports bearer access tokens, user/role mapping and SSO on the Repository tier. This matters: current project documentation frames its OAuth2 option primarily around the built-in `identity-service`; that wording and assumptions need adjustment for Acosix installations.

Important Acosix configuration/details verified from its documentation:

- Typical chain: `authentication.chain=alfrescoNtlm1:alfrescoNtlm,keycloak1:keycloak`.
- `keycloak.adapter.auth-server-url`, `keycloak.adapter.realm`, `keycloak.adapter.resource` and `keycloak.adapter.credentials.secret` configure the adapter.
- `keycloak.adapter.verify-token-audience` is relevant to which client audience is accepted; preserve audience validation and arrange Keycloak audiences/token exchange appropriately.
- `keycloak.authentication.sso.handlePublicApi` is documented as `false` by default. **Test actual bearer handling separately for** `/alfresco/api/...` **and** Web Script endpoints `/alfresco/service/...` or `/alfresco/wcs/...`. Do not assume browser SSO redirects are suitable for machine clients.
- Acosix role mapping and group synchronization affect effective permissions; final authorization is performed by Alfresco.

**Two independent auth boundaries**:

1. **MCP client -> MCP server**: currently optional FastMCP JWT verifier for HTTP/SSE. Production configurations should verify signature **and** expected issuer/audience; a signature check alone does not prove the token was issued for this MCP service.
2. **MCP server -> Alfresco**: send an Alfresco-accepted bearer token using current `OAuth2AuthUtil`, or a narrowly authorized dedicated service identity where appropriate. Preserve per-user ACLs when acting for a user.

Prefer OIDC Authorization Code + PKCE for interactive user login (or an existing trusted client passing an appropriately scoped user access token); avoid adding password grant as the preferred path. Explicitly evaluate token audience and subject mapping for an Acosix deployment before enabling any service-account mode. Never log tokens, put them in URLs, or accept an arbitrary `Authorization` header supplied as a tool argument. A shared global client cache must not accidentally reuse user A's credentials for user B; implement request-scoped identity handling or a clearly documented single-service-identity deployment.

### 5. Safety defaults

- Discovery from trusted, configured sources only. No arbitrary URLs or host headers; reject absolute URLs, traversal and origin changes in a discovered operation.
- Default deny for mutating or destructive operations. Admin opts in to specific methods/operation IDs, and MCP callers must still have Alfresco-side permissions.
- Keep reads and mutations visibly distinct; explicitly flag deletion, action execution, downloads/uploads and externally visible effects.
- Enforce bounded response size, timeouts, redirect handling, TLS verification and sanitized diagnostics.
- Do not infer an endpoint is safe because it uses GET; Web Scripts may have nonstandard side effects.
- Never execute discovered operations during catalog scans.

### 6. Suggested implementation sequence

1. Add immutable descriptor models, JSON Schema validation and offline fixture-based tests.
2. Add manifest/OpenAPI loaders and the list/describe MCP tools. Add custom Web Script fixtures.
3. Implement allowlisted HTTP executor; verify correct encoding, request-body handling, errors and authorization.
4. Add action-definition discovery/execution with target Alfresco version tests.
5. Add Acosix-specific integration tests for user bearer tokens, token expiry, audience, 401/403 and role/ACL enforcement.
6. Only then evaluate live Web Script index scraping, dynamic per-operation tools and hot reload.

### Test matrix before release

| Situation | Required outcome |
| --- | --- |
| Standard REST OpenAPI operation | Parameters and HTTP method represented accurately |
| Custom Web Script with complete manifest | Discoverable and invokable with correct validated parameters |
| Custom Web Script lacking parameter details | Visible as incomplete, execution disabled |
| Action with required enum/multiple parameters | Invalid values rejected locally; valid values serialized correctly |
| Untrusted URL in a descriptor | Blocked before network request |
| Unauthenticated / expired / wrong-audience token | No credentials leaked; proper 401/403 response |
| Acosix user without permission | Alfresco denies action; no service-account fallback |
| Concurrent different MCP users | No user identity/token crossover |
| Web Script index unavailable | Normal operation through preloaded manifests |
| Non-JSON or redirect-to-login responses | Safe error, no HTML treated as success |

## Open questions to validate against a real Alfresco environment

- Exact Alfresco Community version(s), Acosix addon release, Keycloak release and relevant `authentication.chain` settings.
- Whether custom plugins ship `*.desc.xml`, an OpenAPI document, custom Java action executors or only Java/JavaScript Web Script controllers.
- Which endpoint families and mutations administrators want exposed to MCP users.
- Whether MCP is single-service-user, single local user or multi-user HTTP; this changes token lifecycle and cache isolation requirements.

These questions affect compatibility, not the core registry/descriptor approach. No live Alfresco or Keycloak server has been connected or queried during this analysis.
