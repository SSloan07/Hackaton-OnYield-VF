# Pruebas manuales de NFC

App de prueba de viabilidad: pestañas **Leer**, **Escribir** y **Emitir**.
Antes de empezar: NFC encendido en el Android (si está apagado, la app muestra un aviso con un botón a los ajustes).

Para ver el registro de APDUs también en la PC:

```powershell
& "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe" logcat -s NFCDEMO
```

---

## Prueba 1 · Android lee una tarjeta sin contacto (Cívica o bancaria)

1. Abre la pestaña **Leer**.
2. Acerca la tarjeta a la parte de atrás del Android y mantenla quieta 1–2 segundos.

**Esperado:** vibración corta y una tarjeta nueva en la lista con el UID y las tecnologías (`NfcA`, `IsoDep` u otras).
Una tarjeta bancaria normalmente no tiene NDEF ("Formato NDEF: No"); es normal. Algunas tarjetas bancarias usan un UID aleatorio que cambia en cada lectura.

## Prueba 2 · Android escribe y lee un sticker NTAG215

1. Pestaña **Escribir**: deja `https://interledger.org` (o escribe otra URL).
2. Toca **Escribir en el próximo sticker** y acerca el sticker. Mantenlo quieto hasta ver el resultado.
3. Ve a **Leer** y acerca de nuevo el sticker.

**Esperado:** "✔ Escrito correctamente" y, al leer, un registro `URI: https://interledger.org` (la misma URL), con tamaño máximo ≈ 496 bytes y "Escribible: Sí".
Errores posibles: etiqueta no escribible, espacio insuficiente o "retirada demasiado rápido" (vuelve a intentarlo; el modo escritura sigue activo hasta tener éxito o tocar Cancelar).
La app nunca bloquea el sticker (no usa `makeReadOnly`).

## Prueba 3 · iPhone lee el sticker

Condiciones: iPhone XS o más reciente, pantalla encendida y desbloqueada, sin la cámara ni la billetera (Wallet) abiertas.

1. Acerca la parte de arriba del iPhone (zona de la cámara trasera) al sticker escrito en la Prueba 2.

**Esperado:** notificación en el iPhone para abrir la URL en Safari.

## Prueba 4 · iPhone lee el Android en modo Emitir

Mismas condiciones que la Prueba 3.

1. En el Android abre la pestaña **Emitir**. La pantalla se queda encendida mientras estés en esta pestaña; mantenla desbloqueada.
2. Acerca la parte de arriba del iPhone a la parte de atrás del Android (zona del NFC, normalmente al centro).
3. Observa el registro de APDUs en pantalla (`<-` comando recibido, `->` respuesta).

**Esperado:** notificación en el iPhone con la URL.

Secuencia normal en el registro:

```
<- 00 A4 04 00 07 D2 76 00 00 85 01 01 00    -> 90 00        (SELECT AID)
<- 00 A4 00 0C 02 E1 03                      -> 90 00        (SELECT CC)
<- 00 B0 00 00 0F                            -> CC... 90 00  (leer CC)
<- 00 A4 00 0C 02 E1 04                      -> 90 00        (SELECT NDEF)
<- 00 B0 00 00 02                            -> 00 xx 90 00  (leer NLEN)
<- 00 B0 00 02 xx                            -> mensaje 90 00
```

**Diagnóstico:**
- **No llega ningún comando:** el iPhone no detecta la emulación. Revisa que el NFC esté encendido, que estés en la pestaña Emitir con la pantalla desbloqueada, prueba otras posiciones y que la app de Google Wallet u otra no esté interfiriendo.
  En este Galaxy A53 (Android 16) el sistema trae un servicio propio, "Etiqueta incorporada", que registra el **mismo AID**; la app lo evita pidiéndose como servicio preferido mientras la pestaña Emitir está abierta. Si ves una ventana preguntando qué app usar, elige esta.
- **Llega el SELECT del AID pero nunca READ BINARY:** revisa el CC (debe ser `00 0F 20 00 3B 00 34 04 06 E1 04 [tamaño] 00 FF`).
- **Llega READ BINARY pero el iPhone no abre nada:** revisa el archivo NDEF (NLEN y el registro URI); la pestaña Emitir muestra los bytes esperados.

## Prueba 5 · Android intenta leer el iPhone

1. Pestaña **Leer** en el Android y acerca el iPhone (desbloqueado y también bloqueado).

**Esperado:** que **no** funcione, porque el iPhone no emula etiquetas NDEF para apps de terceros. Puede que el Android detecte algo (por ejemplo, el iPhone ofreciendo Apple Pay con IsoDep) pero sin NDEF. Solo anota el resultado.

---

## Resultados

| Prueba | Funcionó (Sí/No) | Modelo de celular | Notas |
|---|---|---|---|
| 1 · Android lee tarjeta sin contacto | | | |
| 2 · Android escribe y lee NTAG215 | | | |
| 3 · iPhone lee el sticker | | | |
| 4 · iPhone lee el Android (Emitir) | | | |
| 5 · Android lee el iPhone | | | |
