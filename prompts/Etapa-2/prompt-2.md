# Base del datáfono y Open Payments

## Prompt

CONTEXTO
Este proyecto (C:\Users\diego\AndroidStudioProjects\NFC) es una app Android de prueba de NFC (Kotlin + Jetpack Compose) que ya lee etiquetas, escribe NDEF y emula HCE. Ahora quiero un PROTOTIPO DE VIABILIDAD desechable: convertir el celular Android en un DATÁFONO que cobra con una tarjeta NFC usando Open Payments de Interledger contra el test wallet. No es para producción ni para competir; quiero saber si el flujo completo funciona de punta a punta.

DATOS
- Wallet address de la app (cliente de Open Payments): https://ilp.interledger-test.dev/app
- Key ID: [OMITIDO]
- Llave privada: C:\Users\diego\AndroidStudioProjects\NFC\private.key
- Wallet address de la tienda: https://ilp.interledger-test.dev/josep
- Nombre de la tienda: Tienda Don José
- Wallet address de la clienta de prueba: https://ilp.interledger-test.dev/rosa, nombre "Doña Rosa"
- Moneda: las tres wallets están en USD (dólares estadounidenses, assetScale 2). Aun así, no la escribas fija en el código: lee siempre assetCode y assetScale de la wallet address y conviértelos con esos valores.
- Montos rápidos del datáfono: 5, 12 y 44 dólares. El teclado permite centavos (hasta 2 decimales).
- Tope semanal por defecto del permiso: 200 dólares.

ARQUITECTURA
1. Servidor Node.js (carpeta nueva /servidor en la raíz del proyecto) con Express y el SDK oficial @interledger/open-payments. Es el único que tiene las llaves y habla con Open Payments.
2. App Android = datáfono del tendero. Lee la tarjeta del cliente por NFC y habla con el servidor por HTTP.
3. Página web del "familiar", servida por el mismo servidor, para registrar clientes y aprobar una vez el permiso semanal de pago.
La conexión del celular al servidor va por USB con: adb reverse tcp:3000 tcp:3000, así la app usa http://localhost:3000.

ANTES DE ESCRIBIR CÓDIGO DE OPEN PAYMENTS
Consulta la documentación oficial para usar la API actual del SDK de Node y no adivinar nombres ni parámetros:
- https://openpayments.dev/sdk/before-you-begin
- https://openpayments.dev/sdk/grant-create-incoming
- https://openpayments.dev/sdk/grant-create-quote
- https://openpayments.dev/sdk/grant-create-outgoing
- https://openpayments.dev/sdk/grant-continue
- https://openpayments.dev/sdk/incoming-create/
- https://openpayments.dev/sdk/quote-create/
- https://openpayments.dev/sdk/outgoing-create/
- https://openpayments.dev/sdk/token-rotate/
- https://openpayments.dev/guides/outgoing-grant-future-payments/
- https://openpayments.dev/identity/hash-verification/
- Snippets de Node: https://github.com/interledger/open-payments/tree/main/snippets/node

SERVIDOR (/servidor)
- Node + Express, módulos ES. Configuración en /servidor/.env (APP_WALLET_ADDRESS, KEY_ID, PRIVATE_KEY_PATH, TIENDA_WALLET_ADDRESS, TIENDA_NOMBRE, PORT=3000). Crea .env.example sin secretos y un .gitignore que excluya .env, private.key, node_modules y datos.json.
- Persistencia simple en /servidor/datos.json: clientes, tarjetas, permisos y cobros.
- Cliente autenticado con createAuthenticatedClient.
- Registra en consola cada llamada a Open Payments con su etapa: [① SOLICITUD], [② AUTORIZACIÓN], [③ EJECUCIÓN], [④ FINALIZACIÓN], mostrando montos en dólares legibles (ej. $44.00 USD) además del valor en unidades mínimas.

Flujo A · Permiso del familiar (una sola vez por cliente)
- GET /familiar: página HTML sencilla, grande y legible, en español. Lista los clientes con el estado de su permiso (tope, cuánto lleva gastado esta semana y cuánto queda), permite agregar un cliente (nombre + wallet address) y tiene un botón "Aprobar permiso semanal" con un tope editable (por defecto 200 dólares).
- Al aprobar: obtener la wallet address del cliente y pedir en su authServer un grant INTERACTIVO de outgoing-payment con actions create/read/list, identifier = wallet del cliente, limits.debitAmount = tope (en unidades mínimas según assetScale, con assetCode de la wallet) e interval = R/<fecha actual ISO>/P1W. interact: start ["redirect"], finish { method "redirect", uri http://localhost:3000/familiar/callback?cliente=<id>, nonce aleatorio }. Redirigir el navegador a interact.redirect.
- GET /familiar/callback: verificar el hash según la documentación, continuar el grant con interact_ref y guardar access_token.value, access_token.manage y la fecha. Mostrar "Permiso activo" y volver a /familiar.
- Si un token vence (401), rotarlo con token.rotate y reintentar una vez.

Flujo B · Cobro en el datáfono
- GET /api/config → nombre de la tienda, assetCode y assetScale de la tienda.
- GET /api/clientes → lista de clientes (para registrar tarjetas).
- POST /api/tarjetas { uid, clienteId } → asocia una tarjeta (UID en hex) a un cliente.
- POST /api/cobros { monto } (monto en dólares, ej. 44 o 12.50) → [①] grant no interactivo de incoming-payment en el authServer de la tienda y crear el incoming payment con incomingAmount convertido a unidades mínimas con assetScale (44.00 USD → value "4400") y expiresAt a 60 segundos. Usa aritmética entera para no tener errores de redondeo. Devuelve id del cobro, monto formateado y vencimiento.
- POST /api/cobros/:id/tarjeta { uid, walletAddress? } → [②] resolver el cliente: primero por walletAddress si la tarjeta trae una en NDEF, si no por UID registrado. Errores claros: TARJETA_NO_REGISTRADA, SIN_PERMISO, COBRO_VENCIDO. Pedir grant no interactivo de quote en el authServer del cliente y crear el quote con receiver = id del incoming payment y method "ilp". Si el debitAmount supera lo que queda del tope semanal, devolver TOPE_SUPERADO con el saldo restante. Devuelve nombre del cliente, monto y nombre de la tienda.
- POST /api/cobros/:id/confirmar { uid } → solo si el UID es el mismo de la primera lectura. [③] crear el outgoing payment en la wallet del cliente con el quoteId y el token del permiso guardado. Si Open Payments lo rechaza por límite, devolver TOPE_SUPERADO.
- GET /api/cobros/:id → [④] consultar el incoming payment y devolver estado: esperando_tarjeta, esperando_confirmacion, procesando, pagado, error (con código y mensaje en español).
- GET /api/pagos → historial para la página del familiar.

APP ANDROID (reemplaza las pestañas de prueba por el datáfono; conserva la pestaña "Leer" como diagnóstico en un menú secundario)
- Kotlin + Compose + Material 3. Llamadas HTTP con HttpURLConnection o OkHttp en corrutinas (si agregas OkHttp, justifícalo). Permite tráfico HTTP solo a localhost y 127.0.0.1 con network_security_config.
- Diseño accesible: texto muy grande, alto contraste, botones gigantes, una acción por pantalla.
- Formato de montos: "$44.00 USD" en pantalla.
- Pantalla 1 · Marcar valor: teclado numérico grande con punto decimal, montos rápidos de 5, 12 y 44 dólares y botón "Cobrar".
- Pantalla 2 · "Acerque la tarjeta": activa reader mode y muestra la cuenta regresiva de 60 s. Al leer: UID en hex y, si la tarjeta tiene un registro NDEF URI que empiece por https, tomarlo como wallet address. Llama a /tarjeta.
- Pantalla 3 · Pregunta: "Doña Rosa, ¿pagar $44.00 a Tienda Don José?" y abajo "Acerque la tarjeta otra vez para confirmar". TextToSpeech en español (es-CO; si no existe, es-US o es) con el monto en palabras: "¿Pagar cuarenta y cuatro dólares a Tienda Don José?"; con centavos: "doce dólares con cincuenta centavos". Botón gigante "Cancelar".
- Segundo toque con la misma tarjeta → /confirmar → Pantalla 4 · "Pagando…" consultando /api/cobros/:id cada segundo.
- Pantalla 5 · "Pago recibido" con chulo grande, voz "Pago hecho, cuarenta y cuatro dólares" y vibración larga. Botón "Nuevo cobro".
- Errores en pantalla y en voz, con vibración de tres pulsos cortos: tarjeta no registrada, sin permiso, tope superado (decir cuánto queda en dólares), cobro vencido y sin conexión con el servidor.
- Pantalla de administración "Registrar tarjeta" (desde un menú): acercar la tarjeta, elegir el cliente de la lista y guardar. Opción "Escribir la wallet address en la tarjeta" si la etiqueta acepta NDEF (nunca makeReadOnly).
- Función para convertir montos a palabras en español con pruebas unitarias: 0 → "cero dólares", 1 → "un dólar", 0.01 → "un centavo", 21 → "veintiún dólares", 44 → "cuarenta y cuatro dólares", 12.50 → "doce dólares con cincuenta centavos", 101 → "ciento un dólares", 1000 → "mil dólares", 21000 → "veintiún mil dólares", 1000000 → "un millón de dólares".

PROCESO
1. Lee el proyecto actual.
2. Crea el servidor, instala dependencias con npm y verifica que arranca. Haz una prueba por consola (script /servidor/probar.mjs) que cree un incoming payment de 1 dólar en la tienda y un quote desde la wallet de Doña Rosa contra el test wallet, para confirmar que las credenciales funcionan. Si falla, muéstrame el error exacto y la causa probable antes de seguir.
3. Implementa la app Android. Compila con .\gradlew.bat assembleDebug, corre .\gradlew.bat testDebugUnitTest y corrige todo.
4. Ejecuta adb reverse tcp:3000 tcp:3000 (adb está en $env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe) e instala con .\gradlew.bat installDebug en el Samsung (no en el emulador; si hay varios dispositivos, usa el serial del Samsung).
5. Crea PRUEBA_DATAFONO.md con el paso a paso: arrancar el servidor, abrir http://localhost:3000/familiar en el navegador del PC, agregar a Doña Rosa, aprobar el permiso en el test wallet, registrar la tarjeta en la app, hacer un cobro de $5 y verificar el saldo en el test wallet (baja en Doña Rosa, sube en la tienda). Incluye una sección de errores comunes y una tabla de resultados para llenar.
6. Termina con un resumen: qué funciona verificado, qué no pudiste verificar sin mí y qué falta.

LÍMITES
- Trabaja solo dentro de esta carpeta. No modifiques configuración del sistema ni instales programas globales.
- No subas nada a GitHub ni hagas git push.
- Nunca imprimas ni copies el contenido de private.key en archivos, logs o la consola.
- Nunca uses makeReadOnly.
