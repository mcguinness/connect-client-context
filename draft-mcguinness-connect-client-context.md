
---
title: "OpenID Connect Client Context Parameter"
abbrev: "OIDC Client Context"
docName: "draft-mcguinness-connect-client-context-00"
category: std
ipr: trust200902
area: Security
workgroup: "OpenID Connect"
keyword:
  - OpenID Connect
  - OAuth 2.0
  - Client Context
  - Mission Context
  - Just-in-Time Access
  - AI Agents
venue:
  group: "OpenID Connect"
  type: Working Group
  mail: "openid-specs-ab@lists.openid.net"
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
  RFC8707:
  RFC9126:
  RFC9396:
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

informative:
  NIST.SP.800-53r5:
    title: "Security and Privacy Controls for Information Systems and Organizations"
    author:
      - org: National Institute of Standards and Technology
    date: 2020-09
    seriesinfo:
      NIST: SP 800-53 Rev. 5
---

# Abstract

This specification defines the `client_context` authorization request parameter for OpenID Connect. The parameter enables a Client to convey structured contextual information to the OpenID Provider (OP) so that the OP can apply context-specific authentication policy and claim release when issuing an ID Token.

The mechanism is designed for deployments where a single `client_id` represents multiple logical applications, tenants, or operational missions — such as AI agents acting on behalf of users or just-in-time administrative workflows requiring privilege elevation.

The OP returns the applied context in the ID Token, allowing the Client to verify that authentication occurred under the intended context and to bind subsequent operations accordingly.

# Status of This Memo

This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79.

Internet-Drafts are working documents of the Internet Engineering Task Force (IETF). Note that other groups may also distribute working documents as Internet-Drafts. The list of current Internet-Drafts is at https://datatracker.ietf.org/drafts/current/.

Internet-Drafts are draft documents valid for a maximum of six months and may be updated, replaced, or obsoleted by other documents at any time. It is inappropriate to use Internet-Drafts as reference material or to cite them other than as "work in progress."

# Introduction

OpenID Connect Core {{!OIDC.Core}} defines a protocol for authenticating End-Users and conveying identity claims to Clients via ID Tokens issued by an OpenID Provider (OP). While this model works well for simple deployments with a one-to-one relationship between a registered client and a logical application, modern deployments increasingly require richer context.

## The Multi-Context Problem

A growing class of deployments uses a single registered `client_id` to represent multiple distinct logical contexts. Consider:

**Multi-application SaaS platforms.** A platform vendor registers one client but hosts dozens of distinct applications (calendar, email, CRM). Each application may require different authentication policies — the email client might allow password authentication, while the financial reporting module requires MFA. The OP has no way to distinguish which application is requesting authentication.

**Multi-tenant enterprise services.** A B2B SaaS provider serves many enterprise customers under one client registration. Each tenant may have its own password policy, MFA requirements, or allowed identity providers (e.g., only employees from `example.com` may authenticate to the `example.com` tenant). Without tenant context, the OP cannot enforce per-tenant policy.

**AI agents acting on behalf of users.** An AI assistant platform registers a single client but spawns many concurrent agent sessions, each operating under a specific task or mission (e.g., "draft a response to these emails" or "provision a new cloud environment"). The OP needs to understand the agent's operational context to determine whether step-up authentication is appropriate and which claims to release.

**Just-in-time administrative workflows.** A privileged access management (PAM) console requires users to authenticate specifically for bounded administrative tasks — resetting another user's MFA, approving a change request, or reviewing audit logs — rather than relying on a long-lived session with broad administrative privileges. The OP needs mission context to enforce step-up authentication and restrict the resulting session.

## Gap in Existing Parameters

Existing OpenID Connect and OAuth 2.0 parameters each serve a distinct purpose but none address client-local identity context:

* `scope` indicates which permissions or resources are requested — not why or under which application context.
* `claims` requests specific identity attributes — not contextual information about the request itself.
* `acr_values` communicates the desired authentication assurance level — not the business or mission context that determines what assurance is appropriate.
* `resource` ({{!RFC8707}}) identifies the protected resource audience for access tokens — not the logical application or tenant context for authentication.
* `authorization_details` ({{!RFC9396}}) conveys structured authorization semantics for access tokens — not contextual metadata for ID Token issuance.

None of these parameters provide a structured, extensible mechanism for a Client to communicate the logical context under which authentication is requested.

## This Specification

This specification introduces the `client_context` request parameter to address this gap. The parameter carries a structured array of typed context objects that the OP can use as input to authentication policy, claim release decisions, and audit logging. The OP returns the applied context in the ID Token, enabling the Client to cryptographically verify that authentication was performed under the intended context.

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
: A registered identifier (e.g., `app`, `tenant`, `mission`) denoting the semantic category of a context object.

**Context Object**
: A JSON object within the `client_context` array, containing at minimum a `type` field and type-specific members.

**Mission**
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
   |     { type: "mission",              |
   |       code: "reset_user_mfa" }      |
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
   |     { type: "mission",              |
   |       code: "reset_user_mfa" }      |
   |   ]                                 |
   |   ...standard claims...             |
   |                                     |
   |  [Client verifies context]          |
   |  [Enables mission-bound actions]    |
~~~

The protocol proceeds as follows:

1. The Client constructs an authorization request that includes `client_context` alongside standard OpenID Connect parameters.

2. The OP receives the request and validates the `client_context` parameter structure.

3. The OP evaluates the supplied context as input to its authentication policy — for example, determining that a `mission` context with code `reset_user_mfa` requires step-up MFA authentication, or that a `tenant` context restricts authentication to a specific identity provider.

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

**Direct query parameter.** The JSON array is URL-encoded and included as a query parameter. This is simple but exposes context values in server logs and browser history. Not RECOMMENDED for sensitive mission context.

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
    { "type": "mission", "code": "reset_user_mfa",
      "expires_at": "2026-03-10T18:00:00Z" }
  ]
}
~~~

**Pushed Authorization Request (PAR).** The parameter is included in the body of a PAR request per {{!RFC9126}}. This is the RECOMMENDED approach for requests containing sensitive context (e.g., mission context for privileged workflows). PAR ensures the context is submitted directly to the OP over a server-to-server TLS connection, preventing exposure in browser redirects.

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
  %7B%22type%22%3A%22mission%22%2C%22code%22%3A%22reset_user_mfa%22%7D%5D
~~~

# Client Processing Rules

## Constructing the Request

A Client generating an authorization request containing `client_context` MUST:

1. Encode the parameter value as a JSON array. An empty array is NOT RECOMMENDED; omit the parameter entirely if no context is applicable.
2. Ensure each element is a JSON object containing a `type` field with a non-empty string value.
3. Ensure each object's structure conforms to the schema defined for that context type.
4. Avoid including contradictory context values (e.g., two `tenant` objects with different `id` values).
5. Include a `nonce` parameter in the authorization request to bind the resulting ID Token to the request.

Clients SHOULD use PAR ({{!RFC9126}}) or a signed Request Object when `client_context` contains sensitive information such as mission codes or administrative identifiers.

## Validating the Response

When receiving an ID Token in response to a request that included `client_context`, the Client MUST:

1. Validate the ID Token per Section 3.1.3.7 of {{!OIDC.Core}}, including signature verification, issuer, audience, and expiration checks.
2. Verify that the `nonce` claim in the ID Token matches the `nonce` sent in the request.
3. If the request included `client_context`, examine the `client_context` claim in the ID Token.
4. Compare the returned context to the requested context. The Client MUST detect and handle:
   - **Missing context:** A context type that was requested is absent from the returned value. The Client MUST apply local policy to determine whether this is acceptable. For security-critical context (e.g., mission context for privileged operations), the Client MUST treat missing context as an error and reject the token.
   - **Modified context:** A context value was altered by the OP (e.g., due to normalization). The Client MUST evaluate whether the modification is acceptable.
   - **Additional context:** The OP included context objects not present in the request. The Client SHOULD ignore unexpected context types.
5. Reject or restrict privileged operations if required context values are absent or inconsistent.

## Example: Client Verification Logic

The following pseudocode illustrates how a Client might verify returned context for a mission-bound administrative operation:

~~~
id_token = validate_and_parse_jwt(token_response.id_token)

# Standard validation
assert id_token.iss == expected_issuer
assert id_token.aud == client_id
assert id_token.nonce == request_nonce

# Context verification
returned_context = id_token.client_context  # array of objects

mission = find(returned_context, type="mission")
if mission is None:
    raise SecurityError("Mission context missing from ID Token")
if mission.code != requested_mission_code:
    raise SecurityError("Mission context mismatch")

tenant = find(returned_context, type="tenant")
if tenant is None or tenant.id != expected_tenant:
    raise SecurityError("Tenant context invalid")

# All checks passed — enable mission-scoped operations
enable_operations(mission.code)
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
* **Mission context** MAY trigger step-up authentication, require explicit re-authentication regardless of existing session, reduce session and token lifetimes, or require additional approval.

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
    { "type": "mission", "code": "reset_user_mfa",
      "expires_at": "2026-03-10T18:00:00Z" }
  ]
}
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

## Mission Context (`mission`)

The `mission` context type identifies the specific operational task, workflow, or purpose for which authentication is being requested. Mission context conveys *why* the authentication is occurring, enabling the OP to apply purpose-specific policy.

Mission context is the primary mechanism for:

* **AI agent task scoping** — identifying the specific task an agent is performing on a user's behalf, so that authentication and session scope can be limited to that task.
* **Just-in-time privilege elevation** — requesting authentication for a specific administrative action rather than a general administrative session.
* **Automated workflow authorization** — identifying a machine-to-machine workflow requiring human authentication for a specific step.
* **Delegated user actions** — recording that a user is authenticating to authorize a specific action taken by another entity.

**Required members:**

`type`
: MUST be `"mission"`.

`code`
: REQUIRED. String identifier for the mission or task type. The code space is defined by the client and registered in client metadata. Example values: `"reset_user_mfa"`, `"approve_change_request"`, `"deploy_to_production"`, `"summarize_inbox"`.

**Optional members:**

`task`
: OPTIONAL. String or object providing a human-readable description of the specific task instance. MAY be displayed to the End-User during authentication to provide context for what they are authorizing. For example: `"Reset MFA for user alice@example.com"`.

`delegated_by`
: OPTIONAL. String identifying the entity (system, agent, or process) that initiated the mission and is requesting authentication on the End-User's behalf. For AI agents, this SHOULD be a stable agent or session identifier.

`allowed_actions`
: OPTIONAL. Array of strings enumerating the specific actions the Client intends to perform under this mission. This is informational for the OP and for the End-User during the authentication consent step. Example: `["user.mfa.reset", "user.mfa.enroll"]`.

`data_scope`
: OPTIONAL. String or object describing the data scope of the mission, indicating which data the Client will access or modify. Intended to inform user consent and OP audit logging.

`expires_at`
: OPTIONAL. String containing a date-time value (formatted per {{!RFC3339}}) indicating when the mission expires. The OP SHOULD ensure the resulting session and any tokens do not outlive this value.

**Example — AI agent task:**

~~~json
{
  "type": "mission",
  "code": "summarize_inbox",
  "task": "Summarize unread emails from the past 7 days",
  "delegated_by": "agent-session-a1b2c3",
  "allowed_actions": ["email.read"],
  "data_scope": "email:inbox:unread",
  "expires_at": "2026-03-10T20:00:00Z"
}
~~~

**Example — just-in-time administrative action:**

~~~json
{
  "type": "mission",
  "code": "reset_user_mfa",
  "task": "Reset MFA for user alice@example.com",
  "allowed_actions": ["user.mfa.reset", "user.mfa.enroll"],
  "expires_at": "2026-03-10T18:00:00Z"
}
~~~

**OP policy guidance:** The OP SHOULD present the mission details to the End-User during authentication so they understand what they are authorizing. The OP SHOULD enforce that session and token lifetimes do not exceed `expires_at`. The OP MAY require step-up authentication for mission codes designated as high-privilege in its configuration.

# Use Cases

## AI Agent Authorization

AI agents act on behalf of users to perform specific tasks. An agent platform registers a single `client_id`, but individual agent sessions operate under specific, bounded tasks. Using `client_context`, the platform can communicate the task context to the OP, enabling:

* **Step-up authentication** when an agent attempts to access sensitive systems.
* **Constrained session lifetime** tied to the task duration.
* **Audit trail** associating authentication events with specific agent tasks.
* **User consent** for the specific data and actions the agent will access.

**Example: An AI assistant requesting access to a user's calendar to schedule a meeting:**

~~~json
"client_context": [
  { "type": "app", "id": "ai_assistant" },
  {
    "type": "mission",
    "code": "schedule_meeting",
    "task": "Schedule a team meeting for next Tuesday",
    "delegated_by": "agent-session-7f3a2b1c",
    "allowed_actions": ["calendar.read", "calendar.write"],
    "expires_at": "2026-03-10T23:00:00Z"
  }
]
~~~

The OP might respond by:

1. Displaying a consent screen to the user: "AI Assistant wants to access your calendar to schedule a meeting. Allow?"
2. Issuing an ID Token with a short lifetime (e.g., 1 hour) matching the task scope.
3. Including the `mission` context in the ID Token so that the agent platform can verify authorization.
4. Potentially issuing an access token with scopes constrained to `calendar.read` and `calendar.write`.

## Just-in-Time Administrative Access

Privileged access management (PAM) systems benefit from just-in-time (JIT) access patterns: users authenticate for a specific administrative task rather than establishing long-lived administrative sessions. This reduces the risk window for credential compromise and provides clear audit trails.

**Example: A PAM console requesting authentication to reset another user's MFA:**

~~~json
"client_context": [
  { "type": "app", "id": "admin_console" },
  { "type": "tenant", "id": "example.com" },
  {
    "type": "mission",
    "code": "reset_user_mfa",
    "task": "Reset MFA for alice@example.com (ticket IT-4821)",
    "allowed_actions": ["user.mfa.reset", "user.mfa.enroll"],
    "expires_at": "2026-03-10T18:00:00Z"
  }
]
~~~

The OP receiving this request might:

1. Require the user to satisfy a high-assurance authentication policy (e.g., phishing-resistant MFA) regardless of any existing session.
2. Display the mission details to the user for explicit confirmation.
3. Reduce the resulting session lifetime to align with `expires_at`.
4. Record the mission code and task details in its audit log.
5. Notify a security team of the elevated access event.

The Client then verifies the returned context before enabling the MFA reset action, ensuring the user cannot perform this action if the OP did not acknowledge the mission context.

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

# Relationship to Other Mechanisms

## Relationship to Authentication Context (`acr`)

The `client_context` parameter and the `acr` claim serve complementary but distinct roles. `client_context` is a *request input* describing the business context of the authentication. The `acr` claim is an *output* describing the authentication assurance that was actually satisfied.

The OP MAY use `client_context` as an input when determining the authentication assurance required. For example, an OP might define policy such as: "any request with a `mission` context whose `code` matches a high-privilege operation list requires `acr` `phishing-resistant-mfa`."

Clients SHOULD evaluate both the returned `client_context` and the `acr` claim when determining whether to permit privileged operations. A matching `client_context` with insufficient `acr` should not be sufficient to authorize a high-privilege action.

## Relationship to Rich Authorization Requests

OAuth 2.0 Rich Authorization Requests ({{!RFC9396}}) define the `authorization_details` parameter to convey fine-grained authorization semantics for access tokens.

The two parameters serve different layers of the protocol:

* `authorization_details` describes *what* the Client is authorized to do — the permissions attached to an access token used at a resource server.
* `client_context` describes *the context of the authentication event itself* — inputs to how the OP authenticates the user and what it includes in the ID Token.

A request MAY include both parameters. For example, an AI agent request might include `client_context` with a `mission` object (to drive authentication policy) and `authorization_details` with specific resource permissions (to scope the resulting access token).

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
  ...
  "client_context_types_supported": [
    "app",
    "tenant",
    "mission"
  ],
  "client_context_schemas": {
    "app": "https://op.example.com/schemas/context/app.json",
    "tenant": "https://op.example.com/schemas/context/tenant.json",
    "mission": "https://op.example.com/schemas/context/mission.json"
  },
  "client_context_par_required": true
}
~~~

# Client Registration Metadata

The following client registration metadata parameter is defined for use with the OpenID Connect Dynamic Client Registration Protocol {{!OIDC.Registration}} and equivalent static registration mechanisms.

`client_context_types`
: OPTIONAL. JSON array of strings listing the context type values that this client is permitted to include in `client_context` parameters. The OP SHOULD reject requests from this client that include context types not in this list.

`client_context_values`
: OPTIONAL. JSON object mapping context type values to arrays of permitted identifier values. For example, to restrict a client to specific application IDs or mission codes:

~~~json
{
  "client_context_types": ["app", "tenant", "mission"],
  "client_context_values": {
    "app": ["calendar", "email", "admin_console"],
    "mission": ["reset_user_mfa", "approve_change_request"]
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
: A context object contains one or more invalid field values (e.g., an `expires_at` in the past, an unrecognized `code` value for a constrained mission type).

**Example error response:**

~~~
HTTP/1.1 302 Found
Location: https://client.example.org/cb?
  error=invalid_client_context
  &error_description=Context+type+%22mission%22+is+not+permitted+for+this+client
  &state=af0ifjsldkj
~~~

For requests submitted via PAR, errors are returned in the PAR response:

~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "unsupported_client_context_type",
  "error_description": "Context type 'mission' is not supported by this server"
}
~~~

# Security Considerations

## Context Integrity

Because `client_context` influences authentication policy and the contents of ID Tokens, it is important to protect its integrity in transit.

Clients SHOULD transmit requests containing `client_context` using one of:

* **Pushed Authorization Requests ({{!RFC9126}})** — Context is submitted directly to the OP over an authenticated server-to-server TLS connection, preventing tampering by user agents or intermediaries. This is the RECOMMENDED mechanism.
* **Signed Request Objects** — When PAR is not available, Client SHOULD sign the Request Object containing `client_context` using the client's private key registered at the OP.

Clients MUST NOT transmit sensitive mission context (e.g., administrative workflow codes) as unprotected query parameters where it may be logged by servers, proxies, or browsers.

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

A compromised or malicious Client might attempt to request elevated context beyond what is appropriate — for example, requesting `mission` codes corresponding to highly privileged operations.

Authorization Servers SHOULD:

* Maintain an allow-list of permitted context values per client registration (`client_context_values`).
* Reject requests for context types or values not in the allow-list with `unsupported_client_context_type` or `invalid_client_context_value`.
* Log context requests for audit and anomaly detection.

## Mission Context Expiration

When `expires_at` is included in a `mission` context object, the OP SHOULD enforce that the resulting session lifetime and any issued tokens (including refresh tokens) do not outlive this value. Clients SHOULD request a new authentication for a new mission rather than reusing an expired-mission session.

## Confidentiality of Mission Context

Mission context MAY contain information that is sensitive from a business or operational security perspective — for example, the name of an administrative action being performed or the identity of a user being affected. Clients SHOULD ensure that mission context is not unnecessarily logged or exposed.

# Privacy Considerations

The `client_context` parameter MAY convey information about the End-User's activities (e.g., which application they are using, the administrative actions they are performing, or the tasks they have delegated to an AI agent). This information is transmitted to the OP and returned in the ID Token, potentially making it available to other parties.

Implementations SHOULD minimize the information included in `client_context` to what is necessary for authentication policy and audit purposes. Free-text fields such as `task` SHOULD NOT contain personally identifiable information beyond what is necessary.

OPs MUST handle `client_context` data in accordance with their privacy policy and applicable law. Context data SHOULD NOT be used for purposes beyond authentication policy evaluation and audit logging without the End-User's explicit consent.

# IANA Considerations

## OAuth Parameters Registry

This specification requests registration of the following parameter in the "OAuth Parameters" registry defined in {{!RFC6749}}.

* **Parameter name:** `client_context`
* **Parameter usage location:** authorization request
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
| `mission` | Identifies the operational task or workflow for which authentication is requested | This document |

Private-use context types MUST use URIs as their `type` value (e.g., `"https://example.com/context-types/custom"`). Short string type names (i.e., non-URI strings) are reserved for registered types.

## JWT Claims Registry

This specification requests registration of the following claim in the "JSON Web Token Claims" registry defined in {{!RFC7519}}.

* **Claim name:** `client_context`
* **Claim description:** Client context applied during authentication
* **Change controller:** IETF
* **Specification document(s):** This document

