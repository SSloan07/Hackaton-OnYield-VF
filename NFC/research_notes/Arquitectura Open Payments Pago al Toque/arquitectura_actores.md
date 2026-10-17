# Open Payments: architecture, actors and terminology (for the "Pago al Toque" diagram)

Research date: 2026-10-16. Main source: openpayments.dev, current docs. Official English terms are kept verbatim. Note: page content was pulled with an automated fetch/summary tool, so quoted text is close to the original but should be checked against the live page before it goes into a final document.

## 1. Official actors and roles, and which ones the diagram needs

### Takeaway
The official actors are the **client**, the **resource owner (RO)**, and the **account servicing entity (ASE)**. Each ASE runs three servers (**wallet address server**, **resource server (RS)**, **authorization server (AS)**) and integrates an **identity provider (IdP)**. Payments always have a **sender's ASE** and a **recipient's ASE**. The Pago al Toque diagram should show: Client (our Node.js server), Resource owner (the wallet owner, who approves in the browser), and the ASE (Interledger test wallet / Rafiki) with its Wallet address server, AS, RS and IdP. If the merchant and payer wallets are separate, split the ASE side into Sender's ASE and Recipient's ASE.

### Cited Findings
- **Client**: "a client is an application, such as a mobile or web app, that consumes one or more Open Payments resources". Clients issue payment instructions. — [Getting started](https://openpayments.dev/overview/getting-started/)
- Glossary definition of **Client**: "A client is an application or service, such as a mobile or web app, that interacts with the authorization server to obtain grants and tokens." — [Glossary](https://openpayments.dev/resources/glossary/)
- The auth page calls the client "the piece of software, known as a client instance (or client for short)". It acts on behalf of the **RO** (resource owner), who issues the grant. — [Authorization](https://openpayments.dev/concepts/auth/)
- **Account servicing entity (ASE)**: "An account servicing entity provides and maintains a payment account for a sender and recipient, and is a regulated entity in the country/countries it operates." The getting-started page gives banks, digital wallet providers and mobile money providers as examples. — [Glossary](https://openpayments.dev/resources/glossary/); [Getting started](https://openpayments.dev/overview/getting-started/)
- ASEs "must operate three servers and integrate with an identity provider":
  - **Wallet address server**: public information about each account, at a stable HTTPS URL (the wallet address).
  - **Resource server**: hosts the incoming-payment, quote and outgoing-payment APIs.
  - **Authorization server**: processes GNAP grant requests, issues access tokens, and coordinates consent for interactive grants.
  - **Identity provider (IdP) integration**: authenticates the RO and collects explicit consent.
  — [ASE overview](https://openpayments.dev/implement/ase-overview/)
- **Authorization server (AS)**: "An authorization server issues tokens to clients, which can then be used to perform authorized actions… the authorization server grants permission for a client to access the Open Payments APIs". — [Glossary](https://openpayments.dev/resources/glossary/)
- **Resource server (RS)**: "A resource server hosts protected resources and enforces access controls based on the tokens provided by the authorization server." — [Glossary](https://openpayments.dev/resources/glossary/)
- **Wallet address server**: "A wallet address server is an API endpoint that retrieves public information about a wallet address, including its associated public keys." — [Wallet addresses](https://openpayments.dev/concepts/wallet-addresses/)
- **Identity provider (IdP)**: "a system or service that manages user authentication and consent during the interactive grant process". Also: "Open Payments requires any authorization server that issues interactive grants to integrate with an IdP." The AS integrates the IdP, not the client. — [Identity providers](https://openpayments.dev/identity/idp/)
- **Grant**: "an authorization issued by the RO that allows a client to access specific resources". — [Authorization](https://openpayments.dev/concepts/auth/)
- **Sender / recipient**: the docs use these only as qualifiers ("sender's ASE", "recipient's ASE"). They are not standalone glossary entries. The incoming payment is created by the recipient's ASE; the quote and outgoing payment are created by the sender's ASE. — [Glossary](https://openpayments.dev/resources/glossary/); [Resources](https://openpayments.dev/concepts/resources/)
- Open Payments carries only the payment instruction. The ASE moves the funds, and ASEs settle with each other over a shared payment rail. — [ASE overview](https://openpayments.dev/implement/ase-overview/)
- Open Payments "is a set of open RESTful APIs that enable clients to interface with Open Payments-enabled accounts". — [Glossary](https://openpayments.dev/resources/glossary/)

### Inferences
- Mapping to Pago al Toque:
  - Node.js server = **Client**.
  - Wallet owner who approves in the browser = **Resource owner** (in the outgoing-payment flow, this is the sender).
  - ilp.interledger-test.dev (Rafiki behind the test wallet) = **ASE** servers.
  - Test wallet consent screen = **IdP**.
- The Android app (NFC/HCE, Pay/Charge) is not an Open Payments actor. It is a front end of our own system. In the diagram it should sit outside the Open Payments boundary, or be labelled as part of the "client application" together with the Node server. The docs' own example of a client is "a mobile or web app", but here the signing keys live only on the server, so the server is the formal Client.
- The docs consistently show the AS and RS as separate boxes inside an ASE box, even though Rafiki runs them as separate services (auth and backend).

### Gaps
- None of the pages fetched define "resource owner" as a standalone term. It appears only as "RO" in the auth and IdP pages.

## 2. The wallet address and the wallet address document; key registry / JWKS

### Takeaway
A wallet address is an HTTPS URL that identifies an account. A `GET` on it (with `Accept: application/json`) returns `id`, `publicName`, `assetCode`, `assetScale`, `authServer` and `resourceServer`. The client's public keys are published at `{walletAddress}/jwks.json`.

### Cited Findings
- "A wallet address is a secure, sharable identifier for an Open Payments-enabled account." "Wallet addresses take the form of HTTPS URLs." — [Glossary](https://openpayments.dev/resources/glossary/); [Wallet addresses](https://openpayments.dev/concepts/wallet-addresses/)
- Example document returned by `GET https://wallet.example.com/alice` with `Accept: application/json`:
  ```json
  {
    "id": "https://wallet.example.com/alice",
    "publicName": "Alice",
    "assetCode": "USD",
    "assetScale": 2,
    "authServer": "https://auth.wallet.example.com",
    "resourceServer": "https://wallet.example.com/op"
  }
  ```
  "Each wallet address supports a single asset code and scale." — [Wallet addresses](https://openpayments.dev/concepts/wallet-addresses/)
- Field meanings (ASE implementation guide):
  - `id`: the canonical URL. It must match the identifier the client used.
  - `publicName`: a display label for clients and IdPs, publicly visible.
  - `assetCode`: "The ISO 4217 currency code of the underlying account".
  - `assetScale`: "The number of decimal places used for that currency".
  - `authServer`: the grant endpoint of the AS. Clients send all grant requests there.
  - `resourceServer`: the base URL of the RS that hosts incoming-payment, quote and outgoing-payment.
  — [Wallet address architecture](https://openpayments.dev/implement/wallet-address-architecture/)
- API reference: `GET /` (wallet address) and `GET /jwks.json` ("Get the keys bound to a Wallet Address"). — [API reference](https://openpayments.dev/apis/#tag/wallet-address/GET/jwks.json); SDK page: [wallet-get-keys](https://openpayments.dev/sdk/wallet-get-keys)
- **Key registry**: the set of public keys a client uses to identify itself. The ASE hosts it on its resource server, at `WALLET_ADDRESS/jwks.json`, for example `https://wallet.example.com/alice/jwks.json`. — [Client keys](https://openpayments.dev/identity/client-keys/)
- **Payment pointer** (older Interledger term): "A payment pointer may serve as a wallet address if Interledger is used as the payment method". — [Glossary](https://openpayments.dev/resources/glossary/)

### Inferences
- For the diagram: the arrow "Client → Wallet address server: GET wallet address" returns `authServer`/`resourceServer`, which then tell the client where to send the next calls. The JWKS lives under the client's own wallet address (the one in `walletAddressUrl` of `createAuthenticatedClient`), not under the payer's or payee's wallet address, unless those are the same.

### Gaps
- No official page fetched gives a concrete test-wallet wallet address in the `https://ilp.interledger-test.dev/<name>` format. The user's context suggests it, but I could not confirm it from openpayments.dev (the SDK pages use placeholders).

## 3. Client identification: keys, key ID, HTTP message signatures, verification

### Takeaway
The client identifies itself with its **client wallet address** plus an **Ed25519** key pair. It signs every request with HTTP Message Signatures (`Signature-Input` + `Signature`, with `Content-Digest` covering the body). The AS reads the `keyid`, fetches `{client wallet address}/jwks.json`, validates the signature and binds the client to the grant.

### Cited Findings
- "Client keys are unique identifiers that allow clients to sign Open Payments requests and verify the client's identity." Every request except new grant requests carries an access token bound to the key. — [Client keys](https://openpayments.dev/identity/client-keys/)
- Keys must be **Ed25519**. Required JWK values: `alg: "EdDSA"`, `kty: "OKP"`, `crv: "Ed25519"`, plus `x` (the public key) and `kid` (the key ID). — [Client keys](https://openpayments.dev/identity/client-keys/)
- Key lifecycle: the SDK generates the key pair, the public key goes to the ASE, the ASE adds it to its RS and usually creates the `keyId`, and the client keeps the private key. — [Client keys](https://openpayments.dev/identity/client-keys/)
- Verification by the AS: when the grant request's `client` holds a `walletAddress`, the AS takes the client's domain from it and binds it to the grant. It then GETs `WALLET_ADDRESS/jwks.json`, finds the key matching the `keyId` in `Signature-Input`, and validates the signature. With "directed identity" (a `jwk` in the body, for certain non-interactive grants only), no JWKS fetch happens. — [Client keys](https://openpayments.dev/identity/client-keys/)
- Official sequence diagram on the client keys page. Participants: **Open Payments Client SDK (C)**, **Authorization server (AS)**, **Resource server (RS)**. Steps:
  1. C initializes the SDK with the wallet address URL, keyId and private key.
  2. C sends a signed POST grant request (interactive outgoing-payment).
  3. AS reads the keyId from `signature-input` and the client domain from the body.
  4. AS → RS: GET the client's jwks.json.
  5. RS returns the JWKS.
  6. AS validates the signature and binds the domain to the grant.
  7. AS returns 200 OK.
  8. Note: consent is collected from the user via the client, AS and an IdP (not shown).
  9. C sends a signed POST grant continuation.
  10. AS takes the keyId and domain from the stored grant.
  11. AS → RS: GET JWKS.
  12. RS returns the JWKS.
  13. AS validates the signature.
  14. AS returns 200 with the access token.
  — [Client keys](https://openpayments.dev/identity/client-keys/)
- HTTP signatures: the message is signed by "adding uniquely labelled signature headers… `Signature-Input` and `Signature`". The algorithm is "the Ed25519 variant of the EdDSA". The page points to GNAP §7.3.1 for the required components. — [HTTP signatures](https://openpayments.dev/identity/http-signatures/)
- Example from that page:
  - `Signature-Input: sig1=("content-type" "content-digest" "content-length" "authorization" "@method" "@target-uri");alg="ed25519";keyid="eddsa_key_1";created=1704722601`
  - `Signature: sig1=:…:`
  - `Content-Digest: sha-512=:…:`
  - `Authorization: GNAP 123454321`

  Generation: SHA-512 hash of the signature base, signed with the private key, then Base64-encoded. — [HTTP signatures](https://openpayments.dev/identity/http-signatures/)
- In Rafiki, the auth service handles "resolving clients' public keys to authenticate and authorize incoming requests". — [Rafiki: Open Payments](https://rafiki.dev/overview/concepts/open-payments/)
- SDK: `createAuthenticatedClient({ walletAddressUrl: CLIENT_WALLET_ADDRESS, privateKey: PRIVATE_KEY_PATH, keyId: KEY_ID })`. — [SDK: grant-create-outgoing](https://openpayments.dev/sdk/grant-create-outgoing/)
- Helper package `@interledger/http-signature-utils` "provides tools for working with HTTP Message Signatures". — [open-payments-node repo](https://github.com/interledger/open-payments-node)

### Inferences
- For the diagram: the Node server holds `private.key`. The public key / JWKS lives at the ASE, under the client wallet address. Draw the AS → JWKS (wallet address server / RS) arrow as "GET {client wallet address}/jwks.json" so it matches the official diagram, which places it on the **Resource server**.
- The SDK signs requests automatically. This is consistent with the SDK pages, but the open-payments-node README I fetched did not state it explicitly.

### Gaps
- The HTTP signatures page does not describe the AS/RS verification steps beyond what the client keys page says, and does not list which components are mandatory.
- I did not fetch the hash-verification page (https://openpayments.dev/identity/hash-verification/), which covers verifying the finish-redirect `hash`.

## 4. Relation with Interledger (ILP), Rafiki and the test wallet; the "ilp" method; getting set up

### Takeaway
Open Payments is an abstraction layer for payment instructions. ILP is currently the only payment method integrated with it. Rafiki is the open-source reference implementation of the ASE server stack. The test wallet (wallet.interledger-test.dev, on a GateHub pre-production environment) is where you get the wallet address, `keyId` and `private.key`. `"ilp"` is relevant as the `methods[].type` of the incoming payment and as the `method` of the quote request.

### Cited Findings
- "Open Payments is designed to be an abstraction layer that can issue payment instructions" across any payment method. ILP is currently the only integrated method. — [Payments](https://openpayments.dev/concepts/payments/)
- The incoming payment's `methods` object includes `type: "ilp"`, `ilpAddress` (for example `g.ilp.iwuyge987y.98y08y`) and `sharedSecret` (for STREAM). "After receiving the incoming payment, the sender's ASE creates a quote request with `"method": "ilp"`." — [Payments](https://openpayments.dev/concepts/payments/)
- ILP is an "open protocol suite for transferring packets of value between peers". — [Payment methods](https://openpayments.dev/resources/payment-methods/)
- Rafiki: "a concrete, open-source reference implementation of an Open Payments-compatible server stack". — [ASE overview](https://openpayments.dev/implement/ase-overview/)
- Rafiki makes accounts Open Payments-enabled and is a reference implementation of the ILP stack, so a financial service provider can become an Interledger node. — [Rafiki overview](https://rafiki.dev/overview/overview/)
- Rafiki services:
  - **Backend**: exposes "the endpoints of the Open Payments APIs" (the RS role).
  - **Auth service**: "a reference implementation of an opinionated Open Payments authorization server" (the AS role).
  — [Rafiki: Open Payments](https://rafiki.dev/overview/concepts/open-payments/)
- Getting-started page: "All requests require signatures, which protect the integrity of the requests" (HTTP Signatures). It has no keys or test-wallet procedure itself and links to /identity/client-keys/ and /sdk/before-you-begin. — [Getting started](https://openpayments.dev/overview/getting-started/)
- Test wallet setup:
  1. Create an account at https://wallet.interledger-test.dev (only the email needs to be real for KYC).
  2. Create an account and deposit play money.
  3. "Add wallet address" (name + public name).
  4. Go to Settings → **Developer Keys** → "Generate public & private key". "A file named `private.key` automatically downloads"; the **key ID** is then shown.

  "The Interledger test wallet environment is running on a GateHub pre-production environment." — [Before you begin](https://openpayments.dev/sdk/before-you-begin)

### Inferences
- In the diagram, label the ASE box "Interledger test wallet (ASE) – Rafiki", with sub-boxes "Wallet address server", "Authorization server (Rafiki auth)", "Resource server (Rafiki backend)" and "IdP (test wallet consent UI)". Fund movement between the sender's and recipient's ASE goes over ILP. This happens outside Open Payments and can be drawn as a dashed "ILP" link.
- The prototype's Node code normally does not set `method: 'ilp'` explicitly unless it creates the quote with an explicit method. The concept page describes it as part of the quote request. Check this against the SDK quote-create example.

### Gaps
- I could not confirm from official docs that the wallet address / Open Payments host for the test wallet is `ilp.interledger-test.dev`. The before-you-begin page only names wallet.interledger-test.dev.
- I did not verify the Rafiki architecture page (component diagram) or the Rafiki IdP integration page.

## 5. Official diagrams on openpayments.dev

### Takeaway
The official diagrams are Mermaid sequence diagrams (with "View full diagram" / "Download diagram" links and no alt text). They appear on /concepts/op-flow/ (per-phase diagrams and "Bringing it all together"), /identity/idp/ (interactive grant, 16 steps) and /identity/client-keys/ (signature validation, listed in section 3). Getting-started, wallet-addresses, auth, resources, payments, http-signatures and ase-overview have no diagrams.

### Cited Findings
- "Bringing it all together" diagram: participants are grouped into a light-blue **sender's ASE** box (IdP, RS, AS) and a light-green **recipient's ASE** box (wallet address URL, AS, RS), with the **Client** outside both. The RS asks the AS to validate the token before responding (the token introspection arrow). — [Open Payments flow](https://openpayments.dev/concepts/op-flow/)
- Diagram labels (verbatim), per phase:
  - "GET wallet address URL (e.g. https://wallet.example.com/alice)" / "200 wallet address found, return public account details"
  - "POST grant request with type=incoming-payment" / "200 OK, returns access token" / "POST create incoming payment" / "201 incoming payment created, return incoming payment with public details"
  - "POST grant request with type=quote" / "POST create quote" / "201 quote created, returns quote details"
  - Outgoing: "POST grant request (with interact object)" / "200 OK, returns interact redirect URI and continue URI" / … / "POST create outgoing payment" / "201 outgoing payment created, returns outgoing payment details"

  — [Open Payments flow](https://openpayments.dev/concepts/op-flow/)
- IdP diagram (Client, Authorization server (AS), Identity provider (IdP)):
  1. Grant request with interact.
  2. 200 with interact redirect URI + continue URI.
  3. Client navigates to the interact redirect URI.
  4. AS starts the interaction and sets the session.
  5. AS returns a 302 redirect to the IdP URI, with grant info in the query string.
  6. Client is redirected to the IdP.
  7. The RO accepts the interaction.
  8. IdP sends the interaction choice to the AS.
  9. AS returns 202 choice accepted.
  10. IdP requests to finish the interaction.
  11. AS ends the session.
  12. AS returns a 302 to the finish URI, "secured with unique hash" and with `interact_ref`.
  13. IdP follows the redirect to the client.
  14. Client verifies the hash.
  15. Client POSTs the continuation with `interact_ref` to the continue URI.
  16. AS returns 200 with the grant access token.

  — [Identity providers](https://openpayments.dev/identity/idp/)

### Inferences
- To stay consistent: use the box names exactly as shown ("Client", "Authorization server (AS)", "Resource server (RS)", "Identity provider (IdP)", "Wallet address"), grouped into an ASE container. Use the arrow verbs "POST grant request", "POST grant continuation request", "POST create …".
- In Pago al Toque, the browser where the owner approves is where the IdP interaction happens. The finish URI (the `interact.finish.uri` set by the Node server) is our server's callback.

### Gaps
- The Mermaid source and images could not be viewed directly. The descriptions above come from the page text and the extracted Mermaid content.

## 6. The "Open Payments flow" overview (high-level steps)

### Takeaway
The canonical flow is:
1. Get wallet address details.
2. Get a non-interactive incoming-payment grant and create an incoming payment (recipient's ASE).
3. Get a non-interactive quote grant and create a quote (sender's ASE).
4. Get an **interactive** outgoing-payment grant (redirect to IdP → consent → finish redirect + hash → continuation) and create the outgoing payment (sender's ASE).
5. Optionally, list or get payments.

### Cited Findings
- The op-flow page phases are: Get account details → Incoming payment → Quote → Outgoing payment (interactive) → Transaction history. "For outgoing payments, explicit user consent is required before proceeding with the continuation request." — [Open Payments flow](https://openpayments.dev/concepts/op-flow/)
- The grants page lists "Path 1" (6 steps: incoming-payment grant → incoming payment → quote grant → quote → interactive outgoing-payment grant → outgoing payment) and "Path 2", where one interactive grant covers both quote and outgoing-payment. — [Grants](https://openpayments.dev/identity/grants/)
- outgoing-payment grants are always interactive. incoming-payment and quote grants are non-interactive by default, but if the request includes `list-all`, the AS "should require interaction". After interaction, "the client must issue a Grant Continuation request". — [Authorization](https://openpayments.dev/concepts/auth/)
- Grant with limits (relevant to the weekly cap): `access_token.access[{ type: 'outgoing-payment', actions: [...,'create'], identifier: walletAddress.id, limits: { debitAmount: {assetCode, assetScale, value}, interval: 'R/2016-08-24T08:00:00Z/P1D' } }]`, with `interact: { start: ['redirect'], finish: { method: 'redirect', uri, nonce } }`, sent to `{ url: walletAddress.authServer }`. — [SDK: grant-create-outgoing](https://openpayments.dev/sdk/grant-create-outgoing/)
- The outgoing payment resource "serve[s] as an instruction to the sender's ASE to make a payment"; Open Payments does not execute payments itself. Quotes "are only valid for a limited time". — [Resources](https://openpayments.dev/concepts/resources/)

### Inferences
- The weekly cap would be an ISO 8601 repeating interval with `P1W` (for example `R/<start>/P1W`) together with `debitAmount`. The docs only show `P1D` examples and do not explain the format.

### Gaps
- No official page explains the interval format. The `P1W` value is inferred from ISO 8601, not from the docs.

## Terminology changes between versions (noted while researching)
- "Payment pointer" (the `$wallet.example/alice` style) has been replaced by "wallet address" (an HTTPS URL). The glossary keeps payment pointer only as an ILP-specific alias. — [Glossary](https://openpayments.dev/resources/glossary/)
- The old docs domain openpayments.guide (for example https://openpayments.guide/snippets/wallet-get-info) is now openpayments.dev. The old test wallet "Rafiki.Money" and wallet.interledger.com/testdev are now wallet.interledger-test.dev, per search results pointing at older pages. — [openpayments.guide snippet](https://openpayments.guide/snippets/wallet-get-info); [Before you begin](https://openpayments.dev/sdk/before-you-begin)
- The SDK repo is now `interledger/open-payments-node`, while some links still point to `interledger/open-payments`. — [open-payments-node](https://github.com/interledger/open-payments-node)
- GNAP is now RFC 9635 (https://www.rfc-editor.org/info/rfc9635/), as linked from the ASE overview. — [ASE overview](https://openpayments.dev/implement/ase-overview/)
