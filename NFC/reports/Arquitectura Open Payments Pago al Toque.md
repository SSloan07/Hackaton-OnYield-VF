# Pago al Toque encaja en Open Payments

En la terminología oficial de Open Payments, **Pago al Toque es un único Client (el servidor Node.js) que actúa entre dos roles de Account servicing entity: el sender's ASE (la wallet del pagador, "rosa") y el recipient's ASE (la wallet del comercio, "josep")**. La app Android queda fuera de la frontera de Open Payments: es un front end propio que captura toques NFC y le pide cosas al Client por HTTP. Cada cobro sigue el "Path 1" oficial (incoming payment en el recipient's ASE, quote y outgoing payment en el sender's ASE), con una diferencia que la documentación prevé. El grant interactivo de outgoing payment no se pide en cada cobro. Se obtiene una sola vez, con un límite semanal (`limits.debitAmount` + `interval: R/…/P1W`), y se reutiliza en cada cobro posterior, igual que en las guías oficiales de pagos recurrentes y futuros. Ambos roles de ASE los cumple la misma instancia de Rafiki detrás del Interledger test wallet (`ilp.interledger-test.dev`). En el diagrama son cajas separadas porque así lo modela la documentación y porque cada wallet address tiene su propio Authorization server y Resource server. La implementación es coherente con los flujos oficiales. Los riesgos están en los bordes del protocolo: el hash de interacción, la revocación de tokens, la ventana de 60 s y el control de estado del incoming payment.

## Tres actores oficiales y una app que queda fuera de la frontera

Open Payments define tres actores. El primero es el **Client**: "an application, such as a mobile or web app, that consumes one or more Open Payments resources" ([Getting started](https://openpayments.dev/overview/getting-started/)). El segundo es el **Resource owner (RO)**, que emite el grant. El tercero es la **Account servicing entity (ASE)**, que "provides and maintains a payment account for a sender and recipient" ([Glossary](https://openpayments.dev/resources/glossary/)). Cada ASE "must operate three servers and integrate with an identity provider": un **Wallet address server**, un **Resource server** (incoming payment, quote, outgoing payment) y un **Authorization server** (grants GNAP y access tokens), además de la integración con un **Identity provider** que autentica al RO y recoge su consentimiento ([ASE overview](https://openpayments.dev/implement/ase-overview/)). Open Payments solo transporta la instrucción de pago: **el dinero lo mueve la ASE**, y entre ASEs se liquida por un riel compartido, que hoy es solo ILP ([Payments](https://openpayments.dev/concepts/payments/)).

La documentación pone como ejemplo de Client una app móvil. Aun así, en Pago al Toque el Client formal es el **servidor Node.js**, porque es el único que tiene la clave Ed25519 (`private.key`) y el `keyId`. También es el que firma cada petición con HTTP Message Signatures. El Authorization server valida esa firma descargando `jwks.json` de la wallet address del propio Client (`https://ilp.interledger-test.dev/app`), no de la del pagador ni de la del comercio ([Client keys](https://openpayments.dev/identity/client-keys/)). La app Android (modos Cobrar/Pagar, HCE con AID `F0 "DATAFONO"`, tarjetas por UID) no firma nada ni tiene tokens. Por eso queda fuera de la frontera Open Payments. El host `https://ilp.interledger-test.dev` aparece como servidor en la especificación OpenAPI del Resource server ([resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)). Rafiki implementa el Authorization server con su servicio *auth* y el Resource server con su *backend* ([Rafiki: Open Payments](https://rafiki.dev/overview/concepts/open-payments/)).

| Actor oficial | Componente en Pago al Toque |
|---|---|
| Client | Servidor Node.js, Express 5, `@interledger/open-payments` 7.4, `createAuthenticatedClient` con wallet `/app` |
| Resource owner | Titular de la wallet del pagador ("rosa"), que aprueba en el test wallet |
| Sender's ASE | Test wallet / Rafiki para `https://ilp.interledger-test.dev/rosa` (USD, assetScale 2) |
| Recipient's ASE | Test wallet / Rafiki para `https://ilp.interledger-test.dev/josep` |
| Identity provider | Pantalla de login y consentimiento del Interledger test wallet |
| Fuera de Open Payments | App Android (Cobrar, Pagar con HCE, tarjetas NFC) y `datos.json` |

### Diagrama 1: componentes sobre los actores oficiales

Los números 1–9 son las llamadas de un cobro. G1–G6 son la aprobación única del grant interactivo. Las líneas discontinuas indican tráfico interno de la ASE o fuera de Open Payments.

```mermaid
flowchart LR
  subgraph APP["App Android Pago al Toque - fuera de Open Payments"]
    COB["Modo Cobrar - lector NFC tipo datáfono"]
    PAG["Modo Pagar - HCE AID F0 DATAFONO tras biometría 60 s"]
    TAR["Tarjeta NFC física - UID"]
  end

  RO(["Resource owner - titular de la wallet rosa"])

  subgraph CLI["Client - servidor Node.js localhost:3000"]
    API["API REST /api/cobros"]
    FAM["Página /familiar y /familiar/completar"]
    SDK["SDK @interledger/open-payments 7.4 - createAuthenticatedClient"]
    DB[("datos.json - cuentas, medios, permisos, cobros")]
  end

  subgraph SASE["Sender's ASE - Interledger test wallet / Rafiki - wallet rosa"]
    SWA["Wallet address server - /rosa"]
    SAS["Authorization server"]
    SRS["Resource server"]
    SIDP["Identity provider - consentimiento del test wallet"]
  end

  subgraph RASE["Recipient's ASE - Interledger test wallet / Rafiki - wallet josep"]
    RWA["Wallet address server - /josep"]
    RAS["Authorization server"]
    RRS["Resource server"]
  end

  KEYS["Key registry del Client - /app/jwks.json"]

  PAG -.->|"toque NFC ISO-DEP"| COB
  TAR -.->|"toque NFC UID"| COB
  COB -->|"HTTP por adb reverse tcp:3000"| API
  API --> SDK
  FAM --> SDK
  SDK --- DB

  SDK -->|"1 walletAddress.get - GET wallet address"| RWA
  SDK -->|"2 grant.request incoming-payment - no interactivo"| RAS
  SDK -->|"3 incomingPayment.create - incomingAmount, expiresAt +60 s"| RRS
  SDK -->|"4 walletAddress.get - datos del pagador"| SWA
  SDK -->|"5 outgoingPayment.getGrantSpentAmounts"| SRS
  SDK -->|"5b token.rotate si 401/403 Inactive Token"| SAS
  SDK -->|"6 grant.request quote - no interactivo"| SAS
  SDK -->|"7 quote.create - method ilp, receiver"| SRS
  SDK -->|"8 outgoingPayment.create - quoteId"| SRS
  SDK -->|"9 incomingPayment.get - polling hasta completed"| RRS

  SDK -->|"G1 grant.request outgoing-payment con interact y limits P1W"| SAS
  SAS -->|"G2 302 redirect al IdP"| SIDP
  RO -->|"G3 autentica y aprueba"| SIDP
  SIDP -->|"G4 redirect a finish URI con hash e interact_ref"| FAM
  RO -->|"G5 pega URL en /familiar/completar"| FAM
  SDK -->|"G6 grant.continue con interact_ref"| SAS

  SRS -.->|"liquidación ILP - fuera de Open Payments"| RRS
  SRS -.->|"introspección de token"| SAS
  RRS -.->|"introspección de token"| RAS
  SAS -.->|"GET jwks.json - valida firma"| KEYS
  RAS -.->|"GET jwks.json - valida firma"| KEYS
```

## Un cobro recorre el Path 1 con el grant interactivo ya guardado

El flujo oficial de extremo a extremo tiene cinco pasos. Primero se obtienen los datos de la wallet address. Después se pide un grant **no interactivo** `incoming-payment` al Authorization server del **recipient** y se crea el incoming payment en su Resource server. Luego, un grant **no interactivo** `quote` y la creación del quote en el **sender's ASE**. Después, un grant **interactivo** `outgoing-payment` y el outgoing payment, también en el sender's ASE. Por último, la consulta opcional de historial ([Open Payments flow](https://openpayments.dev/concepts/op-flow/)). El diagrama oficial "Bringing it all together" agrupa los participantes en una caja **sender's ASE** (IdP, RS, AS) y otra **recipient's ASE** (wallet address URL, AS, RS), con el **Client fuera de ambas**. El diagrama 2 conserva esa estructura.

Pago al Toque saca el bloque interactivo del cobro. La guía de suscripciones recurrentes lo dice explícitamente: en cada intervalo se repiten solo incoming payment, quote y outgoing payment, porque "You don't need to request new grants because the original grants should be valid for the remaining billing periods" ([Recurring subscription](https://openpayments.dev/guides/recurring-subscription-incoming-amount/)). La guía "outgoing grant for future payments" cubre justo el caso de autorizar gasto **sin conocer al receptor de antemano**, con un `debitAmount` por intervalo ([Future payments](https://openpayments.dev/guides/outgoing-grant-future-payments/)). Ninguna guía oficial trata un POS o pago presencial, así que esta combinación es la más cercana documentada.

Hay dos detalles que el diagrama refleja. El primero es el quote. Como el incoming payment ya trae `incomingAmount`, el quote se crea con la variante "Receiver with incoming amount" (`method: "ilp"` + `receiver`, sin importe), que es la exigida por la especificación cuando el receptor fija el importe ([resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)). El `quote.debitAmount` que devuelve es el coste real, comisiones incluidas, y es lo que se compara con el tope restante. El segundo es el saldo del límite. `GET /outgoing-payment-grant` (`getGrantSpentAmounts`) devuelve `spentDebitAmount` **solo para el intervalo actual**, y puede venir `null` ([Spent amounts SDK](https://openpayments.dev/sdk/outgoing-grant-spent-amounts/)). El cálculo correcto es `tope − (spentDebitAmount ?? 0)`.

### Diagrama 2: un cobro de extremo a extremo

Los números ①–④ son las fases de nuestra API. Los números 1–20 son las llamadas Open Payments.

```mermaid
sequenceDiagram
  participant APP as App Android - fuera de Open Payments
  box rgb(214,232,248) Sender's ASE - wallet rosa
    participant SWA as Wallet address server rosa
    participant SRS as Resource server rosa
    participant SAS as Authorization server rosa
  end
  participant C as Client - Node.js
  box rgb(216,242,216) Recipient's ASE - wallet josep
    participant RWA as Wallet address server josep
    participant RAS as Authorization server josep
    participant RRS as Resource server josep
  end

  rect rgb(245,245,245)
  Note over APP,C: Fase 1 - el comercio crea el cobro en modo Cobrar
  APP->>C: ① POST /api/cobros con monto
  C->>RWA: 1. walletAddress.get josep - GET wallet address
  RWA-->>C: 2. 200 assetCode USD, assetScale 2, authServer, resourceServer
  C->>RAS: 3. grant.request incoming-payment create read list complete - grant no interactivo
  RAS-->>C: 4. 200 access token
  C->>RRS: 5. incomingPayment.create incomingAmount, expiresAt ahora +60 s
  RRS-->>RAS: 6. introspección del access token
  RAS-->>RRS: 7. token activo
  RRS-->>C: 8. 201 incoming payment con id y methods ilp
  C-->>APP: cobro pendiente con id
  end

  rect rgb(245,245,245)
  Note over APP,C: Fase 2 - primer toque con HCE o tarjeta UID
  APP->>C: ② POST /api/cobros/:id/tarjeta con medio
  Note over C: resuelve medio a cuenta rosa y busca permiso en datos.json
  C->>SWA: 9. walletAddress.get rosa - authServer y resourceServer del pagador
  SWA-->>C: 10. 200 datos públicos de la wallet
  C->>SRS: 11. outgoingPayment.getGrantSpentAmounts con token interactivo
  alt 401 o 403 Inactive Token
    C->>SAS: 11b. token.rotate - POST manage URL
    SAS-->>C: nuevo access_token value y manage, se reintenta 11
  end
  SRS-->>C: 12. spentDebitAmount del intervalo semanal actual
  C->>SAS: 13. grant.request quote create read - grant no interactivo
  SAS-->>C: 14. 200 access token de quote
  C->>SRS: 15. quote.create method ilp, receiver igual a URL del incoming payment
  SRS-->>C: 16. 201 quote con debitAmount, receiveAmount, expiresAt
  Note over C: si debitAmount supera el tope restante responde TOPE_SUPERADO
  C-->>APP: importe a confirmar
  end

  rect rgb(245,245,245)
  Note over APP,C: Fase 3 - segundo toque con el mismo medio
  APP->>C: ③ POST /api/cobros/:id/confirmar
  C->>SRS: 17. outgoingPayment.create quoteId con token interactivo, conRotacion
  SRS-->>C: 18. 201 outgoing payment con grantSpentDebitAmount
  SRS--)RRS: liquidación ILP STREAM entre ASEs, fuera de Open Payments
  end

  rect rgb(245,245,245)
  Note over APP,C: Fase 4 - la app consulta el estado
  APP->>C: ④ GET /api/cobros/:id en polling
  C->>RRS: 19. incomingPayment.get con token del comercio
  RRS-->>C: 20. receivedAmount y completed true
  C-->>APP: cobro pagado
  end
```

## La aprobación única replica el diagrama oficial del Identity provider

El grant `outgoing-payment` es **siempre interactivo**: "Open Payments requires outgoing payment grant requests to be interactive", y después de la interacción el Client tiene que enviar un Grant Continuation request ([Authorization](https://openpayments.dev/concepts/auth/)). La secuencia oficial es la siguiente ([Identity providers](https://openpayments.dev/identity/idp/)):

1. El Client envía el grant request con el objeto `interact`.
2. El Authorization server devuelve el redirect URI y el continue URI.
3. El navegador va al Authorization server, que responde con un 302 al IdP.
4. El RO acepta en el IdP.
5. El IdP comunica la elección y el Authorization server responde 202.
6. Un 302 envía al finish URI con `hash` e `interact_ref`.
7. El Client verifica el hash.
8. El Client envía `POST` al continue URI con `interact_ref` y recibe el access token.

Open Payments solo admite `start: ["redirect"]` y `finish.method: "redirect"` ([auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)). Nuestra petición usa exactamente eso.

Nuestro grant pide `outgoing-payment` con las acciones `create, read, list, read-all, list-all`, con `identifier` igual a la wallet del pagador (obligatorio en el esquema `access-outgoing`) y `limits: { debitAmount: tope, interval: "R/<inicio>/P1W" }`. El `interval` es un "ISO8601 repeating interval" y los importes "are maxima" por intervalo, sin arrastre de lo no gastado ([auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml); [Future payments](https://openpayments.dev/guides/outgoing-grant-future-payments/)). Los ejemplos oficiales solo usan `P1D` y `P1M`. El valor `P1W` se deduce de ISO 8601, no aparece en ningún ejemplo. Añadir `incoming-payment` con `list-all` dentro de este grant interactivo es correcto: Rafiki exige interacción para `list-all` por defecto (`LIST_ALL_ACCESS_INTERACTION=true`) ([Rafiki auth config](https://raw.githubusercontent.com/interledger/rafiki/main/packages/auth/src/config/app.ts)). La caída a un alcance básico ante un 4xx es una defensa razonable.

El único paso propio, sin equivalente en el diagrama oficial, es el puente manual. El finish URI apunta a `http://localhost:3000/familiar/callback`, que solo resuelve en la máquina del servidor. Por eso el titular pega la URL resultante en `/familiar/completar`. Para el protocolo es indiferente: el Client recibe los mismos `hash` e `interact_ref`.

### Diagrama 3: aprobación del grant interactivo, una sola vez

```mermaid
sequenceDiagram
  actor RO as Resource owner - titular rosa
  participant NAV as Navegador - página /familiar
  participant C as Client - Node.js
  box rgb(214,232,248) Sender's ASE - wallet rosa
    participant SWA as Wallet address server rosa
    participant SAS as Authorization server rosa
    participant IDP as Identity provider - test wallet
  end

  RO->>NAV: abre /familiar y elige tope semanal
  NAV->>C: solicitud de vinculación de la cuenta
  C->>SWA: 1. walletAddress.get rosa - GET wallet address
  SWA-->>C: 2. 200 authServer, resourceServer, assetCode, assetScale
  C->>SAS: 3. grant.request outgoing-payment con limits debitAmount e interval R/inicio/P1W, más incoming-payment read list read-all list-all, interact start redirect, finish redirect con uri y nonce
  alt rechazo 4xx por alcance
    C->>SAS: 3b. grant.request con alcance básico
  end
  SAS-->>C: 4. 200 interact.redirect, interact.finish, continue uri y access_token
  Note over C: guarda nonce, finish y continue
  C-->>NAV: 5. muestra enlace interact.redirect
  NAV->>SAS: 6. navega al redirect URI - inicia interacción y sesión
  SAS-->>NAV: 7. 302 al Identity provider con datos del grant
  NAV->>IDP: 8. sigue el redirect
  RO->>IDP: 9. inicia sesión y aprueba - resource owner accepts
  IDP->>SAS: 10. envía la elección de la interacción
  SAS-->>IDP: 11. 202 elección aceptada
  IDP->>SAS: 12. solicita finalizar la interacción
  SAS-->>NAV: 13. 302 a finish URI /familiar/callback con hash e interact_ref
  RO->>NAV: 14. copia la URL final y la pega en /familiar/completar
  NAV->>C: 15. POST /familiar/completar con la URL
  alt result igual a grant_rejected
    C-->>NAV: permiso rechazado, no se llama a continue
  else hash e interact_ref presentes
    Note over C: 16. verifica hash - base64 de sha256 sobre nonce, finish, interact_ref y authServer separados por salto de línea, admite variante con barra final
    C->>SAS: 17. grant.continue con interact_ref - POST continuation request
    SAS-->>C: 18. 200 access_token value, manage y access
    Note over C: guarda permiso en datos.json - token, manage URL y tope
  end
```

## Coherencia con la documentación

La implementación coincide con la documentación en estos puntos:

- **Actores.** El Client firmante está en el servidor y las ASE tienen sus roles separados ([Client keys](https://openpayments.dev/identity/client-keys/)).
- **Grants.** Los de incoming payment y quote son no interactivos; el de outgoing payment es interactivo ([Authorization](https://openpayments.dev/concepts/auth/)).
- **Orden de recursos.** Incoming payment en el recipient's ASE; quote y outgoing payment en el sender's ASE ([Open Payments flow](https://openpayments.dev/concepts/op-flow/)).
- **Grant reutilizable.** Lleva `limits` e `interval` ([Recurring subscription](https://openpayments.dev/guides/recurring-subscription-incoming-amount/)).
- **Rotación.** Se rota con `token.rotate` sobre la manage URL ([Rotate token](https://openpayments.dev/sdk/token-rotate/)).
- **Detección del pago.** Se hace por polling, que es el único mecanismo para un Client externo, porque los webhooks de Rafiki van a la ASE ([Rafiki webhooks](https://rafiki.dev/integration/requirements/webhook-events/)).

Los riesgos que señalan las notas, en orden de prioridad:

| Riesgo | Qué dice la fuente | Acción sugerida |
|---|---|---|
| Codificación del hash | La documentación dice "Base64 with no padding"; RFC 9635 exige base64 URL-safe sin padding; el ejemplo de la documentación y Rafiki usan `digest('base64')` estándar con `=` ([Hash verification](https://openpayments.dev/identity/hash-verification/); [RFC 9635](https://www.rfc-editor.org/rfc/rfc9635); [Rafiki routes](https://raw.githubusercontent.com/interledger/rafiki/main/packages/auth/src/interaction/routes.ts)) | Normalizar los dos lados (`-_`→`+/`, quitar `=`) antes de comparar, con comparación de tiempo constante. Leer `hash` con `URLSearchParams` para que `+` no se convierta en espacio |
| `result=grant_invalid` | Rafiki lo devuelve si la interacción no se aprobó ni se rechazó ([Rafiki routes](https://raw.githubusercontent.com/interledger/rafiki/main/packages/auth/src/interaction/routes.ts)) | Ya cubierto: `completarPermiso` trata cualquier `result` como fallo y no llama a continue |
| "Inactive Token" también significa revocado | El Resource server responde 403 "Inactive Token" si el token no está activo, sea por caducidad (600 s por defecto) o por revocación ([Rafiki middleware](https://raw.githubusercontent.com/interledger/rafiki/main/packages/backend/src/open_payments/auth/middleware.ts)) | Si `token.rotate` falla (`invalid_rotation`/401), invalidar el permiso y pedir un grant nuevo, sin reintentar en bucle |
| Rotación concurrente | La rotación invalida el valor anterior ([RFC 9635](https://www.rfc-editor.org/rfc/rfc9635)) | `conRotacion` ya persiste el nuevo `value` y `manage`; falta rotar en single-flight por permiso |
| No se guarda el nuevo `continue` | La respuesta de continuation trae un `continue` nuevo, que hace falta para `grant.cancel` ([Future payments](https://openpayments.dev/guides/outgoing-grant-future-payments/); [Grant revoke](https://openpayments.dev/sdk/grant-revoke/)) | Guardar `continue.uri` y `continue.access_token` para poder desvincular. Leer también el `access` concedido, que puede diferir del pedido |
| `expiresAt` de 60 s | Pasado `expiresAt`, la ASE "MUST reject any further payments"; el ejemplo del SDK usa 10 minutos ([resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)) | Los 60 s tienen que cubrir los dos toques **y** el envío ILP, no solo la creación del outgoing payment. Comprobar también `quote.expiresAt` antes de confirmar |
| GET de incoming payment sin autenticar | La vista `public-incoming-payment` solo trae `receivedAmount` y `authServer`, sin `completed` ([resource-server.yaml](https://github.com/interledger/open-payments-specifications/blob/main/openapi/resource-server.yaml)) | Mantener el GET autenticado con el token del comercio. Si alguna vez se usa la vista pública, comparar con el `incomingAmount` guardado |
| Outgoing payment sin quote | La spec 1.4.0 permite `incomingPayment` + `debitAmount` sin quote ([Resources](https://openpayments.dev/concepts/resources/)) | No usarlo: sin quote no se conoce el coste real antes de compararlo con el tope |
| `cardAuthorization` | Propiedad opcional del esquema `access-outgoing`, sin semántica documentada en lo revisado ([auth-server.yaml](https://raw.githubusercontent.com/interledger/open-payments-specifications/main/openapi/auth-server.yaml)) | Investigarla aparte: podría ser la vía oficial para pagos con tarjeta presencial |

## Conclusión

El diagrama deja claro algo que el código no muestra: **todo el peso de la confianza está en el Client**. El servidor Node.js tiene a la vez el token del comercio y el token interactivo del pagador. La documentación lo permite, pero eso convierte a `datos.json` en el activo más sensible del sistema. El toque NFC no tiene valor criptográfico para Open Payments, porque es solo una señal de nuestra app para que el Client use un grant que ya existe. El único control de gasto que impone la ASE es el `debitAmount` semanal. La biometría y el doble toque son controles propios, fuera del protocolo. Si se quiere que el pago presencial tenga respaldo del protocolo y no solo de nuestra app, el siguiente paso es investigar `cardAuthorization`.

En lo operativo, los fallos más probables en una demo no están en el flujo feliz, que sigue el Path 1 oficial. Están en tres puntos concretos: la ventana de 60 s, la verificación del hash con padding y la revocación confundida con caducidad. Corregirlos cuesta pocas líneas y elimina los tres casos que la documentación oficial no ilustra con diagramas.
