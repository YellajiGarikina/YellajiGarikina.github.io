
---
layout: post
title: "MCP Server Security: Tool Poisoning, Prompt Injection, and Why the Model Can't Be Your Access Control"
date: 2026-11-07
tags: [mcp, ai-security, llm-security, prompt-injection, tool-poisoning, oauth, appsec]
---

Connecting an AI assistant to repositories, databases, email systems, and other services through the Model Context Protocol (MCP) changes the security threat model.

The assistant no longer just generates text. It can retrieve information, invoke external tools, and perform operations on connected systems. Its decisions may also be influenced by content originating outside the trusted application.

This creates a security challenge familiar to application security engineers: how do you protect privileged operations when untrusted input can influence the component requesting them?

This article examines MCP's security architecture, published research into tool poisoning and tool selection attacks, OAuth authorization risks, and defensive controls recommended by established security guidance.

**What this article covers:**

- MCP architecture and its security boundaries.
- Prompt injection, tool poisoning, and tool shadowing.
- Excessive permissions and credential exposure.
- Authentication, authorization, and OAuth token handling.
- Confused deputy and BOLA vulnerabilities in agentic systems.
- Security controls recommended by MCP specifications and OWASP.

**Prerequisites:** Basic familiarity with APIs, OAuth, application security, and LLM tool calling.

## MCP is not inherently vulnerable

The Model Context Protocol is an open protocol that standardizes communication between AI applications and external tools and data sources.

It defines how an AI application discovers and interacts with capabilities exposed by MCP servers.

The protocol itself should not be confused with security weaknesses that can exist in its implementations.

The actual risks depend on several factors:

- Which MCP servers the application trusts.
- What information those servers expose.
- Which operations their tools can perform.
- How authentication and authorization are implemented.
- Whether retrieved content can influence privileged operations.
- Whether execution permissions are restricted independently of the model.

The important security question is not whether MCP is secure in isolation. It is what happens when an AI model processes untrusted information while having access to privileged tools.

## Understanding MCP architecture

MCP has three principal components:

| Component | Responsibility |
|---|---|
| MCP Host | AI application managing model interactions and connected clients |
| MCP Client | Maintains communication with an MCP server |
| MCP Server | Exposes tools, resources, and prompts |

The protocol separates its data layer from its transport layer.

The data layer defines messages and capabilities, including tools, resources, prompts, and notifications.

The transport layer defines how those messages are exchanged.

Common transports include:

- **STDIO:** Typically used for locally launched MCP servers.
- **Streamable HTTP:** Used for communication with remote MCP servers.

### Important protocol changes in July 2026

The MCP specification revision dated July 28, 2026 introduced significant architectural changes.

**1. Client identity metadata**

Client identity metadata is self-reported and should be used for identification, logging, and debugging—not as proof of an authenticated client or as a basis for authorization decisions.

**2. Header-based routing**

Streamable HTTP requests now include `Mcp-Method` and `Mcp-Name` headers.

These allow infrastructure components such as gateways and rate limiters to identify requests without parsing their JSON-RPC bodies.

Security gateways must still ensure that routing and authorization decisions are consistent with the actual request.

**3. Multi-round-trip requests**

MRTR replaces the earlier server-initiated request pattern for operations such as elicitation and sampling.

A server can request additional information, and the client responds by retrying the original request with the required input.

**4. Cacheable list responses**

The revised specification introduces `ttlMs` and `cacheScope` for applicable list and resource responses.

These fields provide information about response freshness and caching scope.

**5. Authorization improvements**

The specification strengthens OAuth-related requirements, including authorization server issuer validation and binding client credentials to the authorization server that issued them.

It also formally deprecates Dynamic Client Registration in favor of Client ID Metadata Documents, while retaining backward compatibility during the transition.

**6. Feature deprecations**

Roots, Sampling, and Logging are deprecated. The legacy HTTP+SSE transport is also deprecated.

Deprecated functionality remains available during its deprecation period; it should not be confused with functionality that has already been removed.

**Security implication:** These protocol changes improve architecture and interoperability, but they do not eliminate prompt injection, excessive permissions, or authorization weaknesses.

Sources:

- [MCP July 2026 Specification Release](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [Official MCP Specification Changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)

## Understanding MCP security boundaries

A secure MCP deployment involves several trust boundaries:

1. User and AI host.
2. AI host and language model.
3. MCP host and MCP client.
4. MCP client and MCP server.
5. MCP server and external services.
6. Trusted and untrusted MCP servers.
7. User identity and service identity.

The distinction between user identity and service identity is particularly important.

An MCP server may use its own service credentials to access a downstream system. That does not automatically mean every user interacting with the server should receive the same access.

Authorization must account for the authenticated principal, the requested operation, and the resource being accessed.

A trusted AI host also does not make every connected MCP server trustworthy.

## Three major attack categories

Prompt injection, tool poisoning, and tool shadowing are related security problems, but they describe different attack patterns.

| Category | Attack surface | Security concern |
|---|---|---|
| Prompt injection | User input and retrieved content | Malicious instructions influence model behavior |
| Tool poisoning | Tool descriptions, schemas, and outputs | Malicious tool information influences model decisions |
| Tool shadowing | Competing tool definitions | An adversarial tool influences or intercepts tool selection |

These categories can overlap. A single attack may use malicious tool metadata to influence a model and then exploit excessive permissions to perform an unauthorized operation.

## Risk 1: Prompt injection

Prompt injection occurs when malicious instructions embedded in input influence an LLM to behave outside its intended task.

There are two commonly recognized forms.

**Direct prompt injection**

The attacker supplies malicious instructions directly through user input.

**Indirect prompt injection**

The attacker places instructions in content that the AI application retrieves or processes.

Examples include:

- GitHub issues and pull requests.
- Repository documentation.
- Emails and attachments.
- Web pages.
- External documents.
- API responses.

An important characteristic of indirect prompt injection is that the attacker does not necessarily interact with the AI application directly.

The model may encounter the malicious instructions while performing an otherwise legitimate task.

MCP creates additional opportunities for this problem because servers can supply external information to an AI application.

When retrieved content influences subsequent tool calls, the impact depends on the permissions available to the agent.

OWASP identifies prompt injection as a major LLM security risk and recognizes both direct and indirect injection patterns.

**Security considerations:**

- Treat retrieved content as untrusted data.
- Do not allow external content to establish trusted instructions.
- Validate sensitive tool operations independently of model reasoning.
- Restrict the permissions available to the agent.
- Require appropriate authorization for consequential operations.

Source: [OWASP LLM01: Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)

## Risk 2: Tool poisoning

Tool poisoning targets information that an AI model uses to understand and select available tools.

MCP tools expose metadata, including names, descriptions, and parameter schemas.

An AI application can use this information to decide which tools are relevant to a user request.

A malicious or compromised MCP server can therefore introduce misleading instructions through its declared tool information.

The tool may appear legitimate to a user while its description contains additional instructions directed at the model.

### Published research: MCPTox

The research paper *MCPTox: A Benchmark for Tool Poisoning on Real-World MCP Servers* was published in the AAAI 2026 proceedings.

The researchers constructed a benchmark using:

- 45 real-world MCP servers.
- 353 authentic tools.
- 1,348 malicious test cases.
- 20 prominent LLM agent configurations.

The study evaluated how agents responded to malicious instructions embedded in tool metadata.

Its findings demonstrated that tool poisoning can influence agents into misusing otherwise legitimate tools.

The researchers also reported that more capable models could be more susceptible in their evaluation because the attacks exploit instruction-following behavior.

These results apply to the evaluated configurations and should not be interpreted as universal attack success rates across every MCP implementation.

**Security considerations:**

- Review complete tool definitions and parameter schemas.
- Verify server provenance.
- Monitor changes to previously approved tool definitions.
- Treat tool metadata as untrusted input.
- Restrict the permissions of tools independently of their descriptions.

Source: [MCPTox — AAAI 2026](https://doi.org/10.1609/aaai.v40i42.40895)

## Risk 3: Tool shadowing and tool selection hijacking

Tool shadowing is a security issue in which an adversarial tool competes with a legitimate tool and influences the model's selection.

The attacker does not necessarily need to create an exact copy of the legitimate tool.

Instead, misleading descriptions and metadata can influence the model to select the attacker's tool.

This matters because the model's tool selection is influenced by the information it receives about available capabilities.

### Published research: Confused Deputy Attack Against MCP

A 2026 paper published in *ACM Transactions on Software Engineering and Methodology* examined this issue.

The researchers introduced an automated evaluation framework called Puppet.

Their evaluation covered:

- 14 models.
- 6 model providers.
- 2 MCP hosts.

The paper reported tool-selection hijacking rates of up to 90.89% and end-to-end malicious payload execution rates of up to 86.46% in its evaluated configurations.

The researchers also reported that representative security scanners evaluated in their study failed to detect the metadata-level manipulation attacks.

These figures describe the paper's experimental environment. They should not be interpreted as the probability of successful exploitation in every MCP deployment.

**Security considerations:**

- Identify the server associated with each tool.
- Avoid trusting tool descriptions as proof of legitimacy.
- Review tool selection and invocation policies.
- Restrict sensitive operations to explicitly authorized capabilities.
- Require meaningful approval for high-impact operations.

Source: [Confused Deputy Attack Against Model Context Protocol — ACM TOSEM, 2026](https://doi.org/10.1145/3830467)

## Risk 4: Excessive permissions

Excessive permissions increase the impact of successful prompt injection and tool poisoning.

Consider an AI assistant designed to summarize repository documentation.

It may only require read access to the repository.

However, if the connected MCP server also exposes repository deletion, file modification, and administrative operations, the assistant has considerably more authority than its task requires.

This creates an unnecessarily large attack surface.

The problem is not limited to modification permissions.

Read access can also expose confidential information, including source code, internal documents, and credentials.

If an agent can access sensitive information and also has an outbound communication capability, unauthorized disclosure becomes possible.

**Security considerations:**

- Apply least privilege.
- Separate read and write capabilities.
- Use restricted service identities.
- Avoid granting administrative permissions for ordinary tasks.
- Apply additional authorization to destructive operations.

Source: [OWASP MCP Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/MCP_Security_Cheat_Sheet.html)

## Risk 5: Credential and data exposure

MCP integrations can operate close to sensitive information.

Examples include:

- GitHub access tokens.
- Cloud credentials.
- Database connection strings.
- API keys.
- OAuth access and refresh tokens.
- SSH keys.
- Confidential business documents.

Credential exposure can occur through insecure storage, excessive permissions, inappropriate logging, or compromised execution environments.

Prompt injection can introduce another risk when an agent has access to sensitive information and tools capable of transmitting that information elsewhere.

However, MCP does not automatically expose every credential on a developer's machine.

The actual exposure depends on the permissions, implementation, execution environment, and available tools.

**Security considerations:**

- Keep secrets outside model-accessible context wherever possible.
- Use secure credential storage.
- Avoid exposing credentials through tool responses.
- Redact sensitive information from logs.
- Use short-lived and appropriately scoped credentials.
- Avoid sharing credentials across unrelated MCP servers.

Sources:

- [OWASP MCP Top 10](https://owasp.org/projects/mcp-top-10)
- [OWASP MCP Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/MCP_Security_Cheat_Sheet.html)

## Risk 6: Authentication, authorization, and OAuth

Authentication and authorization are separate security controls.

Authentication establishes who is making a request.

Authorization determines whether that identity is permitted to perform a particular operation on a particular resource.

MCP authorization is optional overall. For implementations that use it, the authorization specification defines an OAuth-based authorization framework for HTTP transports.

It is important not to assume that local STDIO servers use the same authorization flow as remote HTTP servers.

Important security controls include:

| Control | Security purpose |
|---|---|
| OAuth 2.1 security practices | Protect authorization flows |
| PKCE | Protect authorization code exchange |
| HTTPS | Protect communication |
| Resource indicators (RFC 8707) | Bind tokens to intended resources |
| Audience validation | Reject tokens issued for another service |
| Authorization-response issuer validation (RFC 9207) | Mitigate authorization server mix-up attacks |
| Short-lived access tokens | Reduce the impact of token exposure |
| Refresh token rotation | Limit the impact of refresh token compromise |

### Token passthrough

Token passthrough is an important MCP authorization anti-pattern.

Consider a client that provides an access token intended for Service A.

An MCP server accepts the token without validating whether it was issued for that server and forwards it to Service B.

This breaks the intended OAuth resource boundary.

The downstream service may receive a credential that was never intended for it.

The MCP authorization specification explicitly prohibits this pattern.

The correct approach is to validate the incoming token for the MCP server and obtain an appropriately scoped credential for the downstream service.

Source: [Official MCP Authorization Security Considerations](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/basic/authorization/security-considerations.mdx)

## The AppSec perspective: The model is an untrusted client

Application security engineers are already familiar with many of the weaknesses that appear in MCP deployments.

The difference is that the caller is now an AI agent whose behavior can be influenced by external content.

The same security principles used for web applications and APIs still apply.

| MCP security failure | Familiar AppSec vulnerability | Security concern |
|---|---|---|
| Tool trusts a model-supplied customer ID | BOLA / IDOR | Unauthorized access to another user's resource |
| Agent has unnecessary administrative tools | Excessive privilege | Unnecessary attack surface |
| Server forwards an incorrectly scoped token | OAuth audience confusion | Credential misuse across services |
| Server uses excessive OAuth scopes | Scope creep | More permissions than necessary |
| Tool trusts model-generated parameters | Missing input validation | Untrusted input reaches privileged operations |

### Broken Object Level Authorization (BOLA)

BOLA is particularly relevant to MCP integrations.

Consider an application that exposes a tool for retrieving customer records.

The tool receives a customer identifier and queries a downstream database using a service account.

If the server checks only whether its service account can access the record, rather than whether the authenticated user is authorized to access that specific customer record, the application may expose another customer's information.

This is fundamentally the same authorization weakness found in traditional REST APIs.

The presence of an AI model does not change the underlying requirement.

Authorization must be enforced using a validated identity and resource-level access checks on every relevant operation.

### Confused deputy

A confused deputy occurs when a privileged component uses its authority on behalf of a caller without correctly enforcing the caller's permissions.

In MCP deployments, an agent may invoke a server that holds broader privileges than the user.

If the server trusts the requested operation without enforcing the user's authorization, it can unintentionally exercise its own privileges on behalf of an unauthorized caller.

This is why authentication alone is insufficient.

The server must enforce authorization independently of the model's decision.

Source: [Confused Deputy Attack Against Model Context Protocol](https://doi.org/10.1145/3830467)

## Risk 7: Local MCP server security

Local MCP servers commonly run as processes on a developer's machine.

They may interact with local files, environment variables, installed applications, and external services.

This creates a process and operating-system security boundary.

Potential weaknesses include:

- Unverified third-party packages.
- Malicious dependencies.
- Untrusted startup commands.
- Excessive filesystem permissions.
- Inherited environment variables.
- Unnecessary network access.

One important distinction:

**STDIO is a transport, not a sandbox.**

Using STDIO avoids exposing an HTTP listening endpoint, but it does not automatically restrict what the local process can access.

The operating system and execution environment determine its actual permissions.

**Security considerations:**

- Verify package sources.
- Review dependencies.
- Restrict filesystem access.
- Avoid unnecessary environment-variable exposure.
- Use operating-system or container isolation.
- Grant only the permissions required by the server.

Source: [OWASP MCP Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/MCP_Security_Cheat_Sheet.html)

## Security controls for MCP deployments

The security risks discussed above require multiple defensive layers.

No single prompt instruction, scanner, or approval mechanism can replace proper authorization and execution controls.

The following controls are aligned with the official MCP security guidance and OWASP recommendations.

### 1. Tool integrity

Tool definitions should be reviewed before an MCP server is trusted.

This includes:

- Tool names.
- Descriptions.
- Parameter schemas.
- Server provenance.
- Changes to previously approved definitions.

A previously approved tool should not automatically remain trusted after its definition or implementation changes.

### 2. Untrusted content handling

Treat external content and tool responses as untrusted data.

Retrieved content should not be allowed to establish authorization or override trusted application instructions.

Prompt-level restrictions can provide additional defense, but they cannot replace independent security enforcement.

### 3. Least privilege

Grant each MCP server only the permissions necessary for its intended function.

Separate read and write operations where practical.

Restrict local processes and downstream service identities.

### 4. Server-side authorization

Every sensitive operation should be authorized independently of the model.

Authorization should consider:

- Authenticated principal.
- Requested operation.
- Target resource.
- Applicable permissions.
- Delegated authority.

### 5. Secure OAuth implementation

Validate token issuer and audience.

Avoid token passthrough.

Use appropriately scoped downstream credentials and follow OAuth security requirements.

### 6. Credential protection

Use secure credential storage and avoid placing secrets in model-accessible content or application logs.

Apply appropriate credential rotation and scope restrictions.

### 7. Meaningful human approval

Sensitive and destructive operations should require appropriate authorization and confirmation.

Approval interfaces should clearly identify the server, tool, and operation being requested.

A generic approval prompt that hides the actual operation provides limited security value.

### 8. Monitoring and auditing

Security monitoring should cover:

- New MCP server connections.
- Changes to tool definitions.
- Unusual tool invocation patterns.
- Sensitive file access.
- Privileged API operations.
- Authorization failures.
- Unexpected outbound data transfers.
- Credential and token misuse.

Audit records should provide sufficient information to reconstruct important operations while protecting sensitive information in logs.

Sources:

- [MCP Security Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)
- [OWASP MCP Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/MCP_Security_Cheat_Sheet.html)

## Understanding the OWASP MCP Top 10

The OWASP MCP Top 10 provides a dedicated security classification for risks associated with MCP implementations.

It covers areas including:

- Token mismanagement and secret exposure.
- Privilege escalation through scope creep.
- Tool poisoning.
- Software supply-chain attacks.
- Command injection and execution.
- Prompt injection through contextual payloads.
- Insufficient authentication and authorization.
- Lack of audit and telemetry.
- Shadow MCP servers.
- Context injection and over-sharing.

The project is maintained separately from the OWASP Web Application Top 10.

**Important:** The OWASP project currently identifies itself as being in its beta release and pilot-testing phase. Its categories should therefore be treated as an evolving security resource rather than a finalized standard.

Source: [OWASP MCP Top 10](https://owasp.org/projects/mcp-top-10)

## Misconceptions about MCP security

| Misconception | Reality |
|---|---|
| MCP itself is inherently vulnerable | Security depends on implementations and deployment |
| A trusted server is always safe | Approved servers can change or become compromised |
| System prompts prevent unauthorized operations | Authorization must be enforced outside model reasoning |
| OAuth authentication guarantees resource authorization | Resource-level authorization is still necessary |
| Read-only tools are always safe | Read access can expose confidential information |
| STDIO provides process isolation | STDIO is a communication transport, not a sandbox |
| Scanning tool descriptions catches every attack | Static inspection cannot establish implementation safety |

## Conclusion

MCP makes it easier for AI applications to interact with external systems, but it also introduces security challenges around trust, permissions, and authorization.

Prompt injection can influence model behavior. Tool poisoning can manipulate the information used to select capabilities. Tool shadowing can influence which tool an agent invokes.

These risks become more serious when an agent has access to sensitive data or privileged operations.

For application security engineers, many of the underlying problems are familiar: BOLA, excessive permissions, insufficient authorization, credential exposure, and confused deputy vulnerabilities.

The important architectural principle is that **the model must never be the final authority for security decisions**.

Enforce permissions at the server and execution boundaries. Validate credentials for their intended audience. Restrict tool capabilities and monitor sensitive operations.

An AI model can help decide what action is useful, but the application must independently determine whether that action is permitted.

## References

### MCP specification and security guidance

- [MCP July 28, 2026 Specification Release](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [MCP Specification Changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- [MCP Authorization Specification](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)
- [MCP Authorization Security Considerations](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/basic/authorization/security-considerations.mdx)
- [MCP Security Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)
- [MCP Architecture Specification](https://modelcontextprotocol.io/specification/2026-07-28/architecture)

### OWASP

- [OWASP MCP Top 10](https://owasp.org/projects/mcp-top-10)
- [OWASP MCP Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/MCP_Security_Cheat_Sheet.html)
- [OWASP MCP03: Tool Poisoning](https://owasp.github.io/www-project-mcp-top-10/2025/MCP03-2025%E2%80%93Tool-Poisoning)
- [OWASP LLM01: Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)

### Published research

- [MCPTox: A Benchmark for Tool Poisoning on Real-World MCP Servers — AAAI 2026](https://doi.org/10.1609/aaai.v40i42.40895)
- [Confused Deputy Attack Against Model Context Protocol — ACM TOSEM 2026](https://doi.org/10.1145/3830467)
- [Invariant Labs: MCP Tool Poisoning Attacks](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks)

### OAuth and standards

- [RFC 8707: Resource Indicators for OAuth 2.0](https://www.rfc-editor.org/rfc/rfc8707)
- [RFC 9207: OAuth 2.0 Authorization Server Issuer Identification](https://www.rfc-editor.org/rfc/rfc9207)
- [OAuth 2.1 Internet-Draft](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/)
