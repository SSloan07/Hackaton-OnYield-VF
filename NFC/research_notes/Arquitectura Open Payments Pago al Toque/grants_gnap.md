# Open Payments authorization: GNAP grants (definitions, flows, limits, tokens) and how our implementation compares

Research date: 2026-10-16. Primary sources:
- openpayments.dev docs pages
- Open Payments OpenAPI specs (`interledger/open-payments-specifications`, `main` branch)
- RFC 9635 (GNAP)
- Rafiki auth/backend source (`interledger/rafiki`, `main` branch), which runs the Interledger test wallet

Note: `https://openpayments.dev/identity/grant-interaction/` and `https://openpayments.dev/apis/auth-server/` return 404. The current page list comes from https://openpayments.dev/sitemap-0.xml.

## 1. Official definitions: grant, grant request, access token, access rights, continuation, interact

### Takeaway
Open Payments uses GNAP (RFC 9635). A **grant** delegates authorization from a Resource Owner to client software. An **access token** carries the granted access rights (`type`, `actions`, `identifier`, `limits`). The `continue` object (`uri`, `access_token`, `wait`) is how the client finishes a pending (interactive) grant. In Open Payments, `access_token.manage` is a plain URI string, not GNAP's `manage` object.

### Cited Findings
- **Grant**: "a transfer, or delegation, of authorization from a Resource Owner (RO) to a piece of software". The RO can be a person (the end user) or a process (organizational rules). — [Grant negotiation and authorization](https://openpayments.dev/identity/grants/)
- **Grant** (concepts page): "an authorization issued by the RO" that lets a client access specific resources. It defines the allowed actions, may require user interaction, and may depend on the RO's consent. The AS validates the grant each time the client uses its access token. — [Authorization concepts](https://openpayments.dev/concepts/auth/)
- **Authorization**: "the process by which a client obtains permission from a resource owner to access and perform operations on protected resources". Open Payments uses GNAP to delegate a client's authority to use the Open Payments APIs. — [Authorization concepts](https://openpayments.dev/concepts/auth/)
- **Grant request**: the client "must send a grant request to the authorization server" before using the Open Payments APIs. The request names the resource type and the actions wanted on it. — [Grants](https://openpayments.dev/identity/grants/)
- **Access token**: "Access tokens represent a set of access rights and/or attributes granted to the client." A successful grant request returns one or more tokens. — [Grants](https://openpayments.dev/identity/grants/)
- **Access rights**: the set of rights and/or attributes granted to the client, which determine the operations it can perform. — [Authorization concepts](https://openpayments.dev/concepts/auth/)
- **An AS is identified by its grant endpoint URI**, which clients call to start a grant request. — [Grants](https://openpayments.dev/identity/grants/)
- **Access item schemas** (OpenAPI, auth-server.yaml):
  - **`access-incoming`**: `type: incoming-payment`. `actions` enum: `create`, `complete`, `read`, `read-all`, `list`, `list-all` (unique items). `identifier` (URI) is optional. Required fields: `type`, `actions`.
  - **`access-quote`**: `type: quote`. `actions` enum: `create`, `read`, `read-all`.
  - **`access-outgoing`**: `type: outgoing-payment`. `actions` enum: `create`, `read`, `read-all`, `list`, `list-all`. **`identifier` is required.** `limits` and `cardAuthorization` are optional.
  - Source: [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **Grant Request endpoint**: `POST /` (operationId `post-request`).
  - 200 is either Pending (`interact` + `continue`) or Approved (`access_token` + `continue`).
  - 400 `invalid_request` / `invalid_client`; 401 `invalid_client`; 500 `request_denied`.
  - Source: [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **`interact-request`** schema:
  - `start` is required: an array whose enum is only `redirect`.
  - `finish` is an object with `method` (enum: only `redirect`), `uri` and `nonce`. All three are required.
  - **`interact-response`**: `redirect` (URI) and `finish` (string, the AS's nonce). Both are required.
  - Source: [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **`continue`** schema: `access_token` (object with required `value`), `uri`, `wait`. `access_token` and `uri` are required. `wait` is the number of integer seconds the client must wait before calling the continuation URI. — [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **`access_token`** schema: `value` (opaque to the client), `manage` (URI), `expires_in` (seconds), `access`. `value`, `manage` and `access` are required. `additionalProperties: false`. — [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **RFC 9635 §3.1, `continue.uri`**: "The URI at which the client instance can make continuation requests… MUST be an absolute URI. The client instance MUST use this value exactly as given". REQUIRED. — [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635)
- **RFC 9635 §3.1, `continue.wait`**: "The amount of time in integer seconds the client instance MUST wait after receiving this request continuation response and calling the continuation URI. The value SHOULD NOT be less than five seconds, and omission of the value MUST be interpreted as five seconds." RECOMMENDED. — [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635)
- **RFC 9635 §3.1, `continue.access_token`**: the "continuation access token". It MUST be bound to the client's key, MUST NOT be a bearer token, MUST NOT have a `manage` field, and must be presented on all requests to the continuation URI. — [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635)
- **RFC 9635 §3.2.1, token fields**:
  - **`expires_in`**: "The number of seconds in which the access will expire. The client instance MUST NOT use the access token past this time… the access token MAY be revoked by the AS or RS at any point prior to its expiration." OPTIONAL.
  - **`access`**: "MUST reflect the rights associated with the issued access token. These rights MAY vary from what was requested".
  - **`manage`**: in GNAP this is an object (`uri` plus a token-management `access_token`).
  - Source: [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635)
- **Open Payments `manage` differs from GNAP**: `manage` is a URI string. The SDK passes it as `url: MANAGE_URL` together with the access token itself. — [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml); [Rotate token SDK](https://openpayments.dev/sdk/token-rotate/)
- **`interact.redirect`** is the URL the user visits to consent. **`interact.finish`** is the AS nonce used in the hash. — [Grant create outgoing SDK](https://openpayments.dev/sdk/grant-create-outgoing/); [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)

### Inferences
- `interact.finish` is a string field in the **response** (the AS's nonce) but an object in the **request** (`method`/`uri`/`nonce`). Store the response string; it is the second hash input.
- Because `access` "MAY vary from what was requested", our server should read the granted `access` from the continuation response, not assume it matches the request. This matters for actions and limits.

### Gaps
- The openpayments.dev "Grants" page does not define `identifier` or `limits` in prose. The definitions come only from the OpenAPI schema.

## 2. Interactive vs non-interactive grants per resource; `read-all`/`list-all` semantics

### Takeaway
- `incoming-payment` and `quote` grants are **non-interactive by default**.
- `outgoing-payment` grants are **always interactive**.
- Requesting `list-all` makes even an incoming-payment (or, per the docs, quote) grant require interaction, "because the client could list payments it did not create".
- Rafiki makes all of this configurable. The test wallet very likely uses Rafiki's defaults.

### Cited Findings
- **incoming-payment**: "Non-interactive by default." With `list-all`, interaction is required before a token is issued, because the client could list payments it did not create. One grant can create incoming payments across multiple accounts in the same ASE. Directed identity can be used. — [Authorization concepts](https://openpayments.dev/concepts/auth/)
- **quote**: "Non-interactive by default." The page says `list-all` requires interaction for the same reason. — [Authorization concepts](https://openpayments.dev/concepts/auth/)
- **outgoing-payment**: "Open Payments requires outgoing payment grant requests to be interactive." After a successful interaction the client must send a Grant Continuation request to get an access token. — [Authorization concepts](https://openpayments.dev/concepts/auth/)
- **Grants page paths**:
  - Path 1 has six steps: incoming-payment grant, then create IP, then quote grant, then create quote, then interactive outgoing-payment grant, then create OP.
  - Path 2 uses a single interactive grant that covers both `quote` and `outgoing-payment`.
  - Source: [Grants](https://openpayments.dev/identity/grants/)
- **Contradiction**: the spec's `access-quote` actions enum is only `create`, `read`, `read-all`. There is no `list` or `list-all` for quotes, which contradicts the concepts page's claim about quote `list-all`. — [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **Rafiki auth defaults**:
  - `INCOMING_PAYMENT_INTERACTION=false`, `QUOTE_INTERACTION=false`, `LIST_ALL_ACCESS_INTERACTION=true`
  - `WAIT_SECONDS=5`
  - `ACCESS_TOKEN_EXPIRY_SECONDS=600`, `INTERACTION_EXPIRY_SECONDS=600`
  - Source: [Rafiki auth config](https://raw.githubusercontent.com/interledger/rafiki/main/packages/auth/src/config/app.ts)
- **`read-all` / `list-all` in the spec**: they appear only as enum values; the auth spec gives no prose. The resource-server spec describes the list operations as "List all incoming payments on the wallet address" and "List all outgoing payments on the wallet address". — [resource-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/resource-server.yaml)
- **SDK outgoing-grant example** uses `actions: ['list', 'list-all', 'read', 'read-all', 'create']` with `identifier: walletAddress.id`. — [Grant create outgoing SDK](https://openpayments.dev/sdk/grant-create-outgoing/)

### Inferences
- `read`/`list` are scoped to resources this client created. `read-all`/`list-all` extend to resources on the wallet address that other clients created. This follows the docs' rationale ("could list payments it did not create"); the auth spec text does not state it formally.
- Our design is consistent with the docs:
  - non-interactive `incoming-payment` (create/read/list/complete) and `quote` (create/read)
  - interactive `outgoing-payment` plus an `incoming-payment` read/list/read-all/list-all item
- Bundling an `incoming-payment` `list-all` item into the interactive outgoing grant is fine. Requesting `list-all` in a *non-interactive* incoming-payment grant would fail on default Rafiki, or require `interact`.
- For our non-interactive `incoming-payment` grant: `complete` is a valid action in the spec enum, so that is fine. Check whether that grant needs an `identifier`. It is optional in the spec, and the docs say one grant can cover multiple accounts in the same ASE.

### Gaps
- We have no official statement on whether the Interledger test wallet overrides Rafiki's interaction defaults.

## 3. Interaction flow: start/finish methods, IdP, hash, `interact_ref`, rejection, polling

### Takeaway
Open Payments supports only `start: ["redirect"]` and `finish.method: "redirect"`; there is no `push` in the spec enum.

On approval, the AS (via the IdP) redirects to `finish.uri?hash=…&interact_ref=…`. The hash is SHA-256 over `clientNonce\nasNonce\ninteract_ref\ngrantEndpointURI` (ASCII, no trailing newline).

**Encoding discrepancy:**
- RFC 9635 says **URL-safe base64, no padding**.
- The openpayments.dev page says "Base64 with no padding".
- The docs' code sample and Rafiki itself use Node `digest('base64')`, which is **standard base64 WITH padding**.

On rejection, Rafiki redirects with `result=grant_rejected` and sends no hash.

### Cited Findings
- **Flow per the IdP page's sequence diagram:**
  1. The client POSTs a grant request with an `interact` object.
  2. The AS returns the interact redirect URI and the continue URI.
  3. The client opens the redirect URI.
  4. The AS starts the interaction and sets a session.
  5. The AS sends a "302 temporary redirect to identity provider" with grant info in the query string.
  6. The RO accepts at the IdP.
  7. The IdP sends the choice to the AS, which replies 202.
  8. The IdP asks to finish, and the AS ends the session.
  9. The AS sends a 302 to the finish URI from the original request. It is "secured with unique hash" and has `interact_ref` in the query string.
  10. The client verifies the hash.
  11. The client POSTs to the continue URI with `interact_ref` in the body.
  12. The AS returns 200 with the access token.

  Source: [Identity providers](https://openpayments.dev/identity/idp/)
- **IdP definition**: "a system or service that manages user authentication and consent". Authorization servers that issue interactive grants must integrate with an IdP. — [Identity providers](https://openpayments.dev/identity/idp/)
- **ASE-side IdP implementation:**
  - The AS passes the IdP: grant ID, RO identity, requested access, client display name, and a return path.
  - The IdP reports the decision through a signed redirect or a back-channel API.
  - On accept, the AS sets the grant to `approved`, then redirects to `interact.finish` URI (if one was given) with `interact_ref` and `hash`.
  - On deny, the grant "must be moved to a terminal denied state so that subsequent continuation requests return the appropriate error."
  - The diagram has 16 steps, the same flow at finer granularity.
  - Source: [Implement identity provider](https://openpayments.dev/implement/identity-provider/)
- **Open Payments hash spec:**
  - Inputs in order, separated by a single `\n`:
    1. the client's `nonce` from the initial request
    2. the AS's `nonce` returned in the initial grant response
    3. the `interact_ref` from the finish redirect
    4. the grant endpoint URI used for the initial request
  - No padding, leading/trailing whitespace, or trailing newline.
  - "sha-256 … the only hashing algorithm currently supported by Open Payments."
  - "The byte array from the hash function is then encoded using Base64 with no padding."
  - The client **must** verify the hash.
  - Example code: `createHash('sha-256').update(data).digest('base64')`
  - Source: [Hash verification](https://openpayments.dev/identity/hash-verification/)
- **RFC 9635 §4.2.3 (normative GNAP)**:
  - Same four inputs and order, newline 0x0A, ASCII bytes.
  - "The resulting byte array from the hash function is then encoded using **URL-Safe base64 with no padding**".
  - `hash_method` defaults to `sha-256`.
  - "The AS MUST always provide this hash, and the client instance MUST validate the hash when received."
  - Worked example: `x-gguKWTj8rQf7d7i3w3UhzvuJ5bpOlKyAlVpLxBffY`
  - Source: [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635)
- **Rafiki's actual implementation**:
  - `data = \`${clientNonce}\n${interactNonce}\n${interactRef}\n${grantRequestUrl}\``
  - `grantRequestUrl = ensureTrailingSlash(config.authServerUrl) + grant.tenantId`
  - `crypto.createHash('sha-256').update(data).digest('base64')` (standard, padded base64)
  - Sets `hash` and `interact_ref` query params, then redirects.
  - On deny: `grantService.finalize(..., Rejected)` and `result=grant_rejected`.
  - If the interaction is neither approved nor denied: `result=grant_invalid`.
  - Grants without a finish URI ("unfinishable") just get a 202 at the IdP finish step.
  - Source: [Rafiki interaction routes](https://raw.githubusercontent.com/interledger/rafiki/main/packages/auth/src/interaction/routes.ts)
- **Local check**: Node `createHash('sha-256')` over the RFC example base string gives `x+gguKWTj8rQf7d7i3w3UhzvuJ5bpOlKyAlVpLxBffY=`. That is the same digest as the RFC value but in standard alphabet with padding. It confirms that Node accepts `'sha-256'` and that the Open Payments/Rafiki encoding differs from the RFC's. — computed locally against [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635) example
- **RFC 9635 §4.2.1**: the redirect finish adds `hash` (REQUIRED) and `interact_ref` (REQUIRED) as query parameters to the client's redirect URI. — [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635)
- **Continuation request:**
  - Endpoint: `POST /continue/{id}`. Body: optional `interact_ref` ("The interaction reference generated for this interaction by the AS").
  - Responses: 200 (`continue` + `access_token`/`subject`); 400 `too_fast`/`invalid_client`; 401 `invalid_client`/`invalid_continuation`/`request_denied`; 404 `invalid_continuation`/`invalid_request`.
  - The example "Continuing during pending interaction" returns only `continue` with `wait: 30`.
  - Source: [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **Polling (RFC 9635 §5)**: "If a wait parameter was included … the client instance MUST NOT call the continuation URI prior to waiting the number of seconds indicated. If no wait period is indicated, the client instance MUST NOT poll immediately and SHOULD wait at least 5 seconds. If the client instance does not respect the given wait period, the AS MUST return the too_fast error". — [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635)
- **Polling (Open Payments docs)**: when no UI is available, "consider implementing a polling mechanism to check that the interaction has completed". There is no further detail. — [Grant continue SDK](https://openpayments.dev/sdk/grant-continue/); [Future payments guide](https://openpayments.dev/guides/outgoing-grant-future-payments/)
- **SDK continuation call**: `client.grant.continue({ accessToken: CONTINUE_ACCESS_TOKEN, url: CONTINUE_URI }, { interact_ref })`, then `isFinalizedGrantWithAccessToken(grant)`. The result carries `access_token.value` and `access_token.manage`. — [Grant continue SDK](https://openpayments.dev/sdk/grant-continue/)
- **The continuation response also returns a new `continue` object**, alongside `access_token` (`value`, `manage`, `access`). — [Future payments guide](https://openpayments.dev/guides/outgoing-grant-future-payments/)

### Inferences (implementation validation flags)
- **FLAG (hash encoding)**: compute with `digest('base64')`, exactly as Rafiki and the docs' sample do.
  - Do NOT use `base64url`, despite RFC 9635.
  - Do NOT strip the `=` padding, despite the docs' prose saying "no padding". Rafiki's output keeps the padding.
  - For robustness, compare after normalizing (map `-_` to `+/` and strip `=`), or compute both encodings. Use a constant-time compare.
- **FLAG (URL decoding of `hash`)**: standard base64 contains `+` and `/`. Rafiki sets the param with `URL.searchParams.set`, which percent-encodes `+` as `%2B`. Read the param with `URLSearchParams` or Express `req.query`, which decode correctly. Never hand-parse it in a way that turns `+` into a space.
- **FLAG (4th hash input)**: Rafiki uses `authServerUrl` with a trailing slash plus `tenantId`, i.e. the multi-tenant grant endpoint. Our 4th input must be exactly the URL we POSTed the grant request to, normally `walletAddress.authServer` verbatim. A trailing-slash mismatch will break verification. If verification fails, log both strings.
- **FLAG (rejection handling)**: the finish URI handler must handle `?result=grant_rejected` (and Rafiki's `result=grant_invalid`), where no `hash`/`interact_ref` are present. Do not call continue in that case. Treat the grant as finalized/rejected.
- **FLAG (wait)**: if we ever poll the continue URI without `interact_ref`, wait at least `continue.wait` seconds (Rafiki default 5; the spec example is 30), or the AS returns 400 `too_fast`.
- **FLAG (store the new `continue`)**: after a successful continue, store the returned `continue` (uri + access_token). It is needed for `grant.cancel` (DELETE /continue/{id}) later.
- `finish.method: "redirect"` and `start: ["redirect"]` are the only options in the Open Payments spec, so our request is correct. GNAP's `push` finish method is not in the Open Payments enum.

### Gaps
- No openpayments.dev page documents `result=grant_rejected`; the evidence is Rafiki source only.
- No openpayments.dev sequence diagram shows the rejection or polling paths.
- The docs' prose ("no padding") contradicts their own code sample and Rafiki ("base64" with padding). We found no official erratum.

## 4. Limits (`debitAmount`, `receiveAmount`, `interval`) and spent amounts

### Takeaway
`limits` on an `outgoing-payment` access item caps cumulative spend *per interval*:
- `debitAmount` is the amount debited from the sender.
- `receiveAmount` is the amount the receiver gets.
- An optional `receiver` restricts the grant to one incoming payment.

`interval` is an ISO 8601 repeating interval. Amounts reset each interval, and unspent amounts do not roll over.

`GET /outgoing-payment-grant` on the resource server returns `spentDebitAmount`/`spentReceiveAmount` for the current interval. Either can be `null`.

### Cited Findings
- **`limits-outgoing`**: "Open Payments specific property that defines the limits under which outgoing payments can be created." It is `anyOf` three variants:
  - no-amount (`receiver` + `interval`)
  - debit-amount (requires `debitAmount`)
  - receive-amount (requires `receiveAmount`)

  Amounts are maxima per interval, so multiple payments are allowed while the total stays within the limit. — [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **`interval`** schema: "[ISO8601 repeating interval]". Examples: `R11/2022-08-24T14:15:22Z/P1M`, `R/2017-03-01T13:00:00Z/2018-05-11T15:30:00Z`, `R-1/P1Y2M10DT2H30M/2022-05-11T15:30:00Z`. The spec's grant example uses `interval: 'R12/2019-08-24T14:15:22Z/P1M'` with `receiver` and `debitAmount`. — [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **SDK interval example**: `interval: 'R/2016-08-24T08:00:00Z/P1D'` in the outgoing grant `limits`. Other language snippets show `limits.receiver` (an incoming payment URL) and `limits.receiveAmount`. — [Grant create outgoing SDK](https://openpayments.dev/sdk/grant-create-outgoing/)
- **Future payments guide:**
  - `R3/2025-05-20T13:00:00Z/P1M` means three monthly windows: 20 May 13:00 UTC to 20 June 12:59 UTC, and so on.
  - "The amount resets each month and any unspent portions don't roll over."
  - The example `debitAmount` is `{ assetCode: 'CAD', assetScale: 2, value: '10000' }`.
  - The grant is requested with `read` and `create`, and no recipient is known in advance.
  - Source: [Future payments guide](https://openpayments.dev/guides/outgoing-grant-future-payments/)
- **Spent amounts endpoint**: `GET /outgoing-payment-grant` (operationId `get-outgoing-payment-grant`).
  - "Returns the spent amounts for the current outgoing payment grant corresponding to the presented GNAP access token, if any. If a grant was created with an interval (recurring), the amounts returned are for the current interval."
  - Required fields `spentReceiveAmount` ("total amount successfully received (by all receivers) using the current outgoing payment grant") and `spentDebitAmount` ("total amount successfully deducted from the sender's account"). Each is an `amount` or `null`.
  - There is an example "No spent amounts" with both `null`. Errors: 401, 403.
  - Source: [resource-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/resource-server.yaml)
- **SDK spent-amounts call**: `client.outgoingPayment.getGrantSpentAmounts({ url: walletAddress.resourceServer, accessToken })`, where `accessToken` is the outgoing-payment access token from continuation. — [Spent amounts SDK](https://openpayments.dev/sdk/outgoing-grant-spent-amounts/)

### Inferences
- Our `limits: { debitAmount, interval: "R/<start>/P1W" }` is valid. `R` with no count means unbounded weekly repetitions starting at `<start>`.
  - Make `<start>` a full UTC timestamp (e.g. `2026-10-16T00:00:00Z`).
  - The `debitAmount` `assetCode`/`assetScale` must match the payer wallet's asset.
- **FLAG**: handle `null` for both spent fields. A freshly issued grant, or a new interval, has none spent.
- **FLAG**: the values are per *current interval*. Budget left = `debitAmount.value - spentDebitAmount.value` for this week only, not lifetime.
- **FLAG**: call it with the *outgoing-payment* access token, against `walletAddress.resourceServer`, not the auth server. A rotated token stays tied to the same grant.

### Gaps
- The docs do not say how a start date in the future or the past is treated, or how a payment that would exceed the remaining limit fails. The OP-creation error code was not researched here.
- The docs do not say whether `debitAmount` and `receiveAmount` can both be present. The spec uses `anyOf`, which permits both, but Rafiki's behavior was not verified.

## 5. Token management: rotate, revoke, `expires_in`, manage URL; grant cancel

### Takeaway
Rotate with `POST <manage URL>` (`client.token.rotate({ url: manage, accessToken })`). It returns a new `access_token` (new `value` and new `manage`) with the same rights; the old value is invalidated. Revoke a token with `DELETE <manage URL>`. Cancel a whole grant with `DELETE <continue.uri>` using the continuation token. Default token lifetime is 10 minutes, including for rotated tokens.

### Cited Findings
- **Rotate (SDK)**: `client.token.rotate({ url: MANAGE_URL, accessToken: ACCESS_TOKEN })` returns `token.access_token.value` and `token.access_token.manage`. "If an access token expires, a client can request that it be rotated." The new token "replaces the existing token and has the same rights and properties." "All access tokens have a default 10-minute lifespan", including rotated ones. — [Rotate token SDK](https://openpayments.dev/sdk/token-rotate/)
- **Rotate (spec)**: `POST /token/{id}` "Management endpoint to rotate access token." 200 returns a new `access_token`; 400/404 `invalid_rotation`; 401 `invalid_client`; 500 `request_denied`. — [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **RFC 9635 §6.1**: rotation issues "a new access token in place of an existing access token, with the same rights and properties as the original token, apart from an updated token value and expiration time". It is a POST with no body, signed, with the token in the Authorization header. Rotation cannot alter rights. The AS "MUST invalidate the current access token value associated with this URI, if possible". — [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635)
- **Revoke token**: `client.token.revoke({ url: MANAGE_URL, accessToken: ACCESS_TOKEN })` asks that the token be "invalidated for all purposes". Spec: `DELETE /token/{id}` returns 204; 401 `invalid_client`; 500 `request_denied`. — [Revoke token SDK](https://openpayments.dev/sdk/token-revoke/); [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **Cancel grant**: `client.grant.cancel({ accessToken: CONTINUE_ACCESS_TOKEN, url: CONTINUE_URI })`. The grant "is placed into a finalized state", and "no further updates to the grant request are allowed". Spec: `DELETE /continue/{id}`, "Cancel a grant request or delete a grant client side"; 204, 401, 404. — [Grant revoke SDK](https://openpayments.dev/sdk/grant-revoke/); [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **`expires_in`**: seconds until expiry; the client "MUST NOT use the access token past this time". — [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635); [auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)
- **Rafiki resource server ("Inactive Token")**: the RS introspects the token with the AS.
  - Introspection failure gives `401 'Invalid Token'`.
  - Not active gives `403 'Inactive Token'`.
  - Empty access gives `403 'Insufficient Grant'`.
  - Source: [Rafiki backend auth middleware](https://raw.githubusercontent.com/interledger/rafiki/main/packages/backend/src/open_payments/auth/middleware.ts)
- **Rafiki default lifetime**: `ACCESS_TOKEN_EXPIRY_SECONDS` is 600. — [Rafiki auth config](https://raw.githubusercontent.com/interledger/rafiki/main/packages/auth/src/config/app.ts)

### Inferences (implementation validation flags)
- Our approach (on 403 "Inactive Token", call `token.rotate` with the manage URL) matches the docs. Better practice is to refresh proactively using `expires_in` (default ~600 s) instead of waiting for a 403.
- **FLAG**: "Inactive Token" also occurs when the token or grant was **revoked** (user revoked consent in the wallet, `grant.cancel`, `token.revoke`), not only on expiry. In that case rotate fails (`invalid_rotation` / 401). Our code must then fall back to a new interactive grant and must not retry rotate in a loop.
- **FLAG**: after rotating, persist BOTH the new `access_token.value` AND the new `access_token.manage`. The old manage URL and old value become invalid. Rotate once with a mutex/single-flight so concurrent requests don't race and invalidate each other's tokens.
- Use `grant.cancel` (continue URI plus continuation token) to end the outgoing-payment grant when the user unlinks. Use `token.revoke` only to kill a specific token.

### Gaps
- The docs do not say whether Rafiki allows rotating an *expired* token indefinitely, or whether the grant itself has an absolute expiry. The SDK page implies expired tokens can be rotated, and the test wallet's 403-then-rotate behavior suggests the same. We found no explicit maximum.

## 6. Official sequence diagrams of the grant flow

### Takeaway
openpayments.dev has two Mermaid-style sequence diagrams of the interactive grant: a 14-step one on the "Identity providers" page and a 16-step one on the "Implement identity provider" page, both covering the approval path only. The "Grants" page gives numbered step lists (Path 1 and Path 2) rather than diagrams.

### Cited Findings
- **"Identity providers" diagram**: participants are Client, AS, IdP and RO. It shows the steps listed in section 3 (grant request with `interact`, the 302 to the IdP, the 302 back to `finish.uri` with hash + `interact_ref`, client hash verification, then continuation POST and 200 with token). There is no reject path. — [Identity providers](https://openpayments.dev/identity/idp/)
- **"Implement identity provider" diagram**: the same flow in 16 steps, with explicit "AS returns 202 accepting the choice" and "AS ends session" steps. — [Implement identity provider](https://openpayments.dev/implement/identity-provider/)
- **Overall Open Payments flow** (IP grant, IP, quote grant, quote, interactive OP grant, continuation, OP) is described as Path 1/Path 2 lists. A separate flow page exists that we did not fetch. — [Grants](https://openpayments.dev/identity/grants/); [OP flow](https://openpayments.dev/concepts/op-flow/)
- **Future payments guide**: a step list (wallet address, interactive grant, redirect, finish, continue, spent amounts). — [Future payments guide](https://openpayments.dev/guides/outgoing-grant-future-payments/)

### Inferences
- For our diagram, combine:
  - Path 1: non-interactive IP grant, then IP, then non-interactive quote grant, then quote, then interactive OP grant.
  - The IdP diagram for the interactive part.
  - Our additions: the `result=grant_rejected` branch, the 403 "Inactive Token" branch to rotate (with a fallback to a new grant), and `getGrantSpentAmounts`.

### Gaps
- The text of https://openpayments.dev/concepts/op-flow/ was not fetched. Check it for an end-to-end diagram that includes the grant steps.
