# Billetera, cuentas y servidor integrado

## Prompt

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
