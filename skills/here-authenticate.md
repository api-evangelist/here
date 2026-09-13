---
name: here-authenticate
description: Choose and obtain HERE platform credentials — API key, OAuth 2.0 client_credentials bearer token, or OpenID Connect user sign-in — and know which HERE APIs accept which.
api: HERE Identity and Access Management
specs:
  - openapi/here-oauth2-token-v1-openapi.yml
  - openapi/here-authentication-v1-1-openapi.yml
  - openapi/here-authorization-v1-1-openapi.yml
operations:
  - OAuth 2.0 Access Token getOAuth2AccessToken
  - OAuth 2.0 Access Token getAuthServerMetadata
  - OAuth 2.0 Access Token getMcpAuthServerMetadata
  - OAuth 2.0 Access Token RegisterMcpClient
  - OAuth 2.0 Access Token deleteDeviceToken
---

# Authenticate against HERE

## Three schemes, three jobs

| Scheme | How | Use it for |
|---|---|---|
| `ApiKey` | `?apiKey=<key>` query parameter (some services also accept an `x-api-key` header) | Query APIs called from your own backend. Simplest thing that works. |
| `Bearer` (OAuth 2.0) | `Authorization: Bearer <token>` from `POST https://account.api.here.com/oauth2/token` | Server-to-server, rotating credentials, and everything on the HERE platform (Data API, IAM, Workspace). |
| OpenID Connect | authorization code + PKCE at `https://account.here.com` | Signing a human in. |

Almost every HERE service declares `ApiKey` and `Bearer` as alternatives on every operation, so you can
start with a key and move to tokens without changing call sites.

## Get a bearer token

`POST https://account.api.here.com/oauth2/token` (`OAuth 2.0 Access Token getOAuth2AccessToken`).

The discovery document at `https://account.api.here.com/.well-known/oauth-authorization-server` states the
terms exactly:

- `grant_types_supported`: `client_credentials`
- `token_endpoint_auth_methods_supported`: `private_key_jwt`
- `token_endpoint_auth_signing_alg_values_supported`: `RS256`

HERE's older flow signs the token request with **OAuth 1.0a HMAC-SHA256** using the access key id and
secret — the spec says so in its own `securitySchemes` description, and the IAM error registry carries the
OAuth 1.0 failure codes (`401202` malformed header, `401204` timestamp outside the valid period,
`401205` unsupported signature method, `401207` nonce already consumed). Use HERE's own client rather than
hand-rolling the signature: `com.here.account:here-oauth-client` (Maven, 0.4.28) or
`@here/olp-sdk-authentication` (npm, 3.0.0).

## Sign a user in

`https://account.here.com/.well-known/openid-configuration` declares:

- `authorization_endpoint` `https://account.here.com/authorize`, `token_endpoint` `https://account.here.com/token`
- `jwks_uri` `https://account.here.com/openid/jwk`, `id_token_signing_alg_values_supported` `RS256`
- `scopes_supported`: `openid`, `profile`, `email`, `phone`, `readwrite:ha`
- `code_challenge_methods_supported`: `S256` — use PKCE

## If you are an agent

HERE runs a **separate MCP authorization server**:
`https://account.api.here.com/.well-known/oauth-authorization-server/mcp`

- issuer `https://account.here.com/mcp`
- `authorization_code` grant with PKCE `S256`
- **RFC 7591 dynamic client registration** at `https://account.api.here.com/mcp/register`
  (`OAuth 2.0 Access Token RegisterMcpClient`), with `token_endpoint_auth_methods_supported: ["none"]`

The public documentation MCP server at `https://docs.here.com/mcp` is anonymous and needs none of this.
The registration endpoint exists for an authenticated MCP surface; register dynamically rather than
asking a human for a client id.

## Revoking

`DELETE /tokens` (`OAuth 2.0 Access Token deleteDeviceToken`) revokes a device access token.
Application credentials, API keys and access keys are deleted through the Authentication API v1.1
(`deleteAPIKey`, `deleteAccessKey`, `deleteApplication`). None of these deletions has a stated undo window —
they are immediate and terminal.

## Errors

`errorResponse.errorCode` with detail in `errorFields`. `400201` missing required field, `401200`
Authorization header missing, `401300` invalid client credentials, `403403` credentials do not authorize
this operation, `429002` too many requests.
Full registry: https://docs.here.com/identity-and-access-management/docs/error-messages
