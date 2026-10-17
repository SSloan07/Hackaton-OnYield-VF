# Contexto del proyecto: "Pago al Toque" (pagar y cobrar por NFC con Open Payments)

> Pega este archivo (o las secciones que apliquen) al inicio de un prompt para darle contexto a Claude.
> Estado al 15 de octubre de 2026.

## Qué es

Prototipo de viabilidad, desechable. **Una sola app Android con dos modos** que se eligen arriba con un selector grande "Pagar | Cobrar":

- **Modo Pagar** (barra morada): el celular funciona como tarjeta por NFC (Host Card Emulation), previa autorización con huella o bloqueo de pantalla. La cuenta también paga con sus tarjetas físicas registradas.
- **Modo Cobrar** (barra azul): el celular es un datáfono. El incoming payment se crea en la wallet de **la cuenta de ese celular** y la voz dice su nombre de negocio.

Los pagos se hacen con **Open Payments de Interledger** contra el **test wallet**. Cada cuenta que paga necesita un **permiso semanal con tope** (grant interactivo) aprobado una vez por el dueño de la wallet.

**Verificado con pagos reales:** cobro con tarjeta en el datáfono (varios de $1 y $5), renovación automática del token, permiso con alcance completo, migración a cuentas. **Pendiente:** pago celular → celular (falta un segundo Android).

## Arquitectura

```
[Celular en modo Cobrar] --HTTP por USB (adb reverse tcp:3000)--> [Servidor Node :3000] --Open Payments--> [test wallet]
[Celular en modo Pagar]  --HTTP por USB (adb reverse tcp:3000)-->        ^
        ↑ NFC: HCE (AID F0 "DATAFONO") ↓                                  |
[Navegador: /familiar] --------------------------------------------------┘
```

## Cuentas y datos (test wallet, USD, assetScale 2)

| Cuenta (id) | Wallet address | Negocio | Medios | Permiso |
|---|---|---|---|---|
| Doña Rosa (`fa96746d`) | https://ilp.interledger-test.dev/rosa | vacío → publicName "rosa" | Tarjeta 1 `10305DE4`, Tarjeta 2 `082BAC32` | activo, $200/semana, alcance completo |
| Don Jose (`tienda`) | https://ilp.interledger-test.dev/josep | Tienda Don José | — | no tiene |

- Cliente de Open Payments: https://ilp.interledger-test.dev/app · Key ID `dbf01a7e-59cb-4314-8921-60e866914919` · llave `private(App).key` en la raíz (`PRIVATE_KEY_PATH=../private(App).key`). Nunca leerla, imprimirla ni copiarla.
- Dueños en el test wallet: `rosa` es del usuario "Jumbo S.A."; `josep` de "Jaider España". Solo el dueño puede aprobar un permiso.
- La moneda nunca está fija: se leen `assetCode` y `assetScale` de la wallet. Montos con enteros (BigInt / Long).
- Respaldos de datos: `servidor/datos.respaldo-antes-billetera.json` y `servidor/datos.respaldo-antes-cuentas.json`.

## Entorno

- Windows 11, PowerShell 5.1 (ojo: lee los .ps1 como ANSI; para enviar texto con tildes al servidor usar Node). Proyecto en `C:\Users\diego\AndroidStudioProjects\NFC`.
- Samsung Galaxy A53 (SM-A536E), Android 16, serial `R5CTB2FBZFP`, tiene bloqueo de pantalla. Tiene instalada la app con la cuenta **Don Jose** y se usa en modo Cobrar. Hay un emulador `emulator-5554` sin NFC: **nunca instalar ahí**.
- adb: `$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe`. Node 24. Android: Kotlin + Compose + Material 3, AGP 9, minSdk 24, targetSdk 37, un solo módulo `:app`, paquete `com.example.nfc`, nombre visible "Pago al Toque". Sin dependencias añadidas a la plantilla.

```powershell
cd C:\Users\diego\AndroidStudioProjects\NFC\servidor; npm start      # servidor (ventana visible)
node probar.mjs; node --test                                           # credenciales y pruebas
cd C:\Users\diego\AndroidStudioProjects\NFC
.\gradlew.bat :app:assembleDebug :app:testDebugUnitTest
$env:ANDROID_SERIAL = "R5CTB2FBZFP"; .\gradlew.bat :app:installDebug
& $adb -s R5CTB2FBZFP reverse tcp:3000 tcp:3000
```

## Servidor (`/servidor`, Express 5, módulos ES, `@interledger/open-payments` 7.4)

| Archivo | Contenido |
|---|---|
| `servidor.mjs` | Rutas: familiar, cuentas, datáfono, cobros, historial; manejador de errores; `X-HTTP-Method-Override` |
| `billetera.mjs` | Rutas `/api/billetera/*` del celular en modo Pagar (identificado con `?dispositivo=`) |
| `permisos.mjs` | Estado del permiso (`activo`/`pendiente`/`sin_permiso`/`no_valido`) y solicitud del grant interactivo |
| `medios.mjs` | Lógica pura: migración, cuentas/medios, `resolverMedio`, `esMismaCuenta`, historial |
| `op.mjs` | SDK de Open Payments con log por etapas ① ② ③ ④ y `conRotacion` |
| `util.mjs`, `datos.mjs`, `config.mjs`, `paginas.mjs`, `probar.mjs` | Montos/hash, persistencia+migración, `.env`, HTML del familiar, prueba de credenciales |
| `*.test.mjs` | 19 pruebas (`node --test`) |

**Modelo** (`datos.json`): `cuentas[]` {id, nombre, walletAddress, negocio, publicName, assetCode, assetScale}; `medios{uid}` {cuentaId, tipo tarjeta|celular, nombre, bloqueada, ultimoUso} (celular = `CEL:<dispositivoId>`); `permisos{cuentaId}`; `cobros[]` {cuentaCobroId (quién cobra), pagadorId, uid, estado…}. La migración (idempotente) pasó `clientes`→`cuentas`, `tarjetas`→`medios`, `clienteId`→`cuentaId`/`pagadorId` y creó la cuenta `tienda` desde `TIENDA_WALLET_ADDRESS` (ya opcional en `.env`).

**Rutas principales:** `POST/GET/PATCH /api/cuentas[/:id]`, `GET /api/cuentas/:id/historial`, `GET /api/config?cuenta=`, `POST /api/cobros {monto, cuentaId}`, `POST /api/cobros/:id/tarjeta|confirmar|cancelar`, `GET /api/cobros/:id`, `GET /api/cobros?cuenta=`, `GET|POST /api/tarjetas`, `/api/billetera/{vincular, dispositivos/:id, resumen, movimientos, medios, cobro-pendiente, permiso/solicitar}`, `/familiar`, `/familiar/aprobar|callback|completar`.

**Errores:** `TARJETA_NO_REGISTRADA`, `SIN_PERMISO`, `TOPE_SUPERADO`, `COBRO_VENCIDO`, `TARJETA_DISTINTA`, `MEDIO_BLOQUEADO`, `MISMA_CUENTA`, `NO_VINCULADO`, `MEDIO_YA_REGISTRADO`, `CUENTA_NO_ENCONTRADA`, `WALLET_INVALIDA`, `MONTO_INVALIDO`, `CANCELADO`, `PAGO_FALLIDO`, `TIEMPO_AGOTADO`, `ERROR_OPEN_PAYMENTS`.

## App (`/app/src/main/java/com/example/nfc`)

| Archivo | Contenido |
|---|---|
| `MainActivity.kt` | Shell: crear cuenta, selector de modo, pestañas, NFC por modo, voz, vibración, BiometricPrompt, Ajustes |
| `ModoCobrar.kt` | Datáfono: marcar → acerque (60 s) → pregunta → segundo toque → pagando → pagado/error; Registrar tarjeta; diagnóstico |
| `ModoPagar.kt` | Inicio (disponible, aprobar permiso en el navegador, autorizar con huella), Tarjetas, Agregar tarjeta, Mi permiso, overlay "pago en curso" |
| `PestanaHistorial.kt` | Pagos hechos (− rojo) y cobros recibidos (+ verde) con totales de hoy |
| `PagoHceService.kt`, `ProtocoloHce.kt`, `Estado.kt` | HCE, protocolo puro, preferencias y autorización en memoria |
| `ServidorApi.kt`, `MontoUtil.kt`, `NfcLectorEscritor.kt`, `NdefUtil.kt`, `Componentes.kt`, `Entorno.kt` | HTTP, montos/palabras, lectura NFC (celular primero, luego tarjeta), NDEF, UI, interfaz con los modos |

Pruebas: `MontoUtilTest`, `NdefUtilTest`, `ProtocoloHceTest` (incluye "HCE responde solo en Pagar y autorizado").

**NFC por modo:** Cobrar = reader mode activo, sin servicio preferido. Pagar = `setPreferredService` y reader mode apagado (salvo "Agregar tarjeta" y "Leer"). El servicio HCE además revisa el modo guardado: fuera de Pagar o sin autorización responde `6A 82`.

**Protocolo HCE:** AID propietario `F0 44 41 54 41 46 4F 4E 4F` ("F0"+"DATAFONO"), categoría other, `requireDeviceUnlock=true`. Respuesta al SELECT: `v1|CEL:<dispositivoId>|<wallet>` + `90 00`. El que cobra hace primero ese SELECT si el tag tiene IsoDep; si falla, lee como tarjeta (UID + NDEF).

**Autorización:** la huella no puede pedirse durante el toque (dura milisegundos y el datáfono espera 1.5 s). Se autoriza antes (BiometricPrompt BIOMETRIC_STRONG o DEVICE_CREDENTIAL, API del sistema) y queda una ventana de 60 s, que se extiende mientras el cobro espera la confirmación y se revoca al terminar. Un toque sin autorizar hace que la app pida la huella.

## Lecciones aprendidas

- El token del permiso vence a los ~10 min: el test wallet responde **403 "Inactive Token"** (no 401). `conRotacion` rota con 401 o 403 Inactive Token. **Nunca usar el token del permiso desde scripts de prueba**: rotar invalida el token guardado por el servidor (pasó una vez y hubo que reaprobar).
- Solo el dueño de la wallet puede aprobar el permiso; con otra sesión el test wallet va a `/no-access` y **rechaza el grant automáticamente**; reabrir el enlace da `invalid_interaction`.
- Un grant con finish por redirect no se puede consultar por polling: sin `interact_ref` no hay token.
- Incoming no interactivo con `list-all` exige interacción; con `list`/`read` hace falta `identifier`.
- **Rafiki filtra los outgoing por cliente aunque el token tenga `list-all`** (los incoming sí respetan `list-all`): no se pueden ver pagos hechos fuera de esta app ni depósitos. Tampoco hay forma de leer el saldo por Open Payments.
- `HttpURLConnection` no admite PATCH: la app manda POST con `X-HTTP-Method-Override`.
- `adb reverse` se pierde al reconectar el cable → "Sin conexión con el servidor".
- La tarjeta `10305DE4` es IsoDep + NfcA: el datáfono le intenta el SELECT del celular, falla y la lee como tarjeta (verificado).
- Android 16 trae un servicio "Etiqueta incorporada" que responde al AID NDEF `D2760000850101` (por eso se usa un AID propietario).

## Reglas de trabajo

- Prototipo: cambios mínimos, sin dependencias nuevas salvo imprescindibles (justificarlas). Todo en español.
- No tocar la llave. Nunca `makeReadOnly`. No hacer git push ni publicar. Trabajar solo en esta carpeta.
- Instalar solo en celulares físicos con `-s <serial>`, nunca en el emulador.
- Tras cambiar el servidor: reiniciarlo y revisar `adb reverse` de cada celular. Tras cambiar la app: compilar, pruebas, `installDebug`.

## Documentos

- `PRUEBA_DATAFONO.md`: guía de prueba de los dos modos y entre dos celulares, con tabla de resultados.
- `PRUEBAS_NFC.md`: prueba NFC original (lectura, escritura, HCE con iPhone).
