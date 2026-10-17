# Prueba de la app "Pago al Toque" (modos Pagar y Cobrar) con Open Payments

Prototipo de viabilidad. Una sola app Android con dos modos que se eligen arriba con un selector grande:

- **Modo Pagar** (barra morada): el celular funciona como tarjeta por NFC (HCE), previa autorización con huella o bloqueo de pantalla. También se pagan con las tarjetas físicas registradas.
- **Modo Cobrar** (barra azul): el celular es un datáfono; el dinero llega a la wallet de **la cuenta de ese celular**, y la voz dice su nombre de negocio.

El servidor del PC es el único que tiene la llave y habla con Open Payments. Cada celular llega al servidor por USB con `adb reverse`.

## Cuentas actuales

| Cuenta | Wallet address | Negocio (cuando cobra) | Medios de pago |
|---|---|---|---|
| Doña Rosa (`fa96746d`) | https://ilp.interledger-test.dev/rosa | (vacío → "rosa", el publicName de su wallet) | Tarjeta 1 (`10305DE4`), Tarjeta 2 (`082BAC32`) · permiso semanal activo |
| Don Jose (`tienda`) | https://ilp.interledger-test.dev/josep | Tienda Don José | — · sin permiso (solo cobra) |

El Samsung (`R5CTB2FBZFP`) tiene la cuenta de Don Jose y se usa en **modo Cobrar** como datáfono.

## 0. Antes de empezar

```powershell
# Servidor (dejarlo en una ventana visible: ahí se ven las etapas ① ② ③ ④)
cd C:\Users\diego\AndroidStudioProjects\NFC\servidor
node probar.mjs        # ✔ Las credenciales funcionan
npm start
```

En **cada** celular conectado por USB (se pierde al reconectar el cable):

```powershell
$adb = "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe"
& $adb devices                                   # anota los seriales
& $adb -s <SERIAL> reverse tcp:3000 tcp:3000
cd C:\Users\diego\AndroidStudioProjects\NFC
$env:ANDROID_SERIAL = "<SERIAL>"; .\gradlew.bat :app:installDebug
```

Requisitos del celular que **paga** con el celular: NFC encendido, HCE (casi todos los Android con NFC), **un bloqueo de pantalla configurado** (PIN, patrón o huella) y la pantalla desbloqueada.

## 1. Crear la cuenta (primera vez en cada celular)

1. Abre **Pago al Toque**: aparece "Crear mi cuenta".
2. Escribe tu nombre, tu wallet address y, opcional, el nombre del negocio → **Crear mi cuenta**.
3. Si la wallet ya tenía cuenta (por ejemplo `.../rosa`), se recupera con sus tarjetas, su permiso y su historial ("Bienvenido de nuevo").

## Prueba A · Modo Cobrar con tarjeta (un solo celular)

1. En el Samsung, toca **Cobrar** arriba: la barra se pone azul y dice "Modo Cobrar · Tienda Don José".
2. Marca **$5.00** → **Cobrar**.
3. Acerca la tarjeta `10305DE4` de Doña Rosa. Voz: "Doña Rosa, ¿pagar cinco dólares a Tienda Don José?".
4. Retírala y acércala otra vez: vibración doble, "Transfiriendo cinco dólares…", "Pago recibido".
5. Pestaña **Historial**: aparece "+ $5.00 USD · Cobro a Doña Rosa · Pagado".
6. En el test wallet: Doña Rosa baja $5.00 y Don José sube $5.00.

## Prueba B · Modo Pagar (un solo celular, sin cobrar todavía)

1. En el celular de la clienta (cuenta Doña Rosa), toca **Pagar**: barra morada "Modo Pagar · Doña Rosa".
2. **Inicio** muestra "Disponible esta semana", tope, gastado y fecha de renovación.
3. Si no hay permiso: botón grande **Aprobar mi permiso semanal** → se abre el test wallet **en el navegador del celular** → iniciar sesión con la cuenta dueña de la wallet → aceptar → la página dice "Permiso activo" → volver a la app: Inicio muestra el disponible.
4. **Autorizar pago con huella** → huella o PIN → "✔ Pago autorizado · Acerque el celular al datáfono · 60 s".
5. **Tarjetas**: lista el celular y las tarjetas; renombrar, bloquear/desbloquear y eliminar (con confirmación). **Agregar tarjeta** lee una tarjeta (solo mientras esa pantalla está abierta).
6. **Historial**: "− $5.00 USD · Pago a Tienda Don José" en rojo.

## Prueba C · Entre dos celulares (requiere un segundo Android)

Celular **A** (Samsung, Don Jose) en **modo Cobrar**; celular **B** (Doña Rosa) en **modo Pagar**, con su permiso activo, pantalla desbloqueada y `adb reverse` hecho.

1. **Pago con el celular:** en B toca **Autorizar pago con huella**. En A marca $5 → Cobrar. Acerca la parte de atrás de B a la de A.
   - B vibra y muestra "Tienda Don José le cobra $5.00 USD. Acerque el celular otra vez para confirmar" (con voz).
   - A dice "Doña Rosa, ¿pagar cinco dólares a Tienda Don José? Acerque el celular otra vez para confirmar."
   - Retira B y acércalo otra vez → A: "Pago recibido"; B: "Pago hecho, cinco dólares".
   - Aparece en Historial de A (+) y de B (−), y en http://localhost:3000/familiar.
2. **Sin autorizar:** en B no autorices; acerca B a A durante un cobro. A no lo reconoce como celular (lo trata como tarjeta desconocida) y B pide la huella automáticamente. Autoriza y vuelve a acercarlo.
3. **Celular bloqueado:** en B → Tarjetas → **Bloquear** su celular. Cobro en A → error `MEDIO_BLOQUEADO`: A dice "Este celular está bloqueado" y B muestra el error. Desbloquéalo después.
4. **Un celular cobrando no paga:** pon B también en **modo Cobrar**; acércalo a A durante un cobro. A no lo reconoce (HCE desactivada en modo Cobrar) y sigue aceptando tarjetas.
5. **Medio distinto:** primer toque con la tarjeta `10305DE4`, segundo con B → A: "Es otra tarjeta…" (regla `TARJETA_DISTINTA`), no cobra.
6. **Misma cuenta:** con B en modo **Cobrar** (cuenta Doña Rosa), cobra $1 y acerca la tarjeta `10305DE4` (también de Doña Rosa) → error `MISMA_CUENTA`: "Es su propia cuenta".

## Errores comunes

- **"Sin conexión con el servidor"**: el servidor no corre o falta `adb reverse tcp:3000 tcp:3000` en ESE celular (se pierde al reconectar el cable).
- **El celular B no paga y el A lo toma por tarjeta desconocida**: B no está en modo Pagar, no autorizó (huella) o la autorización venció (60 s; se extiende mientras haya un cobro esperando confirmación).
- **"Configure un bloqueo de pantalla"**: sin PIN/patrón/huella no se puede autorizar el pago con el celular.
- **Android pregunta qué app usar para pagar o responde otra app (Google Wallet, Samsung Wallet)**: la app usa un AID propio (`F0 44 41 54 41 46 4F 4E 4F`, categoría "other") y se pide como servicio preferido mientras está abierta en modo Pagar, así que no debería pasar. Si pasa: mantén la app abierta en primer plano en modo Pagar; en Ajustes del sistema → Conexiones → NFC → Pagos sin contacto, no hace falta elegirla como predeterminada (eso es solo para la categoría "payment"). Si otra app de pagos insiste, ciérrala o desactiva temporalmente su opción de "pagar con el celular desbloqueado".
- **No confirma con el segundo toque**: hay que retirar y volver a acercar; si lo dejas quieto, Android no lo vuelve a detectar.
- **Permiso: "Access to the given resource is forbidden" o `/no-access`**: el test wallet tiene abierta otra cuenta (no la dueña de la wallet) y además **rechaza el permiso automáticamente**. Cerrar sesión, entrar con la cuenta correcta y pedir un enlace nuevo.
- **Permiso: `invalid_interaction`**: el enlace ya se había usado; pedir uno nuevo y abrirlo una sola vez.
- **"El permiso ya no es válido"**: el token no se pudo renovar; aprobar el permiso de nuevo.
- **Movimientos de fuera de la app**: los pagos hechos desde el test wallet y los depósitos no salen en el Historial (Rafiki solo deja listar los pagos hechos por esta app).
- **Tarjetas bancarias**: suelen dar un UID distinto en cada lectura; usa NTAG215, Cívica, o escribe la wallet en la tarjeta.

## Resultados

| Prueba | Funcionó (Sí/No) | Celular(es) | Notas |
|---|---|---|---|
| 0 · probar.mjs y servidor | | | |
| 1 · Crear / recuperar cuenta | | | |
| A · Cobrar con tarjeta 10305DE4 | | | |
| A · Historial muestra el cobro (+) | | | |
| A · Saldos en el test wallet | | | |
| B · Modo Pagar: disponible y permiso | | | |
| B · Aprobar permiso desde el navegador del celular | | | |
| B · Autorizar con huella | | | |
| B · Tarjetas: renombrar / bloquear / eliminar / agregar | | | |
| C1 · Pago celular → celular | | | |
| C2 · Sin autorizar: pide huella | | | |
| C3 · Celular bloqueado: MEDIO_BLOQUEADO | | | |
| C4 · Celular en modo Cobrar no paga | | | |
| C5 · Tarjeta y luego celular: no cobra | | | |
| C6 · Misma cuenta: MISMA_CUENTA | | | |
