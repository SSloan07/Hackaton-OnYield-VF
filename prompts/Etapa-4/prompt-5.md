# Billetera, cuentas y servidor integrado

## Prompt

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
