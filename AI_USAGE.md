# AI_USAGE — Pago al Toque

**Pago al Toque** es una app Android que convierte el celular de un pequeño negocio en un punto de cobro por NFC, con pagos reales sobre **Open Payments (Interledger)** en el test wallet.

## 1. Desarrollo local y organización de la entrega

- **El equipo trabajó en local y olvidó registrar los commits durante el desarrollo.** Ahora organizamos los cambios por unidades, revisamos sus pruebas y documentamos su incorporación al repositorio. Esta secuencia de commits registra la incorporación del trabajo local; no representa su cronología original de desarrollo.
- **Herramienta de IA:** Claude Code (Anthropic), usada en la terminal sobre el repositorio.
- **Rol de la IA:** herramienta de implementación e investigación. El equipo definió el producto, escribió las especificaciones detalladas de cada fase, probó en hardware real, detectó los errores y tomó las decisiones.

## 2. Qué hizo el equipo y qué hizo la IA

| Área | Equipo | IA (Claude Code) |
|---|---|---|
| Idea, problema y público objetivo (negocios pequeños, personas mayores, sin smartphone, con discapacidad visual) | ✔ | |
| Flujo de "doble toque" para confirmar pagos | ✔ | |
| Mockups y prototipo en Figma | ✔ | |
| Especificación de cada fase: pantallas, rutas del servidor, códigos de error, reglas de montos, límites de seguridad | ✔ | |
| Arquitectura (servidor con las llaves, app sin secretos, permiso semanal GNAP) | ✔ | Ajustes técnicos |
| Investigación de Open Payments | Dirección y validación | Lectura de la documentación y síntesis |
| Código Android (Kotlin, Compose) y servidor (Node.js) | Revisión y decisiones | Implementación |
| Pruebas en hardware real (Samsung Galaxy A53, tarjetas NFC, pagos en el test wallet) | ✔ | Compilación y pruebas unitarias |
| Detección y diagnóstico de errores (saldos, falso "tope superado", tarjetas duplicadas, aprobación remota del permiso) | ✔ | Corrección |
| Configuración de wallets, llaves y permisos en el test wallet | ✔ | |
| Documentación técnica y diagrama | Revisión | Redacción |

## 3. Proceso

1. **Planeación del equipo:** idea, flujo de doble toque y mockups en Figma.
2. **Viabilidad de NFC:** lectura, escritura de etiquetas y emulación HCE.
3. **Datáfono con Open Payments:** servidor Node.js con el SDK oficial y cobro con tarjeta NFC usando un permiso semanal.
4. **Pruebas reales y corrección de errores** reportados por el equipo.
5. **Billetera** y luego **una sola app con dos modos** (Pagar y Cobrar).
6. **Diagrama de arquitectura** coherente con la documentación de Open Payments.
7. **Interfaz final** a partir del prototipo de Figma, en español e inglés.

## 4. Límites y buenas prácticas

- La IA nunca leyó ni copió la llave privada; se le prohibió de forma explícita.
- Las llaves y tokens viven solo en el servidor. La app no contiene secretos.
- Se exigió verificar contra la documentación oficial y no afirmar pruebas no ejecutadas.
- Montos con enteros (unidades mínimas), sin punto flotante.
- Las funciones simuladas están separadas y documentadas.

## 5. Prompts

Los prompts se reproducen **de forma literal**, incluidos los errores de tipeo. Solo se ocultó el Key ID (`[OMITIDO]`); `[Imagen]` indica una captura adjunta.

Se incluyen solo los prompts principales. Se omitieron los mensajes cortos de una línea (instrucciones operativas como "instálalo en el Samsung" o "reinicia el servidor" y reportes rápidos de errores durante las pruebas) y un borrador duplicado del prompt 2. Están en los registros de sesión de Claude Code si se requieren.

### Fase 1 · Viabilidad de NFC

**Prompt 1**

~~~~text
CONTEXTO
Este es un proyecto de PRUEBA DE VIABILIDAD, desechable. Es una app Android vacía creada con la plantilla "Empty Activity" de Android Studio (Kotlin + Jetpack Compose + Material 3). Quiero comprobar UNA sola cosa: si el NFC funciona como lo necesito entre mi Android y mi iPhone. No agregues nada que no sea NFC: nada de red, backend, pagos, Open Payments, base de datos, voz ni diseño elaborado.

QUÉ QUIERO COMPROBAR
1. Que el Android pueda LEER etiquetas NFC: stickers NTAG215 y tarjetas sin contacto (por ejemplo la tarjeta Cívica del metro o una tarjeta bancaria).
2. Que el Android pueda ESCRIBIR una URL en un sticker NTAG215 vacío.
3. Que el Android pueda EMULAR un sticker NFC (Host Card Emulation) que contenga una URL, y que mi iPhone lo lea solo con acercarlo, sin instalar ninguna app (lectura NFC en segundo plano del iPhone).

LA APP: una sola pantalla con tres pestañas: "Leer", "Escribir" y "Emitir".

Pestaña A · Leer
- Usa NfcAdapter.enableReaderMode en onResume y disableReaderMode en onPause. No uses foreground dispatch.
- Flags: FLAG_READER_NFC_A | FLAG_READER_NFC_B | FLAG_READER_NFC_F | FLAG_READER_NFC_V. NO uses FLAG_READER_SKIP_NDEF_CHECK.
- Al detectar una etiqueta muestra: UID en hexadecimal, lista de tecnologías (tag.techList), si tiene mensaje NDEF, tamaño máximo, si es escribible, y cada registro NDEF decodificado (URI completa con el prefijo resuelto, texto con su idioma, o tipo MIME con los bytes en hex).
- Guarda las últimas 10 lecturas en una lista en memoria, con la hora.
- Vibración corta al leer (permiso VIBRATE).

Pestaña B · Escribir
- Campo de texto con la URL, por defecto https://interledger.org
- Botón "Escribir en el próximo sticker": activa el modo escritura. Al acercar una etiqueta, escribe un NdefMessage con un solo registro NdefRecord.createUri. Si la etiqueta es NdefFormatable (vacía, sin formato), formatéala con ese mensaje.
- Muestra éxito o un error claro: etiqueta no escribible, espacio insuficiente, o etiqueta retirada demasiado rápido (TagLostException).
- NUNCA uses makeReadOnly.

Pestaña C · Emitir (HCE)
- Crea un HostApduService que emule una NFC Forum Type 4 Tag con un archivo NDEF que contenga la URL elegida (por defecto la misma de la pestaña B).
- AID: D2760000850101. Responde a:
  - SELECT por AID: 00 A4 04 00 07 D2760000850101 (con o sin Le 00) → 90 00
  - SELECT del Capability Container: 00 A4 00 0C 02 E103 → 90 00
  - SELECT del archivo NDEF: 00 A4 00 0C 02 E104 → 90 00
  - READ BINARY: 00 B0 [offset 2 bytes] [Le] → los bytes pedidos del archivo seleccionado + 90 00
  - Archivo no encontrado → 6A 82. Comando desconocido → 6D 00.
- Capability Container (15 bytes): 00 0F | 20 | 00 3B | 00 34 | 04 06 E1 04 [tamaño máximo NDEF, 2 bytes] 00 FF (solo lectura).
- Archivo NDEF: 2 bytes NLEN (longitud del mensaje) + el mensaje NDEF con un registro URI.
- Manifiesto:
  - <uses-permission android:name="android.permission.NFC"/>
  - <uses-permission android:name="android.permission.VIBRATE"/>
  - <uses-feature android:name="android.hardware.nfc" android:required="false"/>
  - <uses-feature android:name="android.hardware.nfc.hce" android:required="false"/>
  - El servicio con android:exported="true", android:permission="android.permission.BIND_NFC_SERVICE", intent-filter android.nfc.cardemulation.action.HOST_APDU_SERVICE y meta-data android.nfc.cardemulation.host_apdu_service apuntando a res/xml/apduservice.xml, con aid-group category="other" y requireDeviceUnlock="false".
- Comparte la URL actual entre la Activity y el servicio (SharedPreferences o un object singleton).
- Mientras la pestaña Emitir esté activa, DESACTIVA el reader mode; si no, el teléfono lee en vez de emitir. Usa CardEmulation.setPreferredService al entrar en la pestaña y unsetPreferredService al salir o en onPause, para que Google Wallet u otra app no respondan en su lugar.
- Muestra en pantalla, y en Logcat con el tag NFCDEMO, cada APDU recibida y la respuesta enviada, en hex. Lo necesito para depurar qué hace el iPhone.
- Instrucción en pantalla: "Mantén esta pantalla encendida y desbloqueada. Acerca la parte de arriba del iPhone a la parte de atrás de este celular".

ESTADOS Y ERRORES (siempre visibles arriba)
- El dispositivo no tiene NFC → mensaje claro.
- NFC apagado → mensaje y botón que abre Settings.ACTION_NFC_SETTINGS.
- Si no hay soporte HCE (PackageManager.FEATURE_NFC_HOST_CARD_EMULATION), deshabilita la pestaña Emitir y explica por qué.

REQUISITOS TÉCNICOS
- Kotlin, Jetpack Compose y Material 3 de la plantilla. No agregues dependencias salvo que sean imprescindibles, y si agregas alguna explica por qué.
- No cambies minSdk.
- Todo el texto de la interfaz en español, y los comentarios del código también.
- Pocos archivos: MainActivity.kt (pantalla y pestañas), NfcLectorEscritor.kt (leer y escribir), NdefHceService.kt (emulación) y NdefUtil.kt (hex, construcción del CC, archivo NDEF, respuestas APDU).
- Pruebas unitarias de las funciones puras de NdefUtil: construcción del CC, archivo NDEF con NLEN y respuesta a READ BINARY con distintos offsets y longitudes.

PROCESO
1. Lee el proyecto actual antes de cambiar nada.
2. Implementa todo lo anterior.
3. Compila con .\gradlew.bat assembleDebug (estoy en Windows, PowerShell) y corrige los errores hasta que compile limpio. Corre .\gradlew.bat testDebugUnitTest y corrige lo que falle.
4. Si adb devices muestra un celular conectado, instala con .\gradlew.bat installDebug y abre la app con: adb shell am start -n <paquete>/.MainActivity. Si adb no está en el PATH, usa $env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe.
5. Crea PRUEBAS_NFC.md en la raíz con la guía de prueba manual de abajo.
6. Termina con un resumen: archivos creados o cambiados, cómo probar y riesgos conocidos.

LÍMITES
- Trabaja solo dentro de esta carpeta del proyecto. No modifiques configuración del sistema ni variables de entorno, y no instales programas.
- No hagas git push ni publiques nada. No llames servicios externos.
- Nunca uses makeReadOnly.

CONTENIDO DE PRUEBAS_NFC.md
- Prueba 1 · Android lee una tarjeta sin contacto (Cívica o bancaria): pestaña Leer. Esperado: UID y tecnologías (NfcA, IsoDep u otras).
- Prueba 2 · Android escribe y lee un sticker NTAG215: escribir la URL en la pestaña Escribir y luego leerla en Leer. Esperado: aparece la misma URL.
- Prueba 3 · iPhone lee el sticker: iPhone XS o más reciente, pantalla encendida y desbloqueada, sin la cámara ni la billetera abiertas; acercar la parte de arriba del iPhone al sticker. Esperado: notificación para abrir la URL en Safari.
- Prueba 4 · iPhone lee el Android en modo Emitir: mismas condiciones que la Prueba 3. Esperado: notificación con la URL. Diagnóstico con el registro de APDUs: si no llega ningún comando, el iPhone no detecta la emulación; si llega el SELECT del AID pero nunca READ BINARY, revisar el CC; si llega READ BINARY pero no abre nada, revisar el archivo NDEF.
- Prueba 5 · Android intenta leer el iPhone: se espera que NO funcione, porque el iPhone no emula etiquetas para apps de terceros. Solo anotar el resultado.
- Al final, una tabla para llenar con columnas: Prueba | Funcionó (Sí/No) | Modelo de celular | Notas.
~~~~

### Fase 2 · Datáfono con Open Payments

**Prompt 2**

~~~~text
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
~~~~

### Fase 3 · Aprobación remota del permiso

**Prompt 3**

~~~~text
Mi amigo es el dueño de la wallet de Doña Rosa y está en otro computador, así que no puede abrir localhost. Haz este cambio en /familiar:
- Al tocar "Aprobar permiso semanal", NO redirijas el navegador. Muestra el enlace de aprobación (interact.redirect) en un cuadro con un botón "Copiar enlace" y el texto: "Envía este enlace al dueño de la wallet. Que inicie sesión y acepte."
- Agrega debajo un campo "Pega aquí la dirección a la que llegó tu amigo" y un botón "Completar permiso". El servidor debe extraer interact_ref y hash de esa URL y seguir el mismo proceso del callback: verificar el hash, continuar el grant y guardar el token.
- No cambies nada más. Reinicia el servidor cuando termines.
~~~~

### Fase 4 · Billetera y app unificada

**Prompt 4**

~~~~text
# Tarea: app Android "Billetera" — el celular del cliente como wallet y medio de pago NFC

Arriba tienes el contexto completo del proyecto Datáfono NFC con Open Payments. Léelo entero antes de empezar. El flujo del datáfono con tarjetas ya funciona y está verificado con pagos reales en el test wallet: **no lo rompas**.

Todo es **solo Android** (Kotlin + Jetpack Compose + Material 3). No hay versión iOS ni web en esta tarea.

## Objetivo

Crear una segunda app Android, la **Billetera**, para el celular Android de un usuario general (ej. Doña Rosa). Debe permitir:

1. **Consultar su saldo / disponible** de su wallet.
2. **Revisar sus transacciones** (pagos hechos y recibidos).
3. **Pagar acercando su celular al datáfono.** El celular funciona como tarjeta por NFC con Host Card Emulation (HCE), y el flujo es igual al de la tarjeta: primer toque = pregunta, segundo toque = confirmación.
4. **Administrar sus medios de pago**: sus tarjetas físicas registradas y el propio celular (ver, agregar, renombrar, bloquear/desbloquear, eliminar).
5. **Ver y gestionar su permiso semanal**: tope, gastado, restante, cuándo se renueva y solicitar uno nuevo.

Es un prototipo de viabilidad. Haz cambios mínimos y no agregues dependencias salvo que sean imprescindibles (justifícalas). Todo en español.

## Dispositivos

| Papel | Dispositivo | Serial | App |
|---|---|---|---|
| Datáfono (tendero) | Samsung Galaxy A53, Android 16 | `R5CTB2FBZFP` | `:app` (Datáfono) |
| Celular del cliente | Android con NFC | `<SERIAL_CELULAR_CLIENTE>` | `:billetera` (Billetera) |
| Emulador | `emulator-5554` (sin NFC) | — | **No instalar nada aquí** |

- Ambos celulares van conectados por USB al mismo PC, cada uno con su propio `adb -s <serial> reverse tcp:3000 tcp:3000`.
- Usa siempre `-s <serial>` o `$env:ANDROID_SERIAL` para no instalar en el dispositivo equivocado.
- **Si `<SERIAL_CELULAR_CLIENTE>` no está conectado:** haz las fases 1 a 3, pero para probar la Billetera instálala temporalmente en el Samsung (vincular, inicio, movimientos, medios, permiso). Desinstálala del Samsung antes de volver a probar el Datáfono, y deja la fase 4 (HCE de punta a punta) pendiente y documentada.

## Antes de escribir código

1. Revisa el código actual: `/servidor`, `/app`, `NdefUtil.kt` (tiene restos de la prueba HCE con emulación Type 4), `PRUEBAS_NFC.md` y `PRUEBA_DATAFONO.md`.
2. **Verifica en la documentación del SDK `@interledger/open-payments` 7.4 y de Rafiki / test wallet**, no de memoria:
   - **Saldo:** si existe alguna forma de leer el saldo de una wallet por Open Payments. Mi entendimiento es que **no existe**. Si se confirma, no inventes un saldo: muestra "Disponible esta semana" (tope − gastado según la wallet), los movimientos y un texto claro de que el saldo total se consulta en el test wallet.
   - **Pagos hechos:** cómo listar los outgoing payments del cliente con el token del permiso semanal (¿basta `list` o hace falta `list-all`?).
   - **Pagos recibidos:** cómo listar los incoming payments de la wallet del cliente, con un grant no interactivo de incoming-payment (`read`/`list`/`list-all`) sobre la wallet del cliente.
3. Presenta un **plan corto** (archivos a crear o cambiar, rutas nuevas, protocolo HCE, cambios en `datos.json`) y **espera mi aprobación** antes de implementar.

## Diseño propuesto

Ajústalo si al revisar el código encuentras algo mejor, y explícame por qué.

### 1. Estructura del proyecto Android

- Nuevo módulo **`:billetera`** en el mismo proyecto de Android Studio, paquete `com.example.billetera`, nombre visible **"Billetera"**. Usa las mismas versiones que `:app`: Kotlin, Compose, Material 3, AGP 9, minSdk 24, targetSdk 37.
- Reutiliza `MontoUtil` y los componentes de alto contraste (`BotonGigante`, `TextoGrande`, etc.). Cópialos al módulo o crea un módulo común pequeño, lo que sea más simple. Mantén y extiende sus pruebas unitarias.
- Cliente HTTP igual que en `ServidorApi.kt` (HttpURLConnection + corrutinas, sin librerías nuevas), apuntando a `http://localhost:3000`.
- Mismo `network_security_config.xml` (HTTP solo a `localhost` y `127.0.0.1`).
- Permisos: NFC, VIBRATE, INTERNET. Declara `android.hardware.nfc.hce` con `required="true"`. Agrega `<queries>` para TTS.
- Log en Logcat con tag `BILLETERA`.

### 2. Identidad del celular (prototipo, sin autenticación real)

- Al abrir por primera vez aparece la pantalla **"Vincular este celular"**. El usuario elige su nombre de la lista de clientes o escribe su wallet address. La app genera un `dispositivoId` (UUID) y lo guarda en SharedPreferences.
- El servidor registra el celular como un medio de pago más del cliente, con UID `CEL:<dispositivoId>`, tipo `celular` y el nombre del modelo (`Build.MODEL`). Valida que la wallet corresponda a un cliente registrado.
- Ajustes incluye la opción "Desvincular este celular".
- Documenta como limitación conocida que no hay autenticación ni protección contra copia del identificador.

### 3. Pagar con el celular (HCE) con el mismo flujo de la tarjeta

**En la Billetera (celular del cliente):**

- Crea un `HostApduService` con un **AID propietario** (ej. `F0` + "DATAFONO" en ASCII; justifica la elección) en categoría `other`, declarado en `res/xml/apduservice.xml`. Decide y justifica `requireDeviceUnlock`.
- Ante `SELECT AID` responde un payload simple y versionado, por ejemplo `v1|CEL:<dispositivoId>|<walletAddress>` en UTF-8 seguido de `90 00`.
- Responde con un status de error (ej. `6A 82`) si el celular no está vinculado o si "Pagar con este celular" está **desactivado**.
- Con la app en primer plano, usa `CardEmulation.setPreferredService` en `onResume` y `unsetPreferredService` en `onPause`.
- **La Billetera nunca activa reader mode**, porque desactiva la emulación. La única excepción es la pantalla "Agregar tarjeta", y solo mientras esté abierta.
- En cada toque, el servicio vibra brevemente y avisa a la UI (por ejemplo con un `StateFlow` compartido o un broadcast local) si la app está abierta.

**En el Datáfono (cambio mínimo en `:app`):**

- Al detectar un tag con `IsoDep`, primero intenta `SELECT` del AID de la Billetera.
  - Si responde `90 00`, usa `CEL:<dispositivoId>` como `uid` y la wallet del payload como `walletAddress`.
  - Si falla o no hay `IsoDep`, sigue **exactamente** con la lectura actual de tarjeta (UID + NDEF).
- **El resto del flujo no cambia**: `POST /api/cobros/:id/tarjeta` y `POST /api/cobros/:id/confirmar` reciben el mismo `uid` en ambos toques. Así la regla de "misma tarjeta" (`TARJETA_DISTINTA`) funciona igual con celulares, aunque el UID NFC del celular sea aleatorio en cada toque.
- Si el medio es un celular, la pantalla de pregunta y la voz dicen "Acerque el celular otra vez para confirmar".
- "Registrar tarjeta" del datáfono debe reconocer un celular y avisar que se vincula desde la Billetera.

**Feedback en la Billetera durante el cobro:**

- Con la app abierta, consulta cada 1–2 s el cobro pendiente de su dispositivo y muestra los estados en secuencia:
  1. "Tienda Don José le cobra $5.00 USD. Acerque el celular otra vez para confirmar".
  2. "Pagando…".
  3. "Pago hecho" o el error.
- Usa voz (TTS en es-CO, si no es-US, si no es) y vibración, igual que el datáfono.
- **No reemplaces el segundo toque** por un botón de confirmar en la Billetera: el segundo toque sigue siendo la confirmación. La Billetera sí puede mostrar un botón "Cancelar" que llame a `POST /api/cobros/:id/cancelar`.

### 4. Servidor

Agrega rutas nuevas bajo `/api/billetera` sin romper las existentes.

| Ruta | Qué hace |
|---|---|
| `POST /api/billetera/vincular {dispositivoId, walletAddress, nombreDispositivo}` | Registra el celular como medio de pago del cliente |
| `DELETE /api/billetera/dispositivos/:dispositivoId` | Desvincula el celular |
| `GET /api/billetera/resumen?dispositivo=` | Cliente, assetCode/assetScale, tope, gastado esta semana (según la wallet, con respaldo local), restante, fecha de renovación y estado del permiso |
| `GET /api/billetera/movimientos?dispositivo=` | Outgoing e incoming de la wallet del cliente por Open Payments, combinados y ordenados por fecha. Cada uno con contraparte, monto, estado y el medio usado si se conoce por el registro local. Usa el registro local como respaldo si falla |
| `GET /api/billetera/medios?dispositivo=` | Tarjetas y celulares del cliente, con nombre, tipo, estado y último uso |
| `POST /api/billetera/medios {uid, walletAddress?, nombre}` | Agrega una tarjeta física leída por el propio celular del cliente |
| `PATCH /api/billetera/medios/:uid {nombre?, bloqueada?}` | Renombra, bloquea o desbloquea un medio |
| `DELETE /api/billetera/medios/:uid` | Elimina un medio |
| `GET /api/billetera/cobro-pendiente?dispositivo=` | Cobro en curso asociado a ese medio, para el feedback |
| `POST /api/billetera/permiso/solicitar {tope}` | Inicia el mismo grant interactivo que `/familiar/aprobar` y devuelve el enlace `interact.redirect` |

Reglas del servidor:

- Agrega el código de error **`MEDIO_BLOQUEADO`**. En `POST /api/cobros/:id/tarjeta` y `/confirmar`, rechaza con ese error cualquier tarjeta o celular bloqueado. El datáfono lo muestra y lo dice por voz.
- Migra `datos.json` de forma compatible. Las tarjetas existentes (ej. `10305DE4` → Doña Rosa) deben seguir funcionando y quedan con tipo `tarjeta`, un nombre por defecto y `bloqueada: false`.
- Usa `op.conRotacion` en todo lo que use el token del permiso. Recuerda que 403 "Inactive Token" significa token vencido, no tope superado.
- Mantén el log por etapas en la consola. Las consultas frecuentes de la Billetera (`cobro-pendiente`, `resumen`) no deben inundarlo: regístralas solo cuando cambie algo.
- Agrega pruebas en `node --test` para la lógica pura: parseo del payload HCE si lo hace el servidor, migración de datos, combinación y orden de movimientos, y bloqueo de medios.

### 5. Pantallas de la Billetera

Mantén el mismo estilo del datáfono: texto grande, alto contraste, una acción principal por pantalla y voz opcional.

1. **Inicio**
   - Nombre del cliente y "Disponible esta semana: $X USD" en grande.
   - Debajo, tope, gastado y fecha de renovación.
   - Interruptor grande "Pagar con este celular". Si el NFC del sistema está apagado, avísalo y ofrece abrir los ajustes de NFC.
   - Botones: Movimientos, Mis tarjetas, Mi permiso.
2. **Movimientos**
   - Lista con color según el tipo (pago hecho / recibido) y el estado.
   - Al tocar un movimiento se abre su detalle.
   - Botón "Actualizar".
3. **Mis tarjetas y celulares**
   - Lista de medios con su estado (activa/bloqueada) y último uso.
   - Acciones por medio: renombrar, bloquear/desbloquear y eliminar (con confirmación).
   - Botón "Agregar tarjeta":
     - Activa reader mode solo en esa pantalla y pide acercar la tarjeta.
     - Lee UID + wallet NDEF con la misma lógica de `NfcLectorEscritor`.
     - Si la tarjeta ya está registrada, avisa y dice a quién pertenece.
     - Pide un nombre para la tarjeta.
     - Nunca usa `makeReadOnly`.
4. **Mi permiso**
   - Muestra tope, gastado, restante y fecha de renovación.
   - Si no hay permiso o venció, muestra "Solicitar permiso" con un campo de tope (por defecto $200).
   - Al solicitarlo, muestra el enlace con los botones "Copiar" y "Abrir".
   - Recuerda en pantalla que solo el dueño de la wallet puede aprobarlo, con su sesión abierta en el test wallet, y que el enlace se abre una sola vez.
5. **Pago en curso**: pantalla u overlay automático con el feedback descrito en el punto 3.
6. **Vincular / Ajustes**: vincular, desvincular y activar o desactivar la voz.

Los errores deben mostrarse en pantalla y por voz, con mensajes claros. "Sin conexión con el servidor" debe sugerir revisar el `adb reverse` del celular del cliente.

## Restricciones (las mismas del proyecto)

- No leer, imprimir ni copiar la llave privada. Nunca `makeReadOnly`. No hacer git push ni publicar nada. Trabajar solo dentro de `C:\Users\diego\AndroidStudioProjects\NFC`.
- El Datáfono se instala solo en `R5CTB2FBZFP`. La Billetera se instala solo en `<SERIAL_CELULAR_CLIENTE>`, salo la excepción temporal descrita en "Dispositivos". Nunca instalar en el emulador.
- Interfaz, mensajes y comentarios en español.
- Montos con enteros (Long en Kotlin, BigInt en Node), sin flotantes.
- La moneda se lee siempre de la wallet (`assetCode`, `assetScale`).
- Tras cambiar el servidor, reinícialo y revisa el `adb reverse` de ambos celulares.

## Orden de trabajo y verificación

Avanza por fases y detente al final de cada una para mostrarme el resultado.

1. **Servidor**
   - Implementa las rutas nuevas, la migración y `MEDIO_BLOQUEADO`.
   - Corre `node --test` y `node probar.mjs`.
   - Prueba las rutas con requests desde PowerShell.
2. **Datáfono**
   - Implementa la lectura HCE con fallback a tarjeta.
   - Compila y prueba: `.\gradlew.bat :app:assembleDebug :app:testDebugUnitTest`.
   - Instala con `$env:ANDROID_SERIAL = "R5CTB2FBZFP"; .\gradlew.bat :app:installDebug`.
   - **Comprueba que el cobro con la tarjeta `10305DE4` sigue funcionando igual que antes.**
3. **Billetera**
   - Implementa vincular, inicio, movimientos, medios y permiso.
   - Compila y prueba: `.\gradlew.bat :billetera:assembleDebug :billetera:testDebugUnitTest`.
   - Instala con `$env:ANDROID_SERIAL = "<SERIAL_CELULAR_CLIENTE>"; .\gradlew.bat :billetera:installDebug`.
4. **HCE de punta a punta** (requiere los dos celulares)
   - Haz un cobro de $5 en el datáfono y págalo con el celular de Doña Rosa.
   - Verifica que el pago aparece en Movimientos de la Billetera, en el historial del datáfono y en `/familiar`.
   - Verifica que, con el celular bloqueado, el cobro falla con `MEDIO_BLOQUEADO`.
   - Verifica que, con "Pagar con este celular" desactivado, el datáfono no lo reconoce y sigue aceptando tarjetas.
   - Verifica que, con una tarjeta en el primer toque y el celular en el segundo, el resultado es `TARJETA_DISTINTA`.
5. **`PRUEBA_BILLETERA.md`**
   - Escribe una guía de prueba manual con tabla de resultados, en el mismo formato que `PRUEBA_DATAFONO.md`.
   - Incluye los requisitos del celular del cliente: NFC activado, pantalla desbloqueada y `adb reverse` en ambos celulares.
   - Incluye qué hacer si Android le da prioridad a otra app de pagos.
6. **Contexto del proyecto**: actualízalo con lo nuevo (arquitectura, módulo `:billetera`, rutas, pantallas, protocolo HCE y lecciones aprendidas).

Si algo del diseño no es posible tal cual (por ejemplo, el saldo, el listado de incoming o el comportamiento de HCE en un celular concreto), dímelo con la evidencia y propón la alternativa más simple antes de cambiar de rumbo.
~~~~

**Prompt 5**

~~~~text
NUEVO REQUISITO: UNA SOLA APP, DOS MODOS
Unifica el Datáfono y la Billetera en una sola app Android. Cada cuenta de la app puede PAGAR (Billetera) y COBRAR (Datáfono), y el usuario cambia de modo con un selector grande arriba: "Pagar" | "Cobrar". Sigue siendo un prototipo de viabilidad.

CUENTAS
- Una cuenta = una persona con nombre y wallet address del test wallet. Opcional: "nombre del negocio" para cuando cobra (si está vacío, usar el publicName de su wallet).
- Primera vez que se abre la app: pantalla "Crear mi cuenta" (nombre + wallet address). El servidor valida la wallet con un GET y guarda assetCode y assetScale.
- Servidor: generaliza "clientes" y "tienda" en una sola entidad "cuentas": POST /api/cuentas, GET /api/cuentas/:id. TIENDA_WALLET_ADDRESS del .env deja de ser obligatoria: migra los datos actuales (Doña Rosa y Tienda Don José pasan a ser cuentas) sin perder permisos, tarjetas ni historial.
- Sin autenticación real (es prototipo): guarda el id de la cuenta en el celular y muestra en Ajustes la advertencia de que no hay login.

MODO COBRAR (Datáfono)
- El mismo flujo de hoy, pero el incoming payment se crea en la wallet de LA CUENTA DEL CELULAR que cobra, y la voz dice su nombre de negocio: "¿Pagar cinco dólares a <negocio>?".
- Cobrar no requiere permiso: solo una wallet válida.
- En este modo: reader mode ACTIVO y la emulación de tarjeta (HCE) DESACTIVADA. El servicio HCE debe revisar el modo guardado y, si no está en "Pagar", responder 6A 82 para que nadie le cobre a un celular que está cobrando.
- Acepta tarjetas NFC y celulares en modo Pagar, como hoy.
- Error nuevo MISMA_CUENTA: no se puede cobrar a la propia cuenta.

MODO PAGAR (Billetera)
- Lo de la fase 3: el celular se hace pasar por tarjeta (HCE), pago en curso, "Mi permiso", agregar tarjeta, bloquear y desvincular.
- En este modo: HCE ACTIVO y reader mode APAGADO.
- Si la cuenta no tiene permiso aprobado, mostrar un botón grande "Aprobar mi permiso semanal" que abra el enlace de aprobación del test wallet EN EL NAVEGADOR DEL CELULAR. El finish URI es http://localhost:3000/familiar/callback (funciona en el celular por adb reverse). Al volver, mostrar "Permiso activo".
- Antes de responder al datáfono en un pago, pedir desbloqueo o huella con BiometricPrompt (BIOMETRIC_STRONG o DEVICE_CREDENTIAL). Si no se autentica, no responder. Quita la advertencia de "sin autenticación" de Ajustes cuando esto funcione.

DETALLES
- El modo elegido se recuerda al reabrir la app.
- Cada modo con su color de fondo distinto y su nombre siempre visible ("Modo Cobrar" / "Modo Pagar"), para que nadie se confunda.
- Historial en una pestaña: pagos hechos y cobros recibidos de la cuenta, con signo y color distintos.
- La página /familiar sigue funcionando para aprobar permisos desde el PC.

PROCESO
1. Actualiza el servidor y la migración de datos; corre sus pruebas.
2. Actualiza la app; compila, corre las pruebas y agrega pruebas para: cambio de modo (HCE responde solo en Pagar), MISMA_CUENTA y la migración.
3. Instala en el Samsung y verifica que la tarjeta 10305DE4 sigue pagando en modo Cobrar.
4. Actualiza PRUEBA_DATAFONO.md con una prueba de cada modo y la prueba entre dos celulares (requiere un segundo Android).
5. Resumen: qué verificaste, qué falta y qué necesitas de mí.
~~~~

### Fase 5 · Interfaz final desde Figma

**Prompt 6**

~~~~text
Implementa la interfaz Android de Pago al Toque en Kotlin y Jetpack Compose a partir del proyecto de Figma compartido. La prioridad es reproducir fielmente el prototipo interactivo: apariencia, contenido, navegación, gráficas y estados. Necesitamos una aplicación nativa navegable, no un nuevo diseño ni una descripción de cómo construirlo.

1. REFERENCIAS Y ACCESO

Prototipo interactivo, referencia principal:
https://www.figma.com/make/kO3zj5hpx08U0RRPf6oVrV/Prototipo-interactivo?code-node-id=0-6&p=f&t=FHf0jap2F9qSy4VZ-0&fullscreen=1

Enlace adicional al proyecto:
https://www.figma.com/make/kO3zj5hpx08U0RRPf6oVrV/Prototipo-interactivo?t=LFTfDhNuLSEtxTrB-0

Ambos enlaces apuntan al mismo archivo de Figma Make, con clave kO3zj5hpx08U0RRPf6oVrV. El segundo enlace no identifica un archivo independiente de Figma Design. No presupongas que sus parámetros muestran versiones distintas.

Antes de implementar:
- Lee AGENTS.md y las instrucciones aplicables. Inspecciona la estructura, dependencias, componentes y recursos del proyecto Android disponible.
- Abre el prototipo y recorre ambos modos y sus interacciones. Obtén capturas de referencia de cada pantalla y estado visible.
- Si dispones del conector Figma, sigue sus instrucciones de design-to-code. Para get_design_context de este archivo Make usa fileKey kO3zj5hpx08U0RRPf6oVrV y nodeId 0:1 según la herramienta; no conviertas code-node-id=0-6 en un identificador de frame Design.
- Si tienes acceso al código de Make, extrae estilos, assets, componentes, datos de ejemplo y comportamiento. Traduce esos recursos a Android nativo; el código web es una referencia.
- Distingue entre poder abrir públicamente el prototipo y tener permiso para extraerlo mediante el conector. Un error del conector no prueba por sí solo que el enlace sea privado.
- Si falla una vía, intenta las otras disponibles: navegador, vista publicada o exportación ya compartida. Si ninguna permite inspeccionarlo, explica qué falló y solicita capturas de las pantallas y estados o una exportación del proyecto. Continúa las tareas independientes del diseño; no inventes una interfaz y la presentes como fiel.

2. PRODUCTO Y ALCANCE

Pago al Toque permite a pequeños negocios usar su teléfono Android como punto de cobro. El foco son los negocios. La accesibilidad amplía a quiénes pueden atender, incluyendo clientes sin smartphone, personas mayores y personas con discapacidad visual.

La misma persona puede alternar entre dos modos:
- Pagar: consultar saldo, historial y revisar o confirmar compras, según el prototipo.
- Mi negocio: consultar ventas, ingresar montos, iniciar cobros y recibir pagos.

Conserva los nombres, ubicación y comportamiento reales del selector. Mantén separados el saldo personal y las ventas del negocio, así como los estados y filtros de cada modo.

En el cobro NFC, una tarjeta registrada identifica la billetera del cliente. El comerciante ingresa el monto; el primer toque identifica al cliente; se leen los detalles por voz; el cliente retira y vuelve a acercar la misma tarjeta para confirmar. El servidor gestiona el pago mediante Open Payments con un permiso previamente autorizado.

La tarjeta sencilla no representa aceptación de tarjetas bancarias Visa/Mastercard. El segundo toque expresa confirmación y no constituye autenticación criptográfica. No añadas soporte de tarjetas bancarias, HCE o conversión de divisas como funciones implementadas.

El encargo es el frontend Android y su conexión con los contratos existentes. Conserva la arquitectura: Node.js mantiene las claves privadas y se comunica con Open Payments; Android consume la API del servidor. No introduzcas claves privadas ni tokens de Open Payments en la app. No crees otro backend ni la página /familiar.

3. FIDELIDAD VISUAL

El prototipo actual tiene prioridad sobre los bocetos antiguos y las descripciones genéricas de la conversación. No agregues elementos por haber sido mencionados antes si no aparecen en el diseño aprobado.

Construye un inventario de pantallas, estados y componentes observados. Registra colores, tipografías, pesos, tamaños, espaciados, márgenes, radios, bordes, sombras, iconos, ilustraciones, pinceladas y estilos de gráficas que puedas verificar.

Reproduce:
- Jerarquía visual, alineación, proporciones y distribución.
- Textos y etiquetas, respetando su tono y longitud.
- Protagonismo de los montos y de las acciones de pagar y cobrar.
- Paleta y detalles decorativos originales. Si existen pinceladas, conserva su forma, posición, escala y discreción; no inventes adornos.
- Estados activos, inactivos, seleccionados, de carga, éxito y error.
- Iconos y assets originales disponibles, guardados como recursos locales de Android. No los sustituyas por emojis ni dejes URLs temporales de Figma.

Configura los componentes Compose y Material 3 para reproducir el diseño; evita que sus estilos predeterminados cambien su apariencia. Reutiliza los componentes y tokens existentes que coincidan con la referencia.

Adapta las medidas a dp y sp y a los insets de Android. Conserva la composición en el tamaño de referencia y evita recortes en el teléfono de prueba y en un ancho móvil menor. No conviertas la pantalla en una imagen ni uses un WebView para aparentar fidelidad.

4. GRÁFICAS E HISTORIAL: REQUISITO OBLIGATORIO

Implementa las gráficas del prototipo con datos; no las sustituyas por una imagen, una línea decorativa ni únicamente una lista de movimientos.

Para cada gráfica observada, conserva tipo, geometría, colores, grosor, curvas, rellenos, ejes, etiquetas, escala, formato monetario y posición. Reproduce filtros de período, selección de puntos, tooltips o animaciones cuando existan en la referencia.

Totales, período, gráfica y movimientos deben representar el mismo conjunto de datos. Mantén separados los pagos personales de las ventas. No inventes métricas o períodos ausentes.

Reutiliza una librería compatible instalada o utiliza Compose Canvas si permite reproducir la gráfica con precisión. Decide tras inspeccionar el proyecto; evita dependencias pesadas innecesarias.

Incluye una descripción accesible de la gráfica y acceso a sus datos mediante el historial. Si falta una API, usa fixtures deterministas en una capa de demostración separada y documentada. Un historial vacío debe mostrar un estado vacío; no mezcles ventas ficticias con datos reales.

5. INTERACCIONES Y ESTADOS

Implementa los recorridos observados: cambio de modo, acciones principales, confirmaciones, cancelaciones, resultados y regreso al inicio. Evita botones sin efecto.

El recorrido mínimo acordado del negocio es:
Ingresar monto -> esperar tarjeta -> revisar y escuchar detalles -> confirmar con segundo toque -> procesar -> verificar recepción -> mostrar pago recibido.

Completa los estados necesarios que falten en Figma usando sus mismos componentes y estilos. Identifica esas incorporaciones en la entrega; no rediseñes pantallas existentes.

Reglas funcionales:
- Valida montos positivos y usa aritmética decimal exacta o unidades mínimas; nunca Float/Double para calcular dinero. assetScale 2: 44.00 equivale a "4400".
- Obtén moneda y escala de la configuración; no confundas los ejemplos COP del diseño con USD del test wallet.
- Requiere retirada y una segunda presentación deliberada de la misma tarjeta. Mantenerla sobre la antena no debe confirmar.
- Evita duplicados por toques rápidos, eventos NFC repetidos, recomposición o reintentos de red.
- Respeta el vencimiento y los estados devueltos por el servidor. No declares éxito por una animación o temporizador.
- Evita otra ejecución mientras se procesa y gestiona el polling según el ciclo de vida.
- Explica errores de tarjeta no registrada, falta de permiso, límite alcanzado, vencimiento y conexión. Distingue un fallo confirmado de una operación cuyo resultado aún se desconoce.
- Usa TextToSpeech en español y vibración para NFC, sin repetir mensajes por recomposición. Conserva texto visible y semántica TalkBack.
- Si hay cashback u otros beneficios sin respaldo del servidor, limita su representación al modo de demostración y documenta la simulación; no prometas abonos reales.

6. CONEXIÓN CON EL SERVIDOR

Comprueba primero los contratos reales. Los acordados son:
GET /api/config
GET /api/clientes
POST /api/tarjetas {uid, clienteId}
POST /api/cobros {monto}
POST /api/cobros/:id/tarjeta {uid, walletAddress?}
POST /api/cobros/:id/confirmar {uid}
GET /api/cobros/:id
GET /api/pagos

Estados: esperando_tarjeta, esperando_confirmacion, procesando, pagado, error.

No asumas tipos de respuesta ni formato de monto sin comprobar el backend. Si faltan contratos para saldo, gráficas o historial de ventas, documenta el faltante y usa un repositorio de demostración intercambiable; no inventes endpoints como si existieran.

Desarrollo USB: http://localhost:3000 con adb reverse tcp:3000 tcp:3000. Permite HTTP únicamente a localhost mediante network_security_config; no habilites HTTP globalmente. La URL base debe ser configurable.

7. ORGANIZACIÓN, VALIDACIÓN Y ENTREGA

Trabaja en el proyecto existente sin sobrescribir cambios ajenos. Usa componentes Compose reutilizables, tema centralizado, ViewModels y estado observable según sus convenciones. Mantén acceso a datos, NFC y lógica de presentación fuera de los composables.

Antes de terminar:
- Compila y ejecuta las verificaciones apropiadas al cambio con las herramientas disponibles.
- Recorre ambos modos y sus estados importantes.
- Captura la app en dispositivo o emulador y compárala con Figma al mismo viewport y con los mismos datos. Corrige diferencias de geometría, texto, tipografía, assets y gráficas.
- Comprueba coherencia entre gráficas e historial y que se pueda completar, cancelar y reiniciar un cobro sin duplicados.
- Verifica voz, vibración y NFC en hardware compatible; identifica expresamente lo que solo se probó en simulación.
- No afirmes verificaciones visuales, compilaciones o pagos reales que no ejecutaste.

Entrega los archivos modificados, instrucciones para ejecutar, verificaciones realizadas y diferencias pendientes. Actualiza AI_USAGE.md y guarda los prompts literales en /prompts según las reglas del evento. No incluyas secretos ni datos privados.

CRITERIO DE TERMINACIÓN

La app debe permitir reconocer el prototipo compartido, recorrer ambos modos y consultar el historial mediante sus gráficas. Demuestra fidelidad comparando pantallas reales; una interfaz de estilo parecido no basta. Identifica las funciones conectadas al servidor y las simuladas en la entrega técnica.


[Imagen] [Imagen] [Imagen] [Imagen] [Imagen] [Imagen]. ahi tienes varias pantallas de referencia del figma igualmente para que guardes ese estilo es solamente modificar el apartado visual, quiero que tambien tu pongas el idioma en ingles y espanol y la voz cambie a una femenina
~~~~



## 6. Mapeo del trabajo local a cinco commits

Organizamos el código existente por las cinco fases de la sección 5. Conservamos los archivos completos y las dependencias compartidas; los commits no recrean versiones anteriores de la aplicación. Codex asiste en el mapeo, las verificaciones y la documentación. Los prompts de cada fase se copian del registro anterior a /prompts, sin añadir instrucciones de nuestra conversación de organización.

Las etapas 1 a 3 incorporan módulos y plantillas; el servidor completo queda disponible en la etapa 4 y el proyecto Android completo en la etapa 5. Las guías y los informes previos se conservan como documentación aportada: sus resultados históricos no equivalen a pruebas ejecutadas en esta revisión.

### ETAPA-1 — Viabilidad NFC

Utilidades puras NDEF, emulador Type 4, pruebas JVM existentes y guía manual. El lector Android actual también usa el protocolo de billetera; se incorpora junto a la app completa en la etapa 5. Esta etapa es un módulo fuente, no una APK independiente.

Archivos incorporados:

- [NFC/.gitignore](<NFC/.gitignore>)
- [NFC/app/src/main/java/com/example/nfc/NdefUtil.kt](<NFC/app/src/main/java/com/example/nfc/NdefUtil.kt>)
- [NFC/app/src/test/java/com/example/nfc/NdefUtilTest.kt](<NFC/app/src/test/java/com/example/nfc/NdefUtilTest.kt>)
- [NFC/PRUEBAS_NFC.md](<NFC/PRUEBAS_NFC.md>)
- [prompts/Etapa-1/prompt-1.md](<prompts/Etapa-1/prompt-1.md>)
- [.gitignore](<.gitignore>)

Verificación: Revisión de dependencias: NdefUtil.kt y NdefUtilTest.kt forman una unidad de lógica JVM; 10 métodos @Test presentes. No se ejecutan pruebas Kotlin: no hay compilador Kotlin, SDK Android ni caché Gradle en las ubicaciones locales consultadas. No se prueba NFC físico.

Commit: `chore(nfc): incorpora utilidades NDEF y pruebas de viabilidad`. Sin push.

### ETAPA-2 — Base del datáfono y Open Payments

Configuración, SDK, montos y persistencia. medios.mjs y sus pruebas se incluyen aquí porque datos.mjs importa su migración; contienen también lógica de cuentas e historial de la etapa 4. El servidor ejecutable se incorpora en la etapa 4. MontoUtil es una utilidad Kotlin independiente.

Archivos incorporados:

- [NFC/servidor/.gitignore](<NFC/servidor/.gitignore>)
- [NFC/servidor/.env.example](<NFC/servidor/.env.example>)
- [NFC/servidor/package.json](<NFC/servidor/package.json>)
- [NFC/servidor/package-lock.json](<NFC/servidor/package-lock.json>)
- [NFC/servidor/config.mjs](<NFC/servidor/config.mjs>)
- [NFC/servidor/util.mjs](<NFC/servidor/util.mjs>)
- [NFC/servidor/util.test.mjs](<NFC/servidor/util.test.mjs>)
- [NFC/servidor/op.mjs](<NFC/servidor/op.mjs>)
- [NFC/servidor/datos.mjs](<NFC/servidor/datos.mjs>)
- [NFC/servidor/medios.mjs](<NFC/servidor/medios.mjs>)
- [NFC/servidor/medios.test.mjs](<NFC/servidor/medios.test.mjs>)
- [NFC/servidor/probar.mjs](<NFC/servidor/probar.mjs>)
- [NFC/app/src/main/java/com/example/nfc/MontoUtil.kt](<NFC/app/src/main/java/com/example/nfc/MontoUtil.kt>)
- [NFC/app/src/test/java/com/example/nfc/MontoUtilTest.kt](<NFC/app/src/test/java/com/example/nfc/MontoUtilTest.kt>)
- [prompts/Etapa-2/prompt-2.md](<prompts/Etapa-2/prompt-2.md>)

Verificación: 19 pruebas Node aprobadas: 5 en util.test.mjs y 14 en medios.test.mjs. Comprobación de sintaxis con node --check aprobada para los módulos JavaScript de esta etapa. Las pruebas Kotlin de montos quedan pendientes por falta de toolchain. No se ejecuta probar.mjs: requiere credenciales y llamadas al test wallet; no se han verificado pagos reales.

Commit: `chore(backend): incorpora base Open Payments y montos`. Sin push.

### ETAPA-3 — Aprobación remota del permiso

Plantillas de la página del familiar, enlace copiable y formulario de retorno remoto. Las rutas /familiar/aprobar, /familiar/callback y /familiar/completar comparten servidor.mjs con cuentas y cobros: se incorporan juntas en la etapa 4. Esta etapa permite verificar el HTML generado, no completar una autorización real.

Archivos incorporados:

- [NFC/servidor/paginas.mjs](<NFC/servidor/paginas.mjs>)
- [prompts/Etapa-3/prompt-3.md](<prompts/Etapa-3/prompt-3.md>)

Verificación: Siete comprobaciones del HTML generado aprobadas con node:assert/strict: formulario hacia /familiar/completar, campo URL obligatorio, botón de copia, escape del enlace, escape del nombre del cliente, estado vacío y escape de mensajes. No se ejecuta navegador ni se valida una autorización remota real.

Commit: consultar el historial local por los archivos de esta unidad. Sin push.

### ETAPA-4 — Billetera, cuentas y servidor integrado

Servidor HTTP completo, permisos y rutas de billetera, con sus dependencias ya incorporadas. Incluye el protocolo HCE puro y sus pruebas. La integración Android de los modos se conserva completa con la interfaz final en la etapa 5.

Archivos incorporados:

- [NFC/servidor/permisos.mjs](<NFC/servidor/permisos.mjs>)
- [NFC/servidor/billetera.mjs](<NFC/servidor/billetera.mjs>)
- [NFC/servidor/servidor.mjs](<NFC/servidor/servidor.mjs>)
- [NFC/app/src/main/java/com/example/nfc/ProtocoloHce.kt](<NFC/app/src/main/java/com/example/nfc/ProtocoloHce.kt>)
- [NFC/app/src/test/java/com/example/nfc/ProtocoloHceTest.kt](<NFC/app/src/test/java/com/example/nfc/ProtocoloHceTest.kt>)
- [NFC/PRUEBA_DATAFONO.md](<NFC/PRUEBA_DATAFONO.md>)
- [prompts/Etapa-4/prompt-4.md](<prompts/Etapa-4/prompt-4.md>)
- [prompts/Etapa-4/prompt-5.md](<prompts/Etapa-4/prompt-5.md>)

Verificación: Sintaxis de los 12 módulos JavaScript del backend comprobada con node --check; 32 importaciones relativas resueltas. Las 19 pruebas de utilidades y medios ya aprobadas siguen correspondiendo a archivos sin modificaciones. No se inicia el servidor con las credenciales locales ni se ejecutan pagos; quedan pendientes autorización real, HTTP de punta a punta y pruebas HCE en hardware. Las pruebas JVM del protocolo quedan pendientes por falta de toolchain.

Commit: consultar el historial local por los archivos de esta unidad. Sin push.

### ETAPA-5 — Aplicación Android e interfaz final

Aplicación Android completa, lector NFC, servicio HCE, cuentas, modos Pagar/Cobrar, historial, gráficas, idiomas, voz, recursos y configuración Gradle. Se agrupan para conservar las dependencias entre la lógica y la interfaz existentes. Incluye contexto, informe de arquitectura y notas técnicas locales.

Archivos incorporados:

- [NFC/CONTEXTO_APP.md](<NFC/CONTEXTO_APP.md>)
- [NFC/app/.gitignore](<NFC/app/.gitignore>)
- [NFC/app/build.gradle.kts](<NFC/app/build.gradle.kts>)
- [NFC/app/src/androidTest/java/com/example/nfc/ExampleInstrumentedTest.kt](<NFC/app/src/androidTest/java/com/example/nfc/ExampleInstrumentedTest.kt>)
- [NFC/app/src/main/AndroidManifest.xml](<NFC/app/src/main/AndroidManifest.xml>)
- [NFC/app/src/main/java/com/example/nfc/Componentes.kt](<NFC/app/src/main/java/com/example/nfc/Componentes.kt>)
- [NFC/app/src/main/java/com/example/nfc/Entorno.kt](<NFC/app/src/main/java/com/example/nfc/Entorno.kt>)
- [NFC/app/src/main/java/com/example/nfc/Estado.kt](<NFC/app/src/main/java/com/example/nfc/Estado.kt>)
- [NFC/app/src/main/java/com/example/nfc/Grafica.kt](<NFC/app/src/main/java/com/example/nfc/Grafica.kt>)
- [NFC/app/src/main/java/com/example/nfc/Iconos.kt](<NFC/app/src/main/java/com/example/nfc/Iconos.kt>)
- [NFC/app/src/main/java/com/example/nfc/MainActivity.kt](<NFC/app/src/main/java/com/example/nfc/MainActivity.kt>)
- [NFC/app/src/main/java/com/example/nfc/ModoCobrar.kt](<NFC/app/src/main/java/com/example/nfc/ModoCobrar.kt>)
- [NFC/app/src/main/java/com/example/nfc/ModoPagar.kt](<NFC/app/src/main/java/com/example/nfc/ModoPagar.kt>)
- [NFC/app/src/main/java/com/example/nfc/NfcLectorEscritor.kt](<NFC/app/src/main/java/com/example/nfc/NfcLectorEscritor.kt>)
- [NFC/app/src/main/java/com/example/nfc/PagoHceService.kt](<NFC/app/src/main/java/com/example/nfc/PagoHceService.kt>)
- [NFC/app/src/main/java/com/example/nfc/PestanaHistorial.kt](<NFC/app/src/main/java/com/example/nfc/PestanaHistorial.kt>)
- [NFC/app/src/main/java/com/example/nfc/ServidorApi.kt](<NFC/app/src/main/java/com/example/nfc/ServidorApi.kt>)
- [NFC/app/src/main/java/com/example/nfc/Textos.kt](<NFC/app/src/main/java/com/example/nfc/Textos.kt>)
- [NFC/app/src/main/java/com/example/nfc/Ventas.kt](<NFC/app/src/main/java/com/example/nfc/Ventas.kt>)
- [NFC/app/src/main/java/com/example/nfc/VozFemenina.kt](<NFC/app/src/main/java/com/example/nfc/VozFemenina.kt>)
- [NFC/app/src/main/java/com/example/nfc/ui/theme/Color.kt](<NFC/app/src/main/java/com/example/nfc/ui/theme/Color.kt>)
- [NFC/app/src/main/java/com/example/nfc/ui/theme/Theme.kt](<NFC/app/src/main/java/com/example/nfc/ui/theme/Theme.kt>)
- [NFC/app/src/main/java/com/example/nfc/ui/theme/Type.kt](<NFC/app/src/main/java/com/example/nfc/ui/theme/Type.kt>)
- [NFC/app/src/main/keepRules/rules.keep](<NFC/app/src/main/keepRules/rules.keep>)
- [NFC/app/src/main/res/drawable/ic_launcher_background.xml](<NFC/app/src/main/res/drawable/ic_launcher_background.xml>)
- [NFC/app/src/main/res/drawable/ic_launcher_foreground.xml](<NFC/app/src/main/res/drawable/ic_launcher_foreground.xml>)
- [NFC/app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml](<NFC/app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml>)
- [NFC/app/src/main/res/mipmap-anydpi-v26/ic_launcher_round.xml](<NFC/app/src/main/res/mipmap-anydpi-v26/ic_launcher_round.xml>)
- [NFC/app/src/main/res/mipmap-hdpi/ic_launcher.webp](<NFC/app/src/main/res/mipmap-hdpi/ic_launcher.webp>)
- [NFC/app/src/main/res/mipmap-hdpi/ic_launcher_round.webp](<NFC/app/src/main/res/mipmap-hdpi/ic_launcher_round.webp>)
- [NFC/app/src/main/res/mipmap-mdpi/ic_launcher.webp](<NFC/app/src/main/res/mipmap-mdpi/ic_launcher.webp>)
- [NFC/app/src/main/res/mipmap-mdpi/ic_launcher_round.webp](<NFC/app/src/main/res/mipmap-mdpi/ic_launcher_round.webp>)
- [NFC/app/src/main/res/mipmap-xhdpi/ic_launcher.webp](<NFC/app/src/main/res/mipmap-xhdpi/ic_launcher.webp>)
- [NFC/app/src/main/res/mipmap-xhdpi/ic_launcher_round.webp](<NFC/app/src/main/res/mipmap-xhdpi/ic_launcher_round.webp>)
- [NFC/app/src/main/res/mipmap-xxhdpi/ic_launcher.webp](<NFC/app/src/main/res/mipmap-xxhdpi/ic_launcher.webp>)
- [NFC/app/src/main/res/mipmap-xxhdpi/ic_launcher_round.webp](<NFC/app/src/main/res/mipmap-xxhdpi/ic_launcher_round.webp>)
- [NFC/app/src/main/res/mipmap-xxxhdpi/ic_launcher.webp](<NFC/app/src/main/res/mipmap-xxxhdpi/ic_launcher.webp>)
- [NFC/app/src/main/res/mipmap-xxxhdpi/ic_launcher_round.webp](<NFC/app/src/main/res/mipmap-xxxhdpi/ic_launcher_round.webp>)
- [NFC/app/src/main/res/values/colors.xml](<NFC/app/src/main/res/values/colors.xml>)
- [NFC/app/src/main/res/values/strings.xml](<NFC/app/src/main/res/values/strings.xml>)
- [NFC/app/src/main/res/values/themes.xml](<NFC/app/src/main/res/values/themes.xml>)
- [NFC/app/src/main/res/xml/apduservice.xml](<NFC/app/src/main/res/xml/apduservice.xml>)
- [NFC/app/src/main/res/xml/backup_rules.xml](<NFC/app/src/main/res/xml/backup_rules.xml>)
- [NFC/app/src/main/res/xml/data_extraction_rules.xml](<NFC/app/src/main/res/xml/data_extraction_rules.xml>)
- [NFC/app/src/main/res/xml/network_security_config.xml](<NFC/app/src/main/res/xml/network_security_config.xml>)
- [NFC/app/src/test/java/com/example/nfc/ExampleUnitTest.kt](<NFC/app/src/test/java/com/example/nfc/ExampleUnitTest.kt>)
- [NFC/app/src/test/java/com/example/nfc/VentasTest.kt](<NFC/app/src/test/java/com/example/nfc/VentasTest.kt>)
- [NFC/app/src/test/java/com/example/nfc/VozFemeninaTest.kt](<NFC/app/src/test/java/com/example/nfc/VozFemeninaTest.kt>)
- [NFC/build.gradle.kts](<NFC/build.gradle.kts>)
- [NFC/gradle.properties](<NFC/gradle.properties>)
- [NFC/gradle/gradle-daemon-jvm.properties](<NFC/gradle/gradle-daemon-jvm.properties>)
- [NFC/gradle/libs.versions.toml](<NFC/gradle/libs.versions.toml>)
- [NFC/gradle/wrapper/gradle-wrapper.jar](<NFC/gradle/wrapper/gradle-wrapper.jar>)
- [NFC/gradle/wrapper/gradle-wrapper.properties](<NFC/gradle/wrapper/gradle-wrapper.properties>)
- [NFC/gradlew](<NFC/gradlew>)
- [NFC/gradlew.bat](<NFC/gradlew.bat>)
- [NFC/reports/Arquitectura Open Payments Pago al Toque.md](<NFC/reports/Arquitectura Open Payments Pago al Toque.md>)
- [NFC/research_notes/Arquitectura Open Payments Pago al Toque/arquitectura_actores.md](<NFC/research_notes/Arquitectura Open Payments Pago al Toque/arquitectura_actores.md>)
- [NFC/research_notes/Arquitectura Open Payments Pago al Toque/flujo_pagos.md](<NFC/research_notes/Arquitectura Open Payments Pago al Toque/flujo_pagos.md>)
- [NFC/research_notes/Arquitectura Open Payments Pago al Toque/grants_gnap.md](<NFC/research_notes/Arquitectura Open Payments Pago al Toque/grants_gnap.md>)
- [NFC/settings.gradle.kts](<NFC/settings.gradle.kts>)
- [prompts/Etapa-5/prompt-6.md](<prompts/Etapa-5/prompt-6.md>)

Verificación: Parseo XML aprobado para 12 archivos de manifiesto y recursos. Se comparan hashes SHA-256 de los 86 archivos existentes seleccionados: permanecen idénticos al estado inicial. No se compila ni se ejecutan pruebas Android: faltan SDK Android, compilador Kotlin y caché Gradle en las ubicaciones consultadas. Quedan pendientes instalación, NFC físico, voz, comparación con Figma y recorrido de pagos reales. No se declara que las funciones descritas por los prompts estén todas verificadas.

Commit: consultar el historial local por los archivos de esta unidad. Sin push.

Revisión de formato: el chequeo estándar señala finales CRLF en los archivos Gradle de Windows y una línea vacía final en libs.versions.toml. Se conservan los archivos originales; al reconocer CRLF solo queda esa observación cosmética. No afecta a la sintaxis ni constituye una prueba de compilación.
