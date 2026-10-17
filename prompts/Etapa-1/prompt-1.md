# Viabilidad NFC

## Prompt

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
