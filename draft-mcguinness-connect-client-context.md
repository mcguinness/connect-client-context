---
title: "OpenID Connect Client Context Parameter"
abbrev: "OIDC Client Context"
docName: "draft-mcguinness-connect-client-context-latest"
category: std
ipr: trust200902
# area: Security
# workgroup: "OpenID Connect"
keyword:
  - OpenID Connect
  - OAuth 2.0
  - Client Context
venue:
#  group: "OpenID Connect"
#  type: Working Group
#  mail: "openid-specs-ab@lists.openid.net"
  github: "mcguinness/connect-client-context"
  latest: "https://mcguinness.github.io/connect-client-context/draft-mcguinness-connect-client-context.html"
author:
  -
    ins: K. McGuinness
    name: Karl McGuinness
    org: Independent
date: 2026-03-10

normative:
  RFC2119:
  RFC8174:
  RFC3339:
  RFC6749:
  RFC7519:
  RFC7591:
  RFC8126:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC9126:
  RFC9396:

ref:
  OIDC.TPIL:
    title: "OpenID Connect Third Party Initiated Login 1.0"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
      - ins: E. Jay
    date: 2014-11
    target: https://openid.net/specs/openid-connect-third-party-initiated-login-1_0.html
  OIDC.Core:
    title: "OpenID Connect Core 1.0"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
      - ins: B. de Medeiros
      - ins: C. Mortimore
    date: 2014-11
    target: https://openid.net/specs/openid-connect-core-1_0.html
  OIDC.Discovery:
    title: "OpenID Connect Discovery 1.0"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
      - ins: E. Jay
    date: 2014-11
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  OIDC.Registration:
    title: "OpenID Connect Dynamic Client Registration 1.0"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
    date: 2014-11
    target: https://openid.net/specs/openid-connect-registration-1_0.html
---

--- abstract

This specification defines the `client_context` authorization request parameter for OpenID Connect. The parameter enables a Client to convey structured contextual information to the OpenID Provider (OP) so that the OP can apply context-specific authentication policy and claim release when issuing an ID Token.

The mechanism is designed for deployments where a single `client_id` represents multiple logical applications, tenants, or operational purposes, such as AI agents acting on behalf of users or just-in-time administrative workflows requiring privilege elevation.

The OP returns the applied context in the ID Token, allowing the Client to verify that authentication occurred under the intended context and to bind subsequent operations accordingly.

--- middle

# Introduction

OpenID Connect Core {{!OIDC.Core}} defines a protocol for authenticating End-Users and conveying identity claims to Clients via ID Tokens issued by an OpenID Provider (OP). While this model works well for simple deployments with a one-to-one relationship between a registered client and a logical application, modern deployments increasingly require richer context.

## The Multi-Context Problem

A growing class of deployments uses a single registered `client_id` to represent multiple distinct logical contexts. Consider:

**Multi-application SaaS platforms.** A platform vendor registers one client but hosts dozens of distinct applications (calendar, email, CRM). Each application may require different authentication policies; the email client might allow password authentication, while the financial reporting module requires MFA. The OP has no way to distinguish which application is requesting authentication.

**Multi-tenant enterprise services.** A B2B SaaS provider serves many enterprise customers under one client registration. Each tenant may have its own password policy, MFA requirements, or allowed identity providers (e.g., only employees from `example.com` may authenticate to the `example.com` tenant). Without tenant context, the OP cannot enforce per-tenant policy.

**AI agents acting on behalf of users.** An AI assistant platform registers a single client but spawns many concurrent agent sessions, each operating under a specific task or purpose (e.g., "draft a response to these emails" or "provision a new cloud environment"). The OP needs to understand the agent's operational context to determine whether step-up authentication is appropriate and which claims to release.

**Just-in-time administrative workflows.** Any client application, not just a dedicated privileged access management tool, may need to gate a sensitive operation behind a fresh, purpose-specific authentication. For example, a SaaS admin panel, a developer portal, or an internal tooling app might require a user to re-authenticate specifically to register a new OAuth client, approve a change request, or review audit logs, rather than relying on a long-lived general-purpose session. The OP needs purpose context to enforce step-up authentication and restrict the resulting session to that specific operation.

## Gap in Existing Parameters

Existing OpenID Connect and OAuth 2.0 parameters each serve a distinct purpose but none address client-local identity context:

* `scope` indicates which permissions or resources are requested, not why or under which application context.
* `claims` requests specific identity attributes, not contextual information about the request itself.
* `acr_values` communicates the desired authentication assurance level, not the business or purpose context that determines what assurance is appropriate.
* `resource` ({{!RFC8707}}) identifies the protected resource audience for access tokens, not the logical application or tenant context for authentication.
* `authorization_details` ({{!RFC9396}}) conveys structured authorization semantics for access tokens, not contextual metadata for ID Token issuance.

None of these parameters provide a structured, extensible mechanism for a Client to communicate the logical context under which authentication is requested.

## This Specification

This specification introduces the `client_context` request parameter to address this gap. The parameter carries a structured array of typed context objects that the OP can use as input to authentication policy, claim release decisions, and audit logging. The OP returns the applied context in the ID Token, enabling the Client to cryptographically verify that authentication was performed under the intended context.

## Design Principles

This specification is guided by the following principles:

* `client_context` influences authentication behavior and ID Token content. It does not define access token permissions.
* `purpose.id` is the primary policy key for purpose-scoped authentication.
* `params` contains purpose-specific, validated inputs for authentication policy, consent display, and audit. It is not a general-purpose extension bag.

# Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they appear in all capitals, as shown here.

## Terminology

This specification uses the following terms from OpenID Connect Core {{!OIDC.Core}}:

**OpenID Provider (OP)**
: OAuth 2.0 Authorization Server that implements OpenID Connect and is capable of authenticating the End-User and providing Claims about the End-User to a Relying Party.

**Client**
: OAuth 2.0 Client application using OpenID Connect, also referred to as a Relying Party (RP).

**ID Token**
: JSON Web Token (JWT) that contains Claims about the Authentication event, issued by the OP.

The following terms are defined by this specification:

**Client Context**
: Structured information about the logical context under which a Client is requesting authentication, conveyed via the `client_context` parameter.

**Context Type**
: A registered identifier (e.g., `app`, `tenant`, `purpose`) denoting the semantic category of a context object.

**Context Object**
: A JSON object within the `client_context` array, containing at minimum a `type` field and type-specific members.

**Purpose**
: An operational context representing a specific task, workflow, or purpose for which authentication is being requested, as opposed to a general-purpose login.

# Protocol Overview

The following diagram illustrates the flow of a `client_context`-bearing authorization request through an OpenID Connect interaction.

~~~
 Client                           OpenID Provider
   |                                     |
   |-- Authorization Request ----------->|
   |   client_context: [                 |
   |     { type: "app",                  |
   |       id: "admin_console" },        |
   |     { type: "tenant",               |
   |       id: "example.com" },          |
   |     { type: "purpose",              |
   |       id: "add_oauth_client" }        |
   |   ]                                 |
   |   scope, acr_values, claims, ...    |
   |                                     |
   |                              [Context Validation]
   |                              [Policy Evaluation]
   |                              [Authentication]
   |                              [Context Applied]
   |                                     |
   |<-- Authorization Code --------------|
   |                                     |
   |-- Token Request ------------------->|
   |                                     |
   |<-- ID Token ------------------------|
   |   acr: "http://...mfa"              |
   |   client_context: [                 |
   |     { type: "app",                  |
   |       id: "admin_console" },        |
   |     { type: "tenant",               |
   |       id: "example.com" },          |
   |     { type: "purpose",              |
   |       id: "add_oauth_client" }        |
   |   ]                                 |
   |   ...standard claims...             |
   |                                     |
   |  [Client verifies context]          |
   |  [Enables purpose-bound actions]    |
~~~

The protocol proceeds as follows:

1. The Client constructs an authorization request that includes `client_context` alongside standard OpenID Connect parameters.

2. The OP receives the request and validates the `client_context` parameter structure.

3. The OP evaluates the supplied context as input to its authentication policy, for example, determining that a `purpose` context with `id` `add_oauth_client` requires step-up MFA authentication, or that a `tenant` context restricts authentication to a specific identity provider.

4. The OP authenticates the End-User according to the derived policy.

5. The OP issues an ID Token that includes a `client_context` claim reflecting the context that was actually applied. This MAY differ from the requested context if the OP normalized values or applied local policy.

6. The Client validates the ID Token, verifies the returned `client_context` claim, and enables only the operations appropriate for the confirmed context.

# The `client_context` Authorization Request Parameter

## Definition

`client_context`
: OPTIONAL. A JSON array of Context Objects providing contextual information about the request. Each element MUST be a JSON object containing at minimum a `type` field. The semantics and additional members of each object are determined by the context type definition.

The following shows the parameter embedded in an authorization request:

~~~
GET /authorize?
  response_type=code
  &client_id=s6BhdRkqt3
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
  &scope=openid%20email
  &acr_values=urn%3Amace%3Aincommon%3Aiap%3Asilver
  &client_context=%5B%7B%22type%22%3A%22app%22%2C%22id%22%3A%22calendar%22%7D%5D
  &nonce=n-0S6_WzA2Mj HTTP/1.1
Host: server.example.com
~~~

The decoded `client_context` value is:

~~~json
[
  { "type": "app", "id": "calendar" }
]
~~~

## Context Object Schema

Each element of the `client_context` array MUST be a JSON object conforming to the following:

* The object MUST contain a `type` member whose value is a string identifying the context type. The value MUST be a registered context type name or a URI identifying a private-use extension type.
* Additional members are defined by the context type specification.
* Members not recognized by the recipient SHOULD be ignored (forward-compatibility).
* Unless otherwise specified by the context type, Clients SHOULD include at most one object per context type in a single request.

## Transmission

The `client_context` parameter MAY be transmitted in any of the following ways:

**Direct query parameter.** The JSON array is URL-encoded and included as a query parameter. This is simple but exposes context values in server logs and browser history. Not RECOMMENDED for sensitive purpose context.

**Request Object.** The parameter is included as a claim in a JWT Request Object per Section 6 of {{!OIDC.Core}}. This protects integrity when the Request Object is signed.

~~~json
{
  "iss": "s6BhdRkqt3",
  "aud": "https://op.example.com",
  "response_type": "code",
  "client_id": "s6BhdRkqt3",
  "scope": "openid email",
  "nonce": "n-0S6_WzA2Mj",
  "client_context": [
    { "type": "app", "id": "admin_console" },
    { "type": "tenant", "id": "example.com" },
    {
      "type": "purpose",
      "id": "add_oauth_client",
      "display": {
        "title": "Register OAuth Client",
        "description": "Register new OAuth client 'invoice-service' in the example.com tenant (ticket IT-4821).",
        "locale": "en"
      },
      "params": { "client_name": "invoice-service", "environment": "production" },
      "constraints": { "expires_at": "2026-03-10T18:00:00Z" }
    }
  ]
}
~~~

**Pushed Authorization Request (PAR).** The parameter is included in the body of a PAR request per {{!RFC9126}}. This is the RECOMMENDED approach for requests containing sensitive context (e.g., purpose context for privileged workflows). PAR ensures the context is submitted directly to the OP over a server-to-server TLS connection, preventing exposure in browser redirects.

~~~
POST /as/par HTTP/1.1
Host: op.example.com
Content-Type: application/x-www-form-urlencoded

response_type=code
&client_id=s6BhdRkqt3
&redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
&scope=openid%20email
&acr_values=urn%3Amace%3Aincommon%3Aiap%3Asilver
&client_context=%5B%7B%22type%22%3A%22app%22%2C%22id%22%3A%22admin_console%22%7D%2C
  %7B%22type%22%3A%22tenant%22%2C%22id%22%3A%22example.com%22%7D%2C
  %7B%22type%22%3A%22purpose%22%2C%22id%22%3A%22add_oauth_client%22%7D%5D
~~~

# Registered Context Types

## Application Context (`app`)

The `app` context type identifies the logical application within a multi-application client deployment.

**Required members:**

`type`
: MUST be `"app"`.

`id`
: REQUIRED. String identifier for the logical application. The value space is defined by the client and SHOULD be registered in the client's metadata.

**Optional members:**

`version`
: OPTIONAL. String indicating the version of the application.

**Example:**

~~~json
{
  "type": "app",
  "id": "calendar",
  "version": "3.2"
}
~~~

**OP policy guidance:** The OP MAY use the `id` value to select an application-specific login theme, restrict claim release to claims relevant to that application, or apply application-specific session policies. The OP SHOULD validate that the requested `id` is registered as a permitted application for this `client_id`.

## Tenant Context (`tenant`)

The `tenant` context type identifies the organizational tenant on whose behalf authentication is being requested. This is used in multi-tenant deployments where a single client serves multiple organizations.

**Required members:**

`type`
: MUST be `"tenant"`.

`id`
: REQUIRED. String identifying the tenant. This is typically a domain name (e.g., `"example.com"`), an internal tenant UUID, or another stable tenant identifier registered with the OP.

**Optional members:**

`domain`
: OPTIONAL. The domain associated with the tenant, when the `id` is not itself a domain. Useful for routing home-realm discovery.

**Example:**

~~~json
{
  "type": "tenant",
  "id": "example.com"
}
~~~

**OP policy guidance:** The OP MAY use tenant context to enforce that only users with verified membership in the tenant may authenticate, redirect to a tenant-specific identity provider for federation, apply tenant-specific password and MFA policies, and include tenant-specific claims in the ID Token (e.g., a tenant identifier claim for downstream services).

## Purpose Context (`purpose`)

The `purpose` context type identifies the specific operational task, workflow, or purpose for which authentication is being requested. Purpose context conveys *why* the authentication is occurring, enabling the OP to apply purpose-specific policy.

Purpose context is the primary mechanism for:

* **AI agent task scoping**: identifying the specific task an agent is performing on a user's behalf, so that authentication and session scope can be limited to that task.
* **Just-in-time privilege elevation**: requesting authentication for a specific administrative action rather than a general administrative session.
* **Automated workflow authorization**: identifying a machine-to-machine workflow requiring human authentication for a specific step.
* **Delegated user actions**: recording that a user is authenticating to authorize a specific action taken by another entity.

The purpose object is structured to serve two distinct audiences: a **policy engine** at the OP that evaluates authentication requirements, and a **consent UI** that presents the user with a clear description of what they are authorizing. These concerns are separated into distinct sub-objects to avoid ambiguity and enable each consumer to operate independently on its relevant fields.

### Top-Level Members

**Required members:**

`type`
: MUST be `"purpose"`.

`id`
: REQUIRED. String identifying the purpose type. This is the primary key against which the OP evaluates authentication policy, for example, looking up required ACR, session lifetime caps, or step-up requirements in its purpose catalog. The `id` value space is defined by the client and SHOULD be registered in the client's `client_context_values` metadata. Example values: `"add_oauth_client"`, `"approve_change_request"`, `"deploy_to_production"`, `"summarize_inbox"`.

**Optional members:**

`instance_id`
: OPTIONAL. String containing a client-generated identifier for this specific invocation of the purpose. This value enables later receipts, audit records, or downstream protocols to refer to the same authenticated purpose instance. When present, the OP SHOULD return the validated value in the resulting `client_context` claim. Example value: `"01JNP8Q8W5W4K7QG0R9B3R2M6F"`.

`display`
: OPTIONAL. Object providing human-readable labels for display in the OP's consent and authentication UI. See {{display-object}}.

`params`
: OPTIONAL. Object containing instance-level parameters specific to this execution of the purpose. The schema of `params` is defined per `id` value; different purpose types carry different parameters. See {{params-object}}.

`initiator`
: OPTIONAL. Object identifying the entity that triggered this purpose and is requesting the authentication on behalf of the End-User. See {{initiator-object}}.

`constraints`
: OPTIONAL. Object specifying limiting conditions on the resulting authentication session and tokens. See {{constraints-object}}.

### The `display` Object {#display-object}

The `display` sub-object carries information for the OP's consent and authentication UI. Its contents are treated as client-supplied labels that the OP SHOULD present to the End-User.

`title`
: OPTIONAL. Short human-readable title for the purpose, suitable for a consent screen heading. Example: `"Register OAuth Client"`.

`description`
: OPTIONAL. Human-readable explanation of the specific task instance, including any relevant context the user needs to make an informed decision. Example: `"You are authorizing registration of a new OAuth client 'invoice-service' in the example.com tenant (ticket IT-4821)."`. The description SHOULD be specific to the instance, not just the purpose type.

`locale`
: OPTIONAL. BCP 47 language tag indicating the language of `title` and `description`. Example: `"en"`, `"fr-CA"`.

### The `params` Object {#params-object}

The `params` sub-object carries instance-level parameters for this specific purpose execution. Its members are defined by the purpose type identified by `id` and SHOULD be documented in the client's purpose catalog.

Clients and OPs MUST validate `params` according to the schema and policy associated with the `purpose.id` value. Unknown parameter names, malformed values, or values outside the allowed policy for that `purpose.id` MUST cause the request to be rejected or the `purpose` object to be ignored according to local policy.

`params` serves three purposes:

* **Conditional policy evaluation.** The OP can write policy rules that branch on parameter values. For example: an `add_oauth_client` purpose where `params.environment` is `production` may require a higher assurance level than one targeting a `staging` environment.

* **Consent display.** The OP can use parameter values to construct or supplement its own consent language when `display.description` is absent or insufficient.

* **Structured audit.** Because `params` is a JSON object rather than a free-text string, audit systems can query and aggregate over it. For example: "all `add_oauth_client` registrations for `params.client_name` in the past 30 days."

Clients SHOULD include only policy-relevant instance attributes in `params`. `params` is not a general-purpose authorization descriptor and MUST NOT be used to express resource permissions or downstream access rights.

`params` MAY include policy-relevant target identifiers for an authenticated administrative action, such as a target subject identifier, client identifier, tenant identifier, or deployment environment. `params` MUST NOT be used to convey command execution state, workflow progress, completion status, or other mutable state that belongs to a downstream command or orchestration protocol.

Clients MUST NOT include sensitive personal data in `params` beyond what is necessary for policy evaluation and audit. Clients SHOULD NOT rely on `params` values as the sole basis for access control decisions at downstream resource servers; that is the role of access tokens.

### The `initiator` Object {#initiator-object}

The `initiator` sub-object identifies the entity that triggered the purpose and requested authentication on the End-User's behalf. This enables the OP to apply policy rules based on who or what initiated the authentication, not just what purpose is being performed.

`id`
: OPTIONAL. String containing a stable identifier for the initiating entity. For AI agents, this MAY be a session or agent instance identifier. For services, this MAY be a stable service identifier registered with the platform.

`type`
: REQUIRED if `initiator` is present. String indicating the category of the initiating entity. RECOMMENDED values:

  * `user` - a human user initiated this purpose (e.g., a manager initiating a verification flow for a team member).
  * `agent` - an AI agent or automated agent initiated this purpose.
  * `service` - an application, system, or automated service initiated this purpose.

Clients SHOULD omit `initiator` when the End-User directly starts the authentication within the Client itself. The `initiator` value is informational and is not itself authenticated by this parameter. Clients SHOULD authenticate the initiator through separate means (e.g., service-to-service OAuth credentials) before asserting its identity here. OPs MAY apply more stringent authentication requirements when `initiator.type` is `agent` or `service` to mitigate automated abuse.

### The `constraints` Object {#constraints-object}

The `constraints` sub-object groups conditions that bound the resulting authentication session and any issued tokens.

`expires_at`
: OPTIONAL. Date-time string ({{!RFC3339}}) after which the authenticated authorization to perform this purpose instance is no longer valid. The OP SHOULD enforce that any resulting session, token, or other local authorization state used to carry out the purpose does not outlive this value, including refresh tokens where applicable.

### OP Policy Guidance

The OP SHOULD use `id` as the primary key for policy evaluation, looking up authentication requirements (required ACR, session lifetime cap, step-up rules) from its purpose catalog.

The OP SHOULD present `display.title` and `display.description` to the End-User during authentication to provide a complete picture of what the user is authorizing.

The OP SHOULD evaluate `initiator.type` when `id` alone is insufficient to determine authentication requirements. For example, an OP MAY require re-authentication when `initiator.type` is `agent`, regardless of existing session state.

The OP MAY bind a validated `purpose` object to the authenticated subject and, when available, to a resulting session identifier such as OIDC `sid` or a SAML `SessionIndex`. This allows the returned context to serve as a verifiable record of which subject authenticated for which purpose instance and, where supported, which session carries that authorization.

### Examples

**Example: AI agent task**

~~~json
{
  "type": "purpose",
  "id": "summarize_inbox",
  "display": {
    "title": "Summarize Inbox",
    "description": "AI Assistant will read your unread emails from the past 7 days and generate a summary.",
    "locale": "en"
  },
  "initiator": {
    "type": "agent",
    "id": "agent-session-a1b2c3"
  },
  "constraints": {
    "expires_at": "2026-03-10T20:00:00Z"
  }
}
~~~

**Example: just-in-time administrative action**

~~~json
{
  "type": "purpose",
  "id": "add_oauth_client",
  "instance_id": "01JNP8Q8W5W4K7QG0R9B3R2M6F",
  "display": {
    "title": "Register OAuth Client",
    "description": "You are authorizing registration of a new OAuth client 'invoice-service' in the example.com tenant (ticket IT-4821).",
    "locale": "en"
  },
  "params": {
    "client_name": "invoice-service",
    "environment": "production"
  },
  "constraints": {
    "expires_at": "2026-03-10T18:00:00Z"
  }
}
~~~

# Client Processing Rules

## Constructing the Request

A Client generating an authorization request containing `client_context` MUST:

1. Encode the parameter value as a JSON array. An empty array is NOT RECOMMENDED; omit the parameter entirely if no context is applicable.
2. Ensure each element is a JSON object containing a `type` field with a non-empty string value.
3. Ensure each object's structure conforms to the schema defined for that context type.
4. Avoid including contradictory context values (e.g., two `tenant` objects with different `id` values).
5. Include a `nonce` parameter in the authorization request to bind the resulting ID Token to the request.

Clients SHOULD use PAR ({{!RFC9126}}) or a signed Request Object when `client_context` contains sensitive information such as purpose identifiers or administrative identifiers.

## Validating the Response

When receiving an ID Token in response to a request that included `client_context`, the Client MUST:

1. Validate the ID Token per Section 3.1.3.7 of {{!OIDC.Core}}, including signature verification, issuer, audience, and expiration checks.
2. Verify that the `nonce` claim in the ID Token matches the `nonce` sent in the request.
3. If the request included `client_context`, examine the `client_context` claim in the ID Token.
4. Compare the returned context to the requested context. The Client MUST detect and handle:
   - **Missing context:** A context type that was requested is absent from the returned value. The Client MUST apply local policy to determine whether this is acceptable. For security-critical context (e.g., purpose context for privileged operations), the Client MUST treat missing context as an error and reject the token.
   - **Modified context:** A context value was altered by the OP (e.g., due to normalization). The Client MUST evaluate whether the modification is acceptable.
   - **Additional context:** The OP included context objects not present in the request. The Client SHOULD ignore unexpected context types.
5. Reject or restrict privileged operations if required context values are absent or inconsistent.

## Example: Client Verification Logic

The following pseudocode illustrates how a Client might verify returned context for a purpose-bound administrative operation:

~~~
id_token = validate_and_parse_jwt(token_response.id_token)

# Standard validation
assert id_token.iss == expected_issuer
assert id_token.aud == client_id
assert id_token.nonce == request_nonce

# Context verification
returned_context = id_token.client_context  # array of objects

purpose = find(returned_context, type="purpose")
if purpose is None:
    raise SecurityError("Purpose context missing from ID Token")
if purpose.id != requested_purpose_id:
    raise SecurityError("Purpose context mismatch")

tenant = find(returned_context, type="tenant")
if tenant is None or tenant.id != expected_tenant:
    raise SecurityError("Tenant context invalid")

# All checks passed; enable purpose-scoped operations
enable_operations(purpose.id)
~~~

# Authorization Server Processing Rules

## Parsing and Validation

If the `client_context` parameter is present in an authorization request, the Authorization Server MUST:

1. Parse the parameter value as a JSON array. If parsing fails, return an `invalid_client_context` error (see {{errors}}).
2. Verify that each element of the array is a JSON object. If any element is not an object, return `invalid_client_context`.
3. Verify that each object contains a `type` field with a non-empty string value. If any object is missing `type`, return `invalid_client_context`.
4. For each context type in the request:
   a. If the type is not recognized and the OP does not support unknown types, return `unsupported_client_context_type`.
   b. If the type is recognized, validate the object against the registered schema for that type. If validation fails, return `invalid_client_context_value`.
5. If the client's registration metadata includes `client_context_types`, verify that all requested context types are in the registered list. If not, return `unsupported_client_context_type`.

## Policy Evaluation

The Authorization Server SHOULD use the `client_context` values as input to authentication policy evaluation. Examples of policy decisions driven by context:

* **Application context** MAY determine which login UI is presented, which identity providers are available, and which claims are released.
* **Tenant context** MAY restrict authentication to users belonging to a specific organization or domain, apply tenant-specific MFA requirements, or select a tenant-specific IdP.
* **Purpose context** MAY trigger step-up authentication, require explicit re-authentication regardless of existing session, reduce session and token lifetimes, or require additional approval.

The OP MAY normalize context values according to local policy (e.g., resolving an application name alias to a canonical identifier) before evaluating policy and returning the context in the ID Token.

## Issuing the ID Token

If authentication succeeds, the Authorization Server SHOULD include a `client_context` claim in the ID Token. The value MUST be a JSON array containing the context objects that were actually applied, which MAY differ from the requested context due to normalization or policy.

If the OP does not support `client_context` or chooses not to return it, it MUST NOT include the claim. Clients that required specific context must treat its absence as a verification failure.

The following is an example ID Token payload containing an applied context:

~~~json
{
  "iss": "https://op.example.com",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "nonce": "n-0S6_WzA2Mj",
  "iat": 1741564800,
  "exp": 1741568400,
  "acr": "urn:mace:incommon:iap:silver",
  "amr": ["mfa", "swk"],
  "email": "user@example.com",
  "client_context": [
    { "type": "app", "id": "admin_console" },
    { "type": "tenant", "id": "example.com" },
    {
      "type": "purpose",
      "id": "add_oauth_client",
      "display": {
        "title": "Register OAuth Client",
        "description": "Register new OAuth client 'invoice-service' in the example.com tenant (ticket IT-4821).",
        "locale": "en"
      },
      "params": { "client_name": "invoice-service", "environment": "production" },
      "constraints": { "expires_at": "2026-03-10T18:00:00Z" }
    }
  ]
}
~~~

# Use Cases

## Multi-Tenant SaaS

A SaaS provider serves multiple enterprise customers. Each customer (tenant) may have configured:

* A specific SAML or OIDC identity provider for their employees.
* MFA requirements or exemptions.
* A set of permitted identity attributes to be included in tokens.

**Example:**

~~~json
"client_context": [
  { "type": "app", "id": "project_management" },
  { "type": "tenant", "id": "acme-corp.com" }
]
~~~

The OP uses the tenant context to route the user to Acme Corp's configured identity provider and apply Acme Corp's authentication policy, then returns the tenant context in the ID Token so the application can load Acme Corp's workspace.

## Just-in-Time Administrative Access

Just-in-time access is a pattern, not a product category. Any application can adopt it: rather than relying on a broad administrative session established at login, the client requests a fresh authentication scoped to the specific operation the user is about to perform. This reduces the window of exposure if a session is compromised and creates a clear audit trail linking each privileged action to a deliberate authentication event.

**Example: An admin panel requesting authentication to register a new OAuth client:**

~~~json
"client_context": [
  { "type": "app", "id": "admin_console" },
  { "type": "tenant", "id": "example.com" },
  {
    "type": "purpose",
    "id": "add_oauth_client",
    "instance_id": "01JNP8Q8W5W4K7QG0R9B3R2M6F",
    "display": {
      "title": "Register OAuth Client",
      "description": "You are authorizing registration of a new OAuth client 'invoice-service' in the example.com tenant (ticket IT-4821).",
      "locale": "en"
    },
    "params": { "client_name": "invoice-service", "environment": "production" },
    "constraints": { "expires_at": "2026-03-10T18:00:00Z" }
  }
]
~~~

The OP receiving this request might:

1. Require the user to satisfy a high-assurance authentication policy (e.g., phishing-resistant MFA) regardless of any existing session.
2. Display the purpose details to the user for explicit confirmation.
3. Limit the validity of the authenticated authorization to act so that it does not outlive `expires_at`.
4. Record the purpose identifier and task details in its audit log.
5. Notify a security team of the elevated access event.

The Client then verifies the returned context before enabling the client registration action, ensuring the user cannot perform this action if the OP did not acknowledge the purpose context.

## AI Agent Authorization

AI agents act on behalf of users to perform specific tasks. An agent platform registers a single `client_id`, but individual agent sessions operate under specific, bounded tasks. Using `client_context`, the platform can communicate the task context to the OP, enabling:

* **Step-up authentication** when an agent attempts to access sensitive systems.
* **Constrained session lifetime** tied to the task duration.
* **Audit trail** associating authentication events with specific agent tasks.
* **User consent** tied to the specific task the agent will perform.

**Example: An AI assistant requesting access to a user's calendar to schedule a meeting:**

~~~json
"client_context": [
  { "type": "app", "id": "ai_assistant" },
  {
    "type": "purpose",
    "id": "schedule_meeting",
    "display": {
      "title": "Schedule a Meeting",
      "description": "AI Assistant will access your calendar to schedule a team meeting for next Tuesday.",
      "locale": "en"
    },
    "initiator": { "type": "agent", "id": "agent-session-7f3a2b1c" },
    "constraints": { "expires_at": "2026-03-10T23:00:00Z" }
  }
]
~~~

The OP might respond by:

1. Displaying a consent screen to the user: "AI Assistant wants to access your calendar to schedule a meeting. Allow?"
2. Issuing an ID Token with a short lifetime (e.g., 1 hour) matching the task scope.
3. Including the `purpose` context in the ID Token so that the agent platform can verify the user authenticated specifically for this task.
4. Issuing an access token with scopes constrained to `calendar.read` and `calendar.write`.

### Why Not Token Exchange or RAR?

Two existing mechanisms might appear to address this scenario but operate at a different layer of the protocol stack.

**OAuth 2.0 Token Exchange ({{!RFC8693}})** allows an agent to obtain a new access token derived from an existing one, for example, impersonating a user or narrowing an existing token's scope. Token exchange is a *credential propagation* mechanism: it operates on tokens that already exist and produces new access tokens for use at resource servers. It says nothing about whether the user *authenticated* for the current task, nor does it give the OP an opportunity to step up authentication or present the user with task-specific consent. The user is not present during a token exchange.

**OAuth 2.0 Rich Authorization Requests ({{!RFC9396}})** allow a client to request fine-grained authorization for an access token by including structured `authorization_details` in the token request. RAR is an *access control* mechanism: it shapes the permissions encoded in an access token presented to a resource server. It does not influence how the OP authenticates the user, what assurance level is required, or what the ID Token contains.

`client_context` addresses a different problem: it is an *authentication context* mechanism. Its purpose is to inform the OP why authentication is being requested right now, so that the OP can apply the appropriate authentication policy and return a verifiable record of that context in the ID Token. The key distinctions are:

| | Token Exchange | RAR (`authorization_details`) | `client_context` |
|---|---|---|---|
| Layer | Token endpoint | Authorization / token request | Authorization request |
| User present | No | Optional | Yes |
| Drives authentication policy | No | No | Yes |
| Step-up authentication | No | No | Yes |
| Recorded in ID Token | No | No | Yes |
| Scopes access token | Indirectly | Yes | No |

In an AI agent deployment, all three mechanisms may be used together and are complementary. A typical flow might look like:

1. The agent platform uses `client_context` with a `purpose` object to authenticate the user specifically for the current task, obtaining step-up MFA and task-scoped consent from the OP. The ID Token confirms the user authenticated specifically for this purpose.

2. The same authorization request includes `authorization_details` (RAR) to request fine-grained permissions on the resulting access token, for example, constraining the token to read-only calendar access for a specific date range.

3. If the agent later needs to act on a sub-task using a downstream service, it may use Token Exchange to propagate its credentials, deriving a narrower access token without requiring the user to re-authenticate.

In this layered model, `client_context` handles the authentication event, RAR handles the access token shape, and Token Exchange handles downstream credential propagation. They do not overlap.

# Third-Party Initiated Login

## Overview

OpenID Connect Third Party Initiated Login {{!OIDC.TPIL}} defines a mechanism by which an entity other than the Client (the "initiating party") can start an authentication flow on behalf of a user. The initiating party redirects the user to the Client's login initiation endpoint with parameters, typically `iss`, `login_hint`, and `target_link_uri`, that the Client uses to construct an authorization request to the OP.

A common scenario is an application portal or identity hub that launches a specific application in a specific context on behalf of a user:

* An enterprise portal launching a SaaS application in the context of a specific tenant.
* An orchestration layer triggering an agent workflow by directing the user to authenticate for a specific purpose.
* A service desk tool initiating a just-in-time administrative authentication from within a ticket management interface.

In each case, the initiating party knows the context under which the authentication should occur, but it is the Client, not the initiating party, that constructs the `client_context` parameter and sends the authorization request. A mechanism is needed for the initiating party to communicate the desired context to the Client.

This specification defines the `client_context_hint` parameter for use at the Client's login initiation endpoint.

## The `client_context_hint` Parameter

`client_context_hint`
: OPTIONAL. A JSON array of context objects in the same format as `client_context`. Passed by an initiating party to the Client's login initiation endpoint to suggest the context under which authentication should be requested.

The parameter is a *hint*. The Client determines how to use it. The Client MUST NOT forward the hint value directly to the OP without validation.

**Example: An enterprise portal initiating a login for a specific tenant and application:**

~~~
GET /login-initiation?
  iss=https%3A%2F%2Fop.example.com
  &target_link_uri=https%3A%2F%2Fclient.example.org%2Fapp%2Fcalendar
  &login_hint=alice%40example.com
  &client_context_hint=%5B%7B%22type%22%3A%22app%22%2C%22id%22%3A%22calendar%22%7D%2C
    %7B%22type%22%3A%22tenant%22%2C%22id%22%3A%22example.com%22%7D%5D
  HTTP/1.1
Host: client.example.org
~~~

The decoded `client_context_hint` value is:

~~~json
[
  { "type": "app", "id": "calendar" },
  { "type": "tenant", "id": "example.com" }
]
~~~

**Example: A service desk initiating a just-in-time administrative authentication:**

~~~json
[
  { "type": "app", "id": "admin_console" },
  { "type": "tenant", "id": "example.com" },
  {
    "type": "purpose",
    "id": "add_oauth_client",
    "display": {
      "title": "Register OAuth Client",
      "description": "You are authorizing registration of a new OAuth client 'invoice-service' in the example.com tenant (ticket IT-4821).",
      "locale": "en"
    },
    "params": { "client_name": "invoice-service", "ticket_ref": "IT-4821" },
    "initiator": { "type": "service", "id": "service-desk-app" },
    "constraints": { "expires_at": "2026-03-10T18:00:00Z" }
  }
]
~~~

Here the service desk application knows the task being performed but is not itself the application that will perform it; it directs the user to the admin console client to authenticate and execute the action.

## Client Processing of the Hint

The Client receiving a `client_context_hint` at its login initiation endpoint MUST treat the hint as untrusted input from a third party. The following processing rules apply:

1. **Parse.** The Client MUST parse the hint value as a JSON array. If parsing fails, the Client MUST ignore the hint and proceed with default context or return an error to the initiating party.

2. **Validate context types.** The Client MUST verify that each context type in the hint is one the Client supports and is permitted for this Client's registration. The Client MUST discard or reject any object whose `type` is not in the Client's permitted set.

3. **Validate context values.** The Client MUST validate each field value against its own configuration, for example, confirming that a `tenant.id` value is a tenant the Client is configured to serve, and that a `purpose.id` value is in the Client's list of permitted purpose identifiers. The Client MUST NOT accept context values that would grant access or privilege not authorized by the Client's own policy.

4. **Construct `client_context`.** After validation, the Client uses the validated hint, possibly enriched with additional context the Client determines from its own state, as the basis for the `client_context` parameter in the authorization request to the OP.

   When a hint includes an `initiator` object, the Client MUST NOT forward that value unchanged unless it has independently authenticated and verified the initiating entity. Otherwise, the Client MUST remove the hinted `initiator` or replace it with a locally derived value.

5. **Authenticate the initiating party.** Clients SHOULD authenticate or verify the origin of the initiation request where possible (e.g., by requiring the initiating party to be a registered application in the same platform, or by verifying a signed initiation token). Unauthenticated initiation requests with `client_context_hint` SHOULD be limited to low-privilege context types.

The result of this processing is that the Client, not the initiating party, is responsible for the `client_context` that reaches the OP. The hint influences but does not control the authorization request.

## Discovery Metadata

Clients that accept `client_context_hint` at their login initiation endpoint SHOULD advertise this in their client metadata or in a well-known configuration endpoint if one is published.

The following Client metadata parameter is defined for use where Client metadata is published:

`client_context_hint_types_supported`
: OPTIONAL. JSON array of context type values that this Client accepts in `client_context_hint` at its login initiation endpoint.

# Relationship to Other Mechanisms

## Relationship to Authentication Context (`acr`)

The `client_context` parameter and the `acr` claim serve complementary but distinct roles. `client_context` is a *request input* describing the business context of the authentication. The `acr` claim is an *output* describing the authentication assurance that was actually satisfied.

The OP MAY use `client_context` as an input when determining the authentication assurance required. For example, an OP might define policy such as: "any request with a `purpose` context whose `id` matches a high-privilege operation list requires `acr` `phishing-resistant-mfa`."

Clients SHOULD evaluate both the returned `client_context` and the `acr` claim when determining whether to permit privileged operations. A matching `client_context` with insufficient `acr` should not be sufficient to authorize a high-privilege action.

## Relationship to Rich Authorization Requests

OAuth 2.0 Rich Authorization Requests ({{!RFC9396}}) define the `authorization_details` parameter to convey fine-grained authorization semantics for access tokens.

The two parameters serve different layers of the protocol:

* `authorization_details` describes *what* the Client is authorized to do, the permissions attached to an access token used at a resource server.
* `client_context` describes *the context of the authentication event itself*, inputs to how the OP authenticates the user and what it includes in the ID Token.

A request MAY include both parameters. For example, an AI agent request might include `client_context` with a `purpose` object (to drive authentication policy) and `authorization_details` with specific resource permissions (to scope the resulting access token).

## Relationship to Resource Indicators

OAuth 2.0 Resource Indicators ({{!RFC8707}}) allow a Client to identify the protected resource for which an access token is intended, enabling the OP to audience-restrict the access token.

The `resource` parameter indicates the *target* of an access token. The `client_context` parameter conveys contextual information driving *authentication behavior and ID Token content*. These are complementary; both MAY be present in the same request.

# OP Discovery Metadata

An Authorization Server that supports `client_context` SHOULD advertise this capability in its OpenID Provider Metadata ({{!OIDC.Discovery}}).

The following metadata parameters are defined:

`client_context_types_supported`
: OPTIONAL. JSON array of strings identifying the context type values supported by this OP. If omitted, clients SHOULD NOT assume any types are supported.

`client_context_schemas`
: OPTIONAL. JSON object mapping supported context type values to URLs that resolve to JSON Schema documents describing the structure of that context type.

`client_context_par_required`
: OPTIONAL. Boolean. If `true`, the OP requires that requests containing `client_context` be submitted as Pushed Authorization Requests. Default is `false`.

**Example discovery document excerpt:**

~~~json
{
  "issuer": "https://op.example.com",
  "authorization_endpoint": "https://op.example.com/authorize",
  "client_context_types_supported": [
    "app",
    "tenant",
    "purpose"
  ],
  "client_context_schemas": {
    "app": "https://op.example.com/schemas/context/app.json",
    "tenant": "https://op.example.com/schemas/context/tenant.json",
    "purpose": "https://op.example.com/schemas/context/purpose.json"
  },
  "client_context_par_required": true
}
~~~

# Client Registration Metadata

The following client registration metadata parameter is defined for use with the OpenID Connect Dynamic Client Registration Protocol {{!OIDC.Registration}} and equivalent static registration mechanisms.

`client_context_types`
: OPTIONAL. JSON array of strings listing the context type values that this client is permitted to include in `client_context` parameters. The OP SHOULD reject requests from this client that include context types not in this list.

`client_context_values`
: OPTIONAL. JSON object mapping context type values to arrays of permitted identifier values. For example, to restrict a client to specific application IDs or purpose identifiers:

~~~json
{
  "client_context_types": ["app", "tenant", "purpose"],
  "client_context_values": {
    "app": ["calendar", "email", "admin_console"],
    "purpose": ["add_oauth_client", "approve_change_request"]
  }
}
~~~

This registration metadata allows OPs to enforce allow-lists on context values, preventing clients from escalating their context beyond what was administratively approved.

# Error Handling {#errors}

Authorization Servers MUST return an error response conforming to Section 4.1.2.1 of {{!RFC6749}} if the `client_context` parameter is malformed or rejected.

The following error codes are defined:

`invalid_client_context`
: The `client_context` parameter could not be parsed as a JSON array, or one or more elements are not valid JSON objects, or a required field is missing.

`unsupported_client_context_type`
: The request includes a context type that is not supported by this Authorization Server or not permitted for this client's registration.

`invalid_client_context_value`
: A context object contains one or more invalid field values (e.g., an `expires_at` in the past, an unrecognized `id` value for a constrained purpose type).

**Example error response:**

~~~
HTTP/1.1 302 Found
Location: https://client.example.org/cb?
  error=invalid_client_context
  &error_description=Context+type+%22purpose%22+is+not+permitted+for+this+client
  &state=af0ifjsldkj
~~~

For requests submitted via PAR, errors are returned in the PAR response:

~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "unsupported_client_context_type",
  "error_description": "Context type 'purpose' is not supported by this server"
}
~~~

# Security Considerations

## Context Integrity

Because `client_context` influences authentication policy and the contents of ID Tokens, it is important to protect its integrity in transit.

Clients SHOULD transmit requests containing `client_context` using one of:

* **Pushed Authorization Requests ({{!RFC9126}})**: Context is submitted directly to the OP over an authenticated server-to-server TLS connection, preventing tampering by user agents or intermediaries. This is the RECOMMENDED mechanism.
* **Signed Request Objects**: When PAR is not available, Client SHOULD sign the Request Object containing `client_context` using the client's private key registered at the OP.

Clients MUST NOT transmit sensitive purpose context (e.g., administrative workflow identifiers) as unprotected query parameters where it may be logged by servers, proxies, or browsers.

## Context Injection

A malicious party who can modify the authorization request before it reaches the OP could inject or alter context to escalate privileges. For example, substituting a benign application context for an administrative one.

Mitigations:

* Use PAR or signed Request Objects to prevent modification in transit.
* OPs MUST validate requested context values against the client's registered `client_context_values` allow-list when one is configured.
* Clients MUST verify returned context in the ID Token and not assume that submitted context was accepted as-is.

## Context Verification

A Client that uses `client_context` to gate privileged operations MUST verify the returned `client_context` claim in the ID Token before enabling those operations. Failure to verify allows an attacker who obtains an ID Token issued in a different context to abuse it for privileged actions.

The returned `client_context` claim is protected by the ID Token's signature, so verifying the token signature and then checking the claim value provides cryptographic assurance that the OP confirmed the context.

## Token Replay Across Contexts

Without context binding, an ID Token issued in the context of one application could potentially be replayed in the context of another application hosted by the same client. Including `client_context` in the ID Token binds the token to the context in which it was issued.

Clients SHOULD include `nonce` in authorization requests containing `client_context`, and MUST verify the `nonce` in the returned ID Token. Together with context verification, this prevents token replay across contexts and sessions.

## Over-Privileged Context Requests

A compromised or malicious Client might attempt to request elevated context beyond what is appropriate, for example, requesting `purpose` identifiers corresponding to highly privileged operations.

Authorization Servers SHOULD:

* Maintain an allow-list of permitted context values per client registration (`client_context_values`).
* Reject requests for context types or values not in the allow-list with `unsupported_client_context_type` or `invalid_client_context_value`.
* Log context requests for audit and anomaly detection.

## Purpose Context Expiration

When `expires_at` is included in a `purpose` context object, the OP SHOULD enforce that the authenticated authorization to perform that purpose does not outlive this value. Any resulting session, token, or other local authorization state used to carry out the purpose SHOULD expire no later than `expires_at`. Clients SHOULD request a new authentication for a new purpose rather than reusing an expired-purpose authorization.

## Third-Party Hint Injection

The `client_context_hint` parameter is supplied by an initiating party that may be unauthenticated or partially trusted. An attacker who can send a crafted login initiation request could attempt to inject elevated context values, for example, a `purpose` identifier corresponding to a privileged administrative workflow, in an attempt to cause the Client to request step-up authentication and then perform an action the attacker desires.

Clients MUST validate all hint values against their own allowlists before using them to construct `client_context`. Clients MUST NOT treat the presence of a hint value as equivalent to the Client's own policy decision to use that context. Where possible, Clients SHOULD authenticate the source of the initiation request before accepting elevated context types such as `purpose`.

## Confidentiality of Purpose Context

Purpose context MAY contain information that is sensitive from a business or operational security perspective, for example, the name of an administrative action being performed or the identity of a user being affected. Clients SHOULD ensure that purpose context is not unnecessarily logged or exposed.

# Privacy Considerations

The `client_context` parameter MAY convey information about the End-User's activities (e.g., which application they are using, the administrative actions they are performing, or the tasks they have delegated to an AI agent). This information is transmitted to the OP and returned in the ID Token, potentially making it available to other parties.

Implementations SHOULD minimize the information included in `client_context` to what is necessary for authentication policy and audit purposes. Free-text fields such as `display.description` SHOULD NOT contain personally identifiable information beyond what is necessary.

OPs MUST handle `client_context` data in accordance with their privacy policy and applicable law. Context data SHOULD NOT be used for purposes beyond authentication policy evaluation and audit logging without the End-User's explicit consent.

# IANA Considerations

## OAuth Parameters Registry

This specification requests registration of the following parameters in the "OAuth Parameters" registry defined in {{!RFC6749}}.

* **Parameter name:** `client_context`
* **Parameter usage location:** authorization request
* **Change controller:** IETF
* **Specification document(s):** This document

* **Parameter name:** `client_context_hint`
* **Parameter usage location:** login initiation endpoint request
* **Change controller:** IETF
* **Specification document(s):** This document

## OAuth Authorization Server Metadata Registry

This specification requests registration of the following metadata in the "OAuth Authorization Server Metadata" registry defined in {{!RFC8414}}.

| Metadata Name | Specification |
|---|---|
| `client_context_types_supported` | This document |
| `client_context_schemas` | This document |
| `client_context_par_required` | This document |

## OAuth Dynamic Client Registration Metadata Registry

This specification requests registration of the following metadata in the "OAuth Dynamic Client Registration Metadata" registry defined in {{!RFC7591}}.

| Metadata Name | Specification |
|---|---|
| `client_context_types` | This document |
| `client_context_values` | This document |
| `client_context_hint_types_supported` | This document |

## OAuth Client Context Type Registry

This specification establishes the "OAuth Client Context Type" registry. The registry records context type identifiers that may appear in the `type` field of a `client_context` object.

**Registration policy:** Specification Required per {{!RFC8126}}. Designated experts SHOULD evaluate whether the proposed context type:

* Has a clear, stable semantics.
* Does not overlap significantly with existing registered types.
* Is backed by a publicly available specification.

**Initial registry contents:**

| Type | Description | Reference |
|---|---|---|
| `app` | Identifies the logical application within a multi-application client | This document |
| `tenant` | Identifies the organizational tenant associated with the request | This document |
| `purpose` | Identifies the operational task or workflow for which authentication is requested | This document |

Private-use context types MUST use URIs as their `type` value (e.g., `"https://example.com/context-types/custom"`). Short string type names (i.e., non-URI strings) are reserved for registered types.

## JWT Claims Registry

This specification requests registration of the following claim in the "JSON Web Token Claims" registry defined in {{!RFC7519}}.

* **Claim name:** `client_context`
* **Claim description:** Client context applied during authentication
* **Change controller:** IETF
* **Specification document(s):** This document


--- back

# Combined Request Example: `client_context`, `acr_values`, and `authorization_details` {#combined-example}

This appendix illustrates how `client_context`, `acr_values`, and `authorization_details` ({{!RFC9396}}) work together in a single authorization request. Each parameter operates at a distinct layer and carries information for a different consumer.

The scenario: a user of an admin console is about to register a new OAuth client application. The application needs to:

* Ensure the user authenticates with phishing-resistant MFA for this specific operation (`acr_values`).
* Record in the ID Token that this authentication was for the `add_oauth_client` purpose, bound to the target client name and environment (`client_context`).
* Obtain an access token scoped precisely to the Client Registration API for that specific user (`authorization_details`).

## Authorization Request

The following PAR request body carries all three parameters:

~~~
POST /as/par HTTP/1.1
Host: op.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

response_type=code
&client_id=s6BhdRkqt3
&redirect_uri=https%3A%2F%2Fadmin.example.org%2Fcb
&scope=openid%20email
&nonce=n-0S6_WzA2Mj
&acr_values=urn%3Aietf%3Aparams%3Aac%3Aclasses%3Aphishing-resistant
&client_context=...
&authorization_details=...
~~~

The decoded `client_context` value:

~~~json
[
  { "type": "app", "id": "admin_console" },
  { "type": "tenant", "id": "example.com" },
  {
    "type": "purpose",
    "id": "add_oauth_client",
    "instance_id": "01JNP8Q8W5W4K7QG0R9B3R2M6F",
    "display": {
      "title": "Register OAuth Client",
      "description": "You are authorizing registration of a new OAuth client 'invoice-service' in the example.com tenant (ticket IT-4821).",
      "locale": "en"
    },
    "params": {
      "client_name": "invoice-service",
      "environment": "production"
    },
    "constraints": {
      "expires_at": "2026-03-10T18:00:00Z"
    }
  }
]
~~~

The decoded `authorization_details` value:

~~~json
[
  {
    "type": "oauth_client_registration",
    "locations": ["https://as.example.com/register"],
    "actions": ["client:create"],
    "environment": "production"
  }
]
~~~

## What Each Parameter Does

`acr_values`
: Tells the OP what authentication assurance level is required. The OP uses this to enforce that the user satisfies phishing-resistant MFA before issuing tokens. This is a requirement on the *authentication method*, independent of any context.

`client_context`
: Tells the OP *why* this authentication is occurring. The OP uses the `purpose` object to enforce that this is a purpose-scoped re-authentication (not a reuse of an existing session), present the operation details to the user on the consent screen, constrain the validity window for acting on the authenticated purpose via `expires_at`, and record the purpose, params, and initiator in its audit log. The applied context is returned in the ID Token for the client to verify.

`authorization_details`
: Tells the authorization server what permissions to encode in the access token. The resulting access token is scoped to `client:create` action on the Admin API for the target tenant. This is enforced at the resource server, not by the OP's authentication engine.

## Resulting ID Token

~~~json
{
  "iss": "https://op.example.com",
  "sub": "248289761001",
  "aud": "s6BhdRkqt3",
  "nonce": "n-0S6_WzA2Mj",
  "iat": 1741564800,
  "exp": 1741566600,
  "acr": "urn:ietf:params:ac:classes:phishing-resistant",
  "amr": ["hwk", "pin"],
  "email": "bob@example.com",
  "client_context": [
    { "type": "app", "id": "admin_console" },
    { "type": "tenant", "id": "example.com" },
    {
      "type": "purpose",
      "id": "add_oauth_client",
      "instance_id": "01JNP8Q8W5W4K7QG0R9B3R2M6F",
      "params": { "client_name": "invoice-service", "environment": "production" },
      "constraints": { "expires_at": "2026-03-10T18:00:00Z" }
    }
  ]
}
~~~

The `acr` claim confirms phishing-resistant MFA was satisfied. The `client_context` claim confirms the authentication was bound to the `add_oauth_client` purpose for the specified target. The client verifies both before enabling the client registration.

## Layered Model Summary

~~~
                    Authorization Request
                           |
          +----------------+----------------+
          |                |                |
    acr_values       client_context   authorization_details
          |                |                |
          v                v                v
   Authentication    ID Token         Access Token
      Policy         Content           Scoping
          |                |                |
          v                v                v
    "Did the user    "Was this auth    "What can the
    satisfy MFA?"    for this purpose?" token do?"
          |                |                |
          v                v                v
        OP               OP            Resource Server
   (enforced at      (verified by      (enforced at
   auth time)         the client)       the API)
~~~
