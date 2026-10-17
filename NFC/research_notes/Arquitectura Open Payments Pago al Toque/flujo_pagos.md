# Open Payments: payment resources and end-to-end payment flow (incoming payment, quote, outgoing payment)

Research date: 2026-10-16. Main primary sources:
- Docs site: https://openpayments.dev (pages: `/concepts/op-flow/`, `/concepts/resources/`, `/guides/*`, `/sdk/*`, `/apis/`)
- Docs source repo (the same MDX that renders openpayments.dev): https://github.com/interledger/open-payments (`docs/src/content/docs/...`, Node snippets in `snippets/node/...`)
- OpenAPI specs, **version 1.4.0** (the `VERSION` file in the specs repo): https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml and https://github.com/interledger/open-payments-specifications/blob/main/openapi/auth-server.yaml (the open-payments repo pulls these in as the `open-payments-specifications` git submodule)
- Rafiki (reference ASE implementation): https://rafiki.dev/integration/requirements/webhook-events/ and the GraphQL schema https://github.com/interledger/rafiki/blob/main/packages/backend/src/graphql/schema.graphql

Note: the API reference pages at `openpayments.dev/apis/...` are rendered client-side and redirect (for example `/apis/resource-server/operations/create-outgoing-payment/` goes to `/apis/#tag/outgoing-payment/POST/outgoing-payments`). The field definitions below were therefore taken verbatim from the OpenAPI YAML that feeds that reference.

## Q1. Definitions, fields and lifecycle of incoming payment, quote and outgoing payment (and can an outgoing payment be created without a quote?)

### Takeaway
An incoming payment is created on the **recipient's** resource server. A quote and an outgoing payment are created on the **sender's** resource server. The spec has no explicit "state" field. The lifecycle is expressed as `completed` + `receivedAmount` + `expiresAt` for incoming payments, `expiresAt` for quotes, and `failed` + `sentAmount` for outgoing payments. Rafiki keeps internal state enums (PENDING/PROCESSING/COMPLETED/EXPIRED and FUNDING/SENDING/COMPLETED/FAILED/CANCELLED), but the Open Payments API does not return them. **Yes**: in spec 1.4.0 an outgoing payment can be created without a quote, using `incomingPayment` + `debitAmount`.

### Cited Findings

**Incoming payment**
- Definition: "An **incoming payment** resource represents a payment that will be, is currently being, or has been received by the account." — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- "An `incoming-payment` resource is often the first resource created in a payment flow." — [openpayments.dev/concepts/resources](https://openpayments.dev/concepts/resources/)
- "A client MUST create an **incoming payment** resource before it is possible to send any payments to the wallet address." When it is created, "the receiving Account Servicing Entity generates unique payment details that can be used to address payments to the account". "All of the input parameters are _optional_." — [resource-server.yaml, POST /incoming-payments](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- Create request (`create-incoming-payment-request`, `additionalProperties: false`): `walletAddress` (required), `incomingAmount` ("The maximum amount that should be paid into the wallet address under this incoming payment."), `expiresAt` ("The date and time when payments into the incoming payment must no longer be accepted.", date-time, writeOnly), `metadata` ("Additional metadata associated with the incoming payment. (Optional)"). "The `incomingAmount` must use the same `assetCode` and `assetScale` as the wallet address." — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- Resource fields: `id` ("The URL identifying the incoming payment."), `walletAddress`, `completed` ("Describes whether the incoming payment has completed receiving fund.", default false), `incomingAmount`, `receivedAmount` ("The total amount that has been paid into the wallet address under this incoming payment."), `expiresAt` ("The date and time when payments under this incoming payment will no longer be accepted."), `metadata`, `createdAt`. Required: `id, walletAddress, completed, receivedAmount, createdAt`. The create/GET response (`incoming-payment-with-methods`) adds `methods`, for example `{type: ilp, ilpAddress, sharedSecret}`. — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- Automatic completion rule: "If `incomingAmount` is specified and the total received using the payment details equals or exceeds the specified `incomingAmount`, then the receiving Account Servicing Entity MUST reject any further payments and set `completed` to `true`." — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- Expiry rule: "If an `expiresAt` value is defined, and the current date and time on the receiving Account Servicing Entity's systems exceeds that value, the receiving Account Servicing Entity MUST reject any further payments." — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- `metadata` can hold an external reference for reconciliation: "the client could use the `metadata` property to store an external reference on the **incoming payment** and this can be shared with the account holder to assist with reconciliation." (Example: `metadata: {externalRef: INV2022-02-0137}`.) — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- Without `incomingAmount`, the recipient's ASE cannot tell how much to expect, so the flow ends either by completion (the client calls Complete) or by expiry ("the payment session will eventually expire"). — [openpayments.dev/concepts/resources](https://openpayments.dev/concepts/resources/)
- The official SDK snippet for creating an incoming payment uses `incomingAmount` and `expiresAt: new Date(Date.now() + 60_000 * 10).toISOString()` (10 minutes). — [snippets/node/incoming-payment/incoming-payment-create.ts](https://github.com/interledger/open-payments/tree/main/snippets/node)
- Internal Rafiki states (not exposed in Open Payments): `PENDING` ("initially created and has not started processing"), `PROCESSING` ("after funds have started clearing into the account"), `COMPLETED` ("completed automatically once the expected `incomingAmount` is received or manually via an API call"), `EXPIRED` ("expired before completion, and no further funds will be accepted"). — [Rafiki schema.graphql](https://github.com/interledger/rafiki/blob/main/packages/backend/src/graphql/schema.graphql)

**Quote**
- Definition: "A **quote** resource represents the quoted amount details with which an Outgoing Payment may be created." / "A **quote** is a sub-resource of a wallet address. It represents a quote for a payment from the wallet address." — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- A quote "indicates how much it will cost, including any applicable fees, to make the payment"; "Quotes are only valid for a limited time." — [openpayments.dev/concepts/resources](https://openpayments.dev/concepts/resources/)
- Fields: `id`, `walletAddress` ("The URL of the wallet address from which this quote's payment would be sent."), `receiver` ("The URL of the incoming payment that the quote is created for."; it must match the pattern `^(https|http)://(.+)/incoming-payments/(.+)$`), `receiveAmount`, `debitAmount` ("The total amount that should be deducted from the sender's account when the corresponding outgoing payment has been paid."), `method` (`ilp`), `expiresAt` ("The date and time when the calculated `debitAmount` is no longer valid.", readOnly), `createdAt`. Required: `id, walletAddress, receiver, receiveAmount, debitAmount, createdAt, method`. — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- The three request variants (`oneOf`) are "Receiver with incoming amount" (`walletAddress, receiver, method`), "With fixed-receive amount" (adds `receiveAmount`) and "With fixed-send amount" (adds `debitAmount`). "The quote must be created with a (`debitAmount` xor `receiveAmount`) unless the `receiver` is an Incoming Payment which has an `incomingAmount`." A 400 means "No amount was provided and no amount could be inferred from the receiver." — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- Guide note: "Quote responses include an `expiresAt` timestamp. Create the outgoing payment before the quote expires. If creation fails because the quote expired, request a new quote and try again." The guide's example quote has `createdAt 00:00:51.50Z` and `expiresAt 00:02:51.50Z`, about 2 minutes. — [guides/recurring-subscription-incoming-amount](https://openpayments.dev/guides/recurring-subscription-incoming-amount/)
- When the receiver has an `incomingAmount`: "`debitAmount` - The amount ... that will be charged to the customer"; "`receiveAmount` - The `incomingAmount` value from the incoming payment resource". — [guides/accept-otp-online-purchase](https://openpayments.dev/guides/accept-otp-online-purchase/)

**Outgoing payment**
- Definition: "An **outgoing payment** resource represents a payment that will be, is currently being, or has previously been, sent from the wallet address." "Once created, it is already authorized and SHOULD be processed immediately. If payment fails, the Account Servicing Entity must mark the **outgoing payment** as `failed`." — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- "The purpose of this resource is to serve as an instruction to the sender's ASE to make a payment." "Open Payments requires explicit consent from the sender before the outgoing payment resource is created." — [openpayments.dev/concepts/resources](https://openpayments.dev/concepts/resources/)
- Fields: `id`, `walletAddress` ("from which this payment is sent"), `quoteId` ("The URL of the quote defining this payment's amounts.", readOnly), `failed` ("Describes whether the payment failed to send its full amount.", default false), `receiver` ("The URL of the incoming payment that is being paid."), `receiveAmount`, `debitAmount` ("The total amount that should be deducted from the sender's account when this outgoing payment has been paid."), `sentAmount` ("The total amount that has been sent under this outgoing payment."), `metadata`, `createdAt`. Required: `id, walletAddress, receiver, receiveAmount, debitAmount, sentAmount, createdAt`. — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- The POST response schema `outgoing-payment-with-spent-amounts` adds `grantSpentDebitAmount` ("The total amount successfully deducted from the sender's account using the current outgoing payment grant.") and `grantSpentReceiveAmount` ("The total amount successfully received (by all receivers) using the current outgoing payment grant."). The GET schema (`outgoing-payment`, `additionalProperties: false`) does not include them. — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- **Without a quote (yes, in spec 1.4.0):** "Either provide a `quoteId` to create an outgoing payment based on a quote or provide `incomingPayment` and `debitAmount` to create an outgoing payment directly from an incoming payment." Schema `create-outgoing-payment-request-from-incoming-payment` requires `incomingPayment` ("The URL of the incoming payment this outgoing payment will fulfill."), `debitAmount` and `walletAddress`. Schema `...-from-quote` requires `quoteId` and `walletAddress`. "The `debitAmount` must use the same `assetCode` and `assetScale` as the wallet address." — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml); the concepts page agrees: "A quote isn't required for all outgoing payments. The resource can instead contain `incomingPayment` and `debitAmount` values." — [openpayments.dev/concepts/resources](https://openpayments.dev/concepts/resources/)
- Internal Rafiki states: `FUNDING` ("reserving funds and will transition to `SENDING` once funds are secured"), `SENDING`, `COMPLETED`, `FAILED`, `CANCELLED`. — [Rafiki schema.graphql](https://github.com/interledger/rafiki/blob/main/packages/backend/src/graphql/schema.graphql)

**Amount format**
- `amount = {value, assetCode, assetScale}`. `value` is "an unsigned 64-bit integer amount, represented as a string"; for example "An integer amount of `1000` with an `assetCode` of `USD` and `assetScale` of `2` translates to $10.00." — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)

### Inferences
- For Pago al Toque, the 60 s `expiresAt` on the incoming payment is a hard deadline. After it passes, the merchant's ASE "MUST reject any further payments", so the outgoing payment created on the second tap has to finish sending before then, not just be created. The quote also has its own `expiresAt`, which may be shorter or longer. The server should check both before the second tap.
- The from-incoming-payment variant could skip the quote, but then the client does not know the fees in advance. The fixed `debitAmount` is debited and the `receiveAmount` may come out lower than `incomingAmount`. To check "debitAmount vs. weekly limit" before charging, the quote is what gives the real `debitAmount`.
- When the receiver has an `incomingAmount`, the quote must be created without an amount (variant "Receiver with incoming amount"). That matches Pago al Toque's design (`method: "ilp"`, `receiver` = incoming payment URL).

### Gaps
- The spec defines no default quote lifetime; it depends on the ASE (the guide example shows about 2 min). I did not verify the default value in Rafiki or the Interledger Test Wallet.
- Lifecycle states are not exposed in the Open Payments API, so the client has to infer state from `completed`, `failed`, `receivedAmount`, `sentAmount` and `expiresAt`.

## Q2. Official sequence of the Open Payments flow: which wallet / AS / RS is contacted at each step

### Takeaway
The canonical order is: (1) GET wallet address(es); (2) **non-interactive** grant `incoming-payment` on the **recipient's** AS, then POST incoming payment on the **recipient's** RS; (3) **non-interactive** grant `quote` on the **sender's** AS, then POST quote on the **sender's** RS; (4) **interactive** grant `outgoing-payment` on the **sender's** AS (redirect, IdP consent, finish, continuation with `interact_ref`), then POST outgoing payment on the **sender's** RS; (5) optionally GET/list for history.

### Cited Findings
- Steps from the Open Payments flow page ([openpayments.dev/concepts/op-flow](https://openpayments.dev/concepts/op-flow/)):
  1. *Get account details*: Client sends GET to the recipient's wallet address URL; 200 with public details "including asset code, scale, and the authorization and resource server URLs". No grant needed.
  2. *Incoming payment*: Client sends POST grant request (`incoming-payment`) to the **recipient's Authorization server**, gets 200 with access token; then POST to the **recipient's Resource server**, gets 201 with the incoming payment. Grant: **non-interactive**.
  3. *Quote*: Client sends POST grant request (`quote`) to the **sender's Authorization server**, gets 200 with token; POST to the **sender's Resource server**, gets 201 with the quote `id` and cost. Grant: **non-interactive**.
  4. *Outgoing payment*: "Outgoing payments require an **interactive grant**", which means explicit consent from the sender. The sender's AS and identity provider (IdP) handle the interface. Client sends POST grant with `interact`; then redirect, IdP, consent, finish URI with `hash` and `interact_ref`; client verifies the hash; POST continuation to the `continue` URI with `interact_ref`; gets the access token; POST outgoing payment on the **sender's Resource server**, gets 201.
  5. *Transaction history*: GET list of incoming/outgoing payments (filtered by `wallet-address`) and GET by `id`.
- The page assumes "the client already has the sender's Open Payments-enabled account details". — [openpayments.dev/concepts/op-flow](https://openpayments.dev/concepts/op-flow/)
- "Clients must receive grants before issuing payment instructions." — [openpayments.dev/overview/getting-started](https://openpayments.dev/overview/getting-started/)
- The B2C guide "Accept a one-time payment for an online purchase" follows the same sequence. Step 1 GETs both wallet addresses (customer and retailer). Step 2 requests an incoming-payment grant using "the retailer's `authServer`". Step 3 creates the incoming payment "on the retailer's wallet account". Step 4 requests a quote grant using "the customer's `authServer`". Step 5 creates the quote with `receiver` = the incoming payment's `id`. Step 6 requests an interactive outgoing-payment grant on the customer's `authServer` "limited up to the amount of `140000`". Then interaction, continuation and outgoing payment follow. — [guides/accept-otp-online-purchase](https://openpayments.dev/guides/accept-otp-online-purchase/)
- If there is no UI for the redirect: "In a scenario where a user interface isn't available, consider implementing a polling mechanism to check for the completion of the interaction." — [guides/outgoing-grant-future-payments](https://openpayments.dev/guides/outgoing-grant-future-payments/)
- Grant actions (auth-server.yaml): `incoming-payment` allows `create, complete, read, read-all, list, list-all`; `outgoing-payment` allows `create, read, read-all, list, list-all` and **requires `identifier`** (the sender's wallet address), with optional `limits` and `cardAuthorization`; `quote` allows `create, read, read-all`. — [auth-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/auth-server.yaml)
- Official guides listed on getting-started: accept-otp-online-purchase, onetime-remittance-fixed-debit, onetime-remittance-fixed-receive, recurring-subscription-incoming-amount, recurring-remittance-fixed-debit, recurring-remittance-fixed-receive, split-payments, outgoing-grant-future-payments, verify-wallet-address-ownership. — [openpayments.dev/overview/getting-started](https://openpayments.dev/overview/getting-started/)

### Inferences
- Pago al Toque reverses the order of steps 3 and 4 compared with the one-time guide. The interactive outgoing-payment grant is obtained once beforehand, at onboarding with a weekly limit, and stored. In each charge, only incoming payment, then quote, then outgoing payment (with the stored token) run. That matches the official pattern in "recurring-subscription" ("You don't need to request new grants because the original grants should be valid for the remaining billing periods") and in "outgoing-grant-future-payments".
- Mapping to Pago al Toque: merchant = recipient (merchant's AS/RS for the incoming payment); payer = sender (payer's AS/RS for the quote and outgoing payment). Our Node.js server is the single "Client" in the diagram and holds the tokens for both sides.

### Gaps
- None material for this question.

## Q3. Which documented use case best matches POS / merchant charging with a pre-approved recurring limit; guidance on limits with intervals

### Takeaway
No documented use case is a POS. The closest match combines two guides. **"Get an outgoing payment grant for future payments"** covers a grant with a `debitAmount` limit plus `interval`, without knowing the recipient, plus the spent-amounts query. **"Set up recurring payments with a fixed incoming amount"** shows that each new charge repeats only incoming payment, quote and outgoing payment, reusing the grants. The "Accept a one-time payment for an online purchase" guide (B2C merchant) is the model for merchant-initiated charges.

### Cited Findings
- "Open Payments also supports a different approach: the client gets permission from the sender's ASE to send money before knowing who the recipient will be. Web Monetization's pay-as-you-browse model uses this approach." Scenario: "send payments of up to $100 CAD a month for three months without specifying a recipient beforehand." — [guides/outgoing-grant-future-payments](https://openpayments.dev/guides/outgoing-grant-future-payments/)
- Interval: "`R3/2025-05-20T13:00:00Z/P1M`" = 3 repetitions, start 20 May 2025 13:00 UTC, period 1 month. "The amount resets each month and any unspent portions don't roll over." — [guides/outgoing-grant-future-payments](https://openpayments.dev/guides/outgoing-grant-future-payments/)
- Schema: `interval` is an "[ISO8601 repeating interval]", examples `R11/2022-08-24T14:15:22Z/P1M`, `R/2017-03-01T13:00:00Z/2018-05-11T15:30:00Z`. `limits-outgoing` is the "Open Payments specific property that defines the limits under which outgoing payments can be created", either `anyOf` no-amount / `debitAmount` / `receiveAmount`, each with optional `receiver` and `interval`. On amounts: "All amounts are maxima, i.e. multiple payments can be created under a grant as long as the total amounts of these payments do not exceed the maximum amount per interval as specified in the grant." — [auth-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/auth-server.yaml)
- Official SDK snippet for a grant with an interval: `client.grant.request({url: walletAddress.authServer}, {access_token: {access: [{identifier: walletAddress.id, type: 'outgoing-payment', actions: ['list','list-all','read','read-all','create'], limits: {debitAmount: {...}, interval: 'R/2016-08-24T08:00:00Z/P1D'}}]}, interact: {start: ['redirect'], finish: {method: 'redirect', uri, nonce}}})`. — [snippets/node/grant/grant-outgoing-payment-interval.ts](https://github.com/interledger/open-payments/tree/main/snippets/node/grant)
- Recurring subscription: "For recurring payments, include the `interval` property", example `R12/2025-10-14T00:03:00Z/P1M`. After the first payment: "At the next interval ... repeat the following steps to request the creation of: 1. An incoming payment resource 2. A quote resource 3. An outgoing payment resource. You don't need to request new grants because the original grants should be valid for the remaining billing periods." "If a grant's access token has expired, call the Rotate Access Token API". — [guides/recurring-subscription-incoming-amount](https://openpayments.dev/guides/recurring-subscription-incoming-amount/)
- The continuation response returns `access_token.value`, `access_token.manage` (the token management URI) and the granted `access` with `limits` and `interval`. — [guides/recurring-subscription-incoming-amount](https://openpayments.dev/guides/recurring-subscription-incoming-amount/)

### Inferences
- For a weekly limit, use `interval: "R/<start-ISO>/P1W"` (`R` without a number = unbounded repetitions, per ISO 8601) with `limits.debitAmount` = the weekly cap. Omit `limits.receiver` so the grant works with any merchant. Store `access_token.value` and `access_token.manage`, and rotate with `token.rotate` when the token expires.
- The ASE enforces the limit when it creates the outgoing payment. Our pre-check against `quote.debitAmount` is a UX/anticipation step; the authoritative source for the remaining balance is the `/outgoing-payment-grant` endpoint (Q5).

### Gaps
- No official POS / card-present / NFC guide was found. The access-outgoing schema has a `cardAuthorization` property (auth-server.yaml), but I did not investigate its semantics. It may be relevant to "card" flows and is worth separate research.
- The docs do not say what HTTP error the RS returns when a payment exceeds the grant limit.

## Q4. How completion is detected (polling, `completed`, `receivedAmount`, webhooks, "complete")

### Takeaway
The Open Payments client detects completion by **polling** GET `/incoming-payments/{id}` (or GET `/outgoing-payments/{id}`). The spec defines no webhooks for clients. Rafiki webhooks go to the **ASE/tenant** operating Rafiki, not to third-party Open Payments clients. Completion happens automatically when `receivedAmount >= incomingAmount` (`completed: true`) or manually through POST `/incoming-payments/{id}/complete`.

### Cited Findings
- "A client can fetch the latest state of an incoming payment to determine the amount received into the wallet address." — [resource-server.yaml, GET /incoming-payments/{id}](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- Complete: "A client with the appropriate permissions MAY mark a non-expired **incoming payment** as `completed` indicating that the client is not going to make any further payments toward this **incoming payment**, even though the full `incomingAmount` may not have been received. This indicates to the receiving Account Servicing Entity that it can begin any post processing of the payment such as generating account statements or notifying the account holder of the completed payment." — [resource-server.yaml, POST /incoming-payments/{id}/complete](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- "A client can fetch the latest state of an outgoing payment." `failed` = "whether the payment failed to send its full amount"; `sentAmount` = total sent. — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- Rafiki webhooks go to the "tenant's configured webhook URL" (fallback `WEBHOOK_URL`; optionally the operator via `SEND_TENANT_WEBHOOKS_TO_OPERATOR`). Events: `incoming_payment.created`, `incoming_payment.completed` ("won't accept more incoming funds. Received funds should be withdrawn and credited"), `incoming_payment.partial_payment_received`, `incoming_payment.expired` (only if funds were received), `outgoing_payment.created` (waiting for liquidity; the tenant must fund or cancel it), `outgoing_payment.completed`, `outgoing_payment.failed`, and others. — [rafiki.dev/integration/requirements/webhook-events](https://rafiki.dev/integration/requirements/webhook-events/)

### Inferences
- These webhooks are an ASE ledger-integration mechanism (crediting, liquidity). An external client such as our Node.js server does not receive them unless it is also the ASE. With third-party wallets (for example the Interledger Test Wallet), the server should poll.
- Recommended polling for Pago al Toque: authenticated GET on the merchant's incoming payment (incoming grant with `read`) until `completed === true` or `receivedAmount.value >= incomingAmount.value`, with a timeout of `expiresAt`. In parallel or as a fallback, GET the outgoing payment on the payer's RS to detect `failed === true`. Note that Rafiki's outgoing `FUNDING` state means the sender's ASE has to fund it first, so there can be latency between 201 and settlement.
- Calling `complete` is not needed when `incomingAmount` is set and fully received (completion is automatic). It is useful to close a charge early, after a partial amount or a cancellation, and the grant must include the `complete` action.

### Gaps
- I found no official recommended polling interval.

## Q5. Resource server endpoints, grant spent amounts, and public (unauthenticated) GET of incoming payment

### Takeaway
RS endpoints (spec 1.4.0): POST/GET `/incoming-payments`, GET `/incoming-payments/{id}`, POST `/incoming-payments/{id}/complete`, POST/GET `/outgoing-payments`, GET `/outgoing-payments/{id}`, POST `/quotes`, GET `/quotes/{id}`, GET `/outgoing-payment-grant`. GET incoming payment accepts optional signature. Without auth it returns `public-incoming-payment` with only `receivedAmount` and `authServer` (it does **not** include `completed`).

### Cited Findings
- RS description: "**Endpoints:** `/incoming-payments` — create and manage incoming payments; `/outgoing-payments` — create and manage outgoing payment instructions; `/outgoing-payment-grant` — retrieve spent amounts for the current outgoing payment grant; `/quotes` — request a quote". "All resources are scoped to a wallet address and require an access token issued by the authorization server." Servers: `https://ilp.interledger-test.dev` and `.../.well-known/pay` "for when the wallet address has no pathname". — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- operationIds: `create-incoming-payment`, `list-incoming-payments`, `get-incoming-payment`, `complete-incoming-payment`, `create-outgoing-payment`, `list-outgoing-payments`, `get-outgoing-payment`, `create-quote`, `get-quote`, `get-outgoing-payment-grant`. — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- GET `/incoming-payments/{id}` uses `optional-signature-input` / `optional-signature` and returns `anyOf: incoming-payment-with-methods | public-incoming-payment`. `public-incoming-payment` = "An **incoming payment** resource with public details", properties `receivedAmount` and `authServer` ("The URL of the authorization server endpoint for getting grants and access tokens for this wallet address."), required: `authServer`. — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- The official SDK has an unauthenticated snippet: `const client = await createUnauthenticatedClient({}); await client.incomingPayment.get({ url: INCOMING_PAYMENT_URL })`. — [snippets/node/incoming-payment/incoming-payment-get-public.ts](https://github.com/interledger/open-payments/tree/main/snippets/node)
- GET `/outgoing-payment-grant`: "Returns the spent amounts for the current outgoing payment grant corresponding to the presented GNAP access token, if any. If a grant was created with an interval (recurring), the amounts returned are for the current interval." Response: `spentReceiveAmount` and `spentDebitAmount` (amount or `null`, for example `{spentReceiveAmount: null, spentDebitAmount: null}` when nothing has been spent). — [resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)
- The guide uses it to show how much is left: "You can provide your user with information about how much they have left to spend by calling the Get Spent Amounts for Current Outgoing Payment Grant API ... the API will only return the amounts for the current interval." — [guides/outgoing-grant-future-payments](https://openpayments.dev/guides/outgoing-grant-future-payments/)
- Inconsistency in the docs: that guide links the endpoint as `/apis/auth-server/operations/get-outgoing-payment-grant/`, but in the spec it lives in **resource-server.yaml** and the SDK page links `/apis/#tag/outgoing-payment/GET/outgoing-payment-grant`. — [guides/outgoing-grant-future-payments](https://openpayments.dev/guides/outgoing-grant-future-payments/) vs. [sdk/outgoing-grant-spent-amounts](https://openpayments.dev/sdk/outgoing-grant-spent-amounts/)

### Inferences
- Remaining weekly limit = `limits.debitAmount.value − spentDebitAmount.value` (treat `null` as 0), queried on the **payer's** RS with the stored outgoing-payment token. After each POST outgoing payment, `grantSpentDebitAmount` in the 201 response gives the same accumulated figure, so a separate call is not needed.
- An unauthenticated poll only sees `receivedAmount` (no `completed` and no `incomingAmount`), so the server has to compare against the `incomingAmount` it stored itself. An authenticated GET (with the merchant's incoming token holding `read`) gives the full object, including `completed`.

### Gaps
- The docs do not say whether `receivedAmount` is always present in the public view (it is not marked required).

## Q6. Official sequence diagrams: participants and arrows

### Takeaway
The op-flow page has per-step diagrams and a combined full-transaction diagram. The combined diagram's participants are **Sender's ASE** (Identity provider, Resource server, Auth server), **Client**, and **Recipient's ASE** (Wallet address URL, Auth server, Resource server), with 32 arrows.

### Cited Findings (all from [openpayments.dev/concepts/op-flow](https://openpayments.dev/concepts/op-flow/))
- Outgoing-payment interactive grant diagram (autonumbered, 18 steps): 1 Client→AS POST grant with `interact`; 2 AS→Client 200 with interact redirect URI + continue URI; 3 Client→AS navigates to redirect URI; 4 AS starts interaction, sets session; 5 AS→Client 302 to the IdP with grant info in the query string; 6 Client→IdP follows the redirect; 7 IdP: resource owner accepts; 8 IdP→AS interaction choice; 9 AS→IdP 202 choice accepted; 10 IdP→AS request to finish; 11 AS ends session; 12 AS→IdP 302 to the finish URI "secured with a hash and `interact_ref`"; 13 IdP→Client follows the redirect; 14 Client verifies the hash; 15 Client→AS POST continuation with `interact_ref` to the continue URI; 16 AS→Client 200 with access token; 17 Client→sender's RS POST outgoing payment; 18 RS→Client 201.
- Full transaction diagram (32 arrows):
  1 Client→Recipient wallet address: request the wallet address; 2 response with details;
  3 Client→Recipient AS: non-interactive incoming payment grant; 4 token + grant;
  5 Client→Recipient RS: create incoming payment; 6 Recipient RS→Recipient AS: token validation; 7 token validated; 8 Recipient RS→Client: incoming payment details;
  9 Client→Sender AS: non-interactive quote grant; 10 token + grant;
  11 Client→Sender RS: create quote; 12 quote created;
  13 Client→Sender AS: interactive outgoing payment grant; 14 interact redirect + continue URIs;
  15 Client→Sender AS: navigate to the redirect URI; 16 Sender AS starts interaction/session; 17 Sender AS→Client: IdP URI;
  18 Client→Sender IdP: redirect; 19 the sender accepts and confirms payment intent; 20 IdP→AS choice; 21 AS→IdP confirms; 22 IdP→AS finalize; 23 AS completes session; 24 AS→IdP redirect to the finish URI; 25 IdP→Client follows the redirect;
  26 Client verifies hash; 27 Client→Sender AS grant continuation; 28 Sender AS→Client access token;
  29 Client→Sender RS create outgoing payment; 30 Sender RS→Sender AS token validation; 31 validated; 32 Sender RS→Client outgoing payment details.
- Note: the combined diagram does not show the wallet-address GET for the sender (the page assumes the client already has the sender's details), and it does not show completing or polling the incoming payment.

### Inferences
- For the Pago al Toque diagram: split "Pago al Toque onboarding" (arrows 13–28 plus storing the token, run once per payer) from the "Charge" flow (arrows 1–12 plus 29–32 plus polling GET of the incoming payment against the Recipient RS). The RS→AS arrows (token introspection) are internal to each ASE; draw them dashed or leave them out.

### Gaps
- I could not see the original diagram images (Mermaid/SVG) directly. The participants and arrows come from the page text, extracted with a reading tool, not from reading the Mermaid source.

## Q7. SDK `@interledger/open-payments`: method for each step

### Takeaway
Methods confirmed in the official Node snippets: `walletAddress.get`, `walletAddress.getKeys`, `grant.request`, `grant.continue`, `grant.cancel`, `incomingPayment.create/get/list/complete`, `quote.create/get`, `outgoingPayment.create/get/list/getGrantSpentAmounts`, `token.rotate`, `token.revoke`. Clients are created with `createAuthenticatedClient({walletAddressUrl, privateKey, keyId})` or `createUnauthenticatedClient({})`.

### Cited Findings
- Methods counted in [snippets/node](https://github.com/interledger/open-payments/tree/main/snippets/node): `client.walletAddress.get`, `client.walletAddress.getKeys`, `client.grant.request`, `client.grant.continue`, `client.grant.cancel`, `client.incomingPayment.create`, `.get`, `.list`, `.complete`, `client.quote.create`, `.get`, `client.outgoingPayment.create`, `.get`, `.list`, `client.token.rotate`, `client.token.revoke`. Type guards: `isPendingGrant`, `isFinalizedGrantWithAccessToken`. — [open-payments repo snippets](https://github.com/interledger/open-payments/tree/main/snippets/node)
- `client.outgoingPayment.getGrantSpentAmounts({ url: walletAddress.resourceServer, accessToken: OUTGOING_PAYMENT_ACCESS_TOKEN })` returns `spentDebitAmount` / `spentReceiveAmount`; "For grants created with a recurring interval, the amounts returned reflect the current interval only." — [openpayments.dev/sdk/outgoing-grant-spent-amounts](https://openpayments.dev/sdk/outgoing-grant-spent-amounts/)
- Call shapes (snippets):
  - `client.incomingPayment.create({url: walletAddress.resourceServer, accessToken}, {walletAddress, incomingAmount: {value, assetCode, assetScale}, expiresAt})`
  - `client.quote.create({url: walletAddress.resourceServer, accessToken: QUOTE_ACCESS_TOKEN}, {method: 'ilp', walletAddress, receiver: INCOMING_PAYMENT_URL})`
  - `client.outgoingPayment.create({url: walletAddress.resourceServer, accessToken: OUTGOING_PAYMENT_ACCESS_TOKEN}, {walletAddress, quoteId: QUOTE_URL})`
  - `client.grant.request({url: walletAddress.authServer}, {access_token: {access: [{type: 'incoming-payment', actions: ['list','read','read-all','complete','create']}]}})`
  - `client.token.rotate({url: MANAGE_URL, accessToken: ACCESS_TOKEN})`
  — [snippets/node](https://github.com/interledger/open-payments/tree/main/snippets/node)
- SDK doc pages in the repo: before-you-begin, wallet-get-info, wallet-get-keys, grant-create-incoming, grant-create-quote, grant-create-outgoing, grant-continue, grant-revoke, incoming-create, incoming-get, incoming-list, incoming-complete, quote-create, quote-get, outgoing-create, outgoing-get, outgoing-list, outgoing-grant-spent-amounts, token-rotate, token-revoke. They are published at `https://openpayments.dev/sdk/<name>/`. — [docs/src/content/docs/sdk](https://github.com/interledger/open-payments/tree/main/docs/src/content/docs/sdk)

### Inferences
- Pago al Toque step-to-method mapping:
  1. `walletAddress.get` (merchant and payer)
  2. `grant.request` incoming (merchant AS)
  3. `incomingPayment.create` (merchant RS, `incomingAmount`, `expiresAt` = now+60 s)
  4. `grant.request` quote (payer AS)
  5. `quote.create` (payer RS, `method: 'ilp'`, `receiver`)
  6. `outgoingPayment.getGrantSpentAmounts` (or the local counter) to compare against the limit
  7. `outgoingPayment.create` (payer RS, `quoteId`, stored token)
  8. `incomingPayment.get` (polling) and/or `outgoingPayment.get`
  9. `token.rotate` if the token has expired
  Onboarding, once per payer: `grant.request` outgoing with `limits` + `interval` + `interact`, then the user consents, then `grant.continue({url: continue.uri, accessToken: continue.access_token.value}, {interact_ref})`.
- `walletAddress.get` returns `authServer` and `resourceServer`. `url` for the RS calls is `walletAddress.resourceServer`; for grants it is `walletAddress.authServer`.

### Gaps
- I did not check the exact signature of `grant.continue` or `incomingPayment.complete` in the code (only that they are called in the snippets). The `isFinalizedGrantWithAccessToken` check in the incoming-grant snippet seemed to have inverted logic in the extracted text; check it against the source before copying it.
