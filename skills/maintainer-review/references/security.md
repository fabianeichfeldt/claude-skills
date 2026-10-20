# Security review

The security aspect of the maintainer review. You are an attacker with the diff in hand: every input the change accepts is yours to control, every identifier yours to swap, and every check is something to get around. A vector is **open** when you can trace a path from attacker-controlled input to the harmful effect without hitting a control that stops it. Report the path, not just the pattern: `file:line` where the input enters, `file:line` where it does harm, and what is missing in between.

Judge controls in the code that actually runs. A check in a sibling endpoint, a comment saying "validated upstream", or a framework feature that is turned off elsewhere in config does not count until you have found it on this path. Read the surrounding code (middleware, filters, base classes, route config) to find what the diff inherits.

The review is done when every entry point is in the endpoint table and every category of each pass that applies has either a finding or has been checked against the diff.

## 1. Endpoints

List every entry point the diff adds or changes: HTTP routes and controllers, GraphQL queries, mutations and resolvers, gRPC methods, webhooks and callbacks, message and event consumers, file and import handlers, CLI commands, scheduled jobs, MCP tools, resources and prompts. An existing endpoint whose handler, request model or authorization changed counts.

For each, check against the OWASP API Security Top 10 (2023):

- **AuthN** (API2): is authentication required, through the repo's usual mechanism? An anonymous endpoint needs a reason the goal states. Token and session handling: expiry, signature verification, audience, no secrets in URLs.
- **AuthZ, object level** (API1): every ID taken from the path, query, body or headers is checked against the caller's ownership or tenant before it is read or changed. Swapping the ID for someone else's must fail.
- **AuthZ, function level** (API5): admin and privileged operations check the role or permission, not just "logged in". Same for HTTP methods: a protected GET with an unprotected DELETE on the same route is a finding.
- **AuthZ, property level** (API3): request binding accepts only the fields the caller may set (no mass assignment of `role`, `ownerId`, `isAdmin`, `price`); responses return only the fields the caller may see (no whole entities serialized, no internal IDs, hashes or PII the client doesn't need).
- **Input**: type, length, range and format validated at the boundary, in the repo's usual way. File uploads: type, size, name and storage location.
- **Resources** (API4, API6): pagination limits, request size, batch size, expensive queries, rate limiting for login, signup, password reset, OTP and other abuse-prone business flows.
- **Outbound** (API7, API10): URLs or hosts taken from input before a server-side request are allow-listed; data from third-party APIs is validated like user input.
- **Inventory** (API9): new routes are documented the way the repo documents them (OpenAPI, route tables); no debug, test or versioned-out endpoints left reachable.

## 2. Vectors

Scan the whole diff, endpoints or not, against the OWASP Top 10 (2025):

- **A01 Broken Access Control:** the authorization checks above, plus path traversal in file access, CORS that reflects any origin or allows credentials broadly, and SSRF in any server-side fetch.
- **A02 Security Misconfiguration:** debug modes, verbose errors or stack traces returned to clients, default credentials, permissive security headers, cookies without `Secure`, `HttpOnly`, `SameSite`, XML parsers with external entities enabled.
- **A03 Software Supply Chain Failures:** new or bumped dependencies. Run the ecosystem's audit for the touched manifests (`npm audit`, `pnpm audit`, `pip-audit`, `dotnet list package --vulnerable --include-transitive`, `govulncheck ./...`, `cargo audit`, `bundle audit`) when available, and report known vulnerabilities with their advisory ID. Also: unpinned versions, new registries or install scripts, CI changes that fetch and run remote code.
- **A04 Cryptographic Failures:** secrets, keys or tokens committed in code or config; weak or homemade crypto (MD5/SHA1 for passwords, ECB, static IVs, `Math.random` for tokens); plaintext transport or storage of sensitive data; disabled certificate validation.
- **A05 Injection:** input reaching SQL, NoSQL, LDAP, OS commands, shell, template engines, `eval`, regexes (ReDoS), or headers without parameterization or encoding. For the frontend: XSS through `innerHTML`, `dangerouslySetInnerHTML`, `v-html`, unescaped templates, `javascript:` URLs.
- **A06 Insecure Design:** business-flow abuse the code invites (negative quantities, skipped steps in a multi-step flow, race conditions on balances or limits, trusting client-side state or prices).
- **A07 Authentication Failures:** password handling, session fixation, missing re-authentication for sensitive changes, user enumeration through different errors or timings, MFA bypass.
- **A08 Software or Data Integrity Failures:** deserialization of untrusted data into types (`BinaryFormatter`, `pickle`, Java serialization, `TypeNameHandling`), unsigned webhooks or updates, trusting data from a client or queue as if it were internal.
- **A09 Security Logging and Alerting Failures:** security events (login failures, access denied, privilege changes) not logged the repo's way; passwords, tokens or PII written to logs.
- **A10 Mishandling of Exceptional Conditions:** failing open (an exception in an authorization check that lets the request through), swallowed errors that skip a security control, internal details leaked in error responses, missing limits that let one bad input exhaust memory or threads.

## 3. MCP

Run this pass when the diff adds or changes an MCP server, its tools, resources or prompts, an MCP client or host integration, or MCP configuration (`.mcp.json`, server entries in settings, agent configs). The model is the interpreter here: anything that reaches its context is input, and every tool is an endpoint the model can be talked into calling. Check against the OWASP MCP Top 10 (2025, https://owasp.org/projects/mcp-top-10):

- **MCP01 Token Mismanagement & Secret Exposure:** credentials hard-coded in server code or config, long-lived or broadly scoped tokens, secrets placed in tool results, prompts or model context, tokens written to logs or debug traces.
- **MCP02 Privilege Escalation via Scope Creep:** the server's credentials and the tools it exposes reach further than the goal needs (write where read would do, whole filesystem or database instead of a path or schema, admin API keys); no expiry on granted scopes.
- **MCP03 Tool Poisoning:** tool names, descriptions and schemas that steer the model (hidden instructions, misleading descriptions, a tool shadowing another's name); tool definitions fetched or changed at runtime so a trusted tool can turn malicious later (rug pull); tool output fed back to the model unmarked as untrusted.
- **MCP04 Software Supply Chain Attacks & Dependency Tampering:** MCP servers, connectors and SDKs pulled unpinned (`npx -y pkg`, `uvx pkg`, `latest` images) or from unknown publishers; no provenance or signature check. Run the A03 audit on them too.
- **MCP05 Command Injection & Execution:** tool arguments, prompts or retrieved content reaching shell commands, `eval`, SQL, file paths or URLs without the validation in A05; tools that run arbitrary code or commands by design without a sandbox or an allow-list.
- **MCP06 Prompt Injection via Contextual Payloads:** untrusted content (web pages, files, emails, tickets, OCR text, other tools' output) entering the model's context where it can trigger tool calls with side effects; no human confirmation for destructive or outward-facing actions.
- **MCP07 Insufficient Authentication & Authorization:** remote servers (HTTP/SSE transport) reachable without authentication; OAuth flows that skip audience or redirect validation or pass the client's token through to downstream APIs; tools that act with the server's identity instead of checking what the calling user may do (the API1/API5 checks, per tool).
- **MCP08 Lack of Audit and Telemetry:** tool calls, their arguments, the acting user and the outcome not logged the repo's way; no trail for actions with side effects.
- **MCP09 Shadow MCP Servers:** servers added outside the repo's approved list or deployment process, with default credentials, debug transports, permissive CORS or listening on `0.0.0.0`.
- **MCP10 Context Injection & Over-Sharing:** context, memory or session state shared across users, tenants or tasks; resources and tool results returning more data than the request needs; sensitive data persisted in conversation state.

## Output

The endpoint table:

```markdown
| Endpoint | AuthN | AuthZ | Input | Exposure |
|---|---|---|---|---|
| POST /orders/{id}/cancel | ok | missing ownership check (API1) | ok | ok |
```

Then one line per finding: `C<n> | file:line | <OWASP ID> <finding, with the input → harm path> | blocker / should fix / nit | concrete fix`.

Severity: a vector you can trace end to end (injection, missing authorization, exposed secret, known exploitable dependency, untrusted context that can trigger a side-effecting tool) is a **blocker**. A missing defense in depth with no open path today (no rate limit, a missing header, a log gap) is **should fix**. Hardening suggestions are **nits**. A suspected vector you could not trace either way is reported as should fix, with what you could not confirm.
