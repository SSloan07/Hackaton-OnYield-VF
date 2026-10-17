# Aprobación remota del permiso

## Prompt

Mi amigo es el dueño de la wallet de Doña Rosa y está en otro computador, así que no puede abrir localhost. Haz este cambio en /familiar:
- Al tocar "Aprobar permiso semanal", NO redirijas el navegador. Muestra el enlace de aprobación (interact.redirect) en un cuadro con un botón "Copiar enlace" y el texto: "Envía este enlace al dueño de la wallet. Que inicie sesión y acepte."
- Agrega debajo un campo "Pega aquí la dirección a la que llegó tu amigo" y un botón "Completar permiso". El servidor debe extraer interact_ref y hash de esa URL y seguir el mismo proceso del callback: verificar el hash, continuar el grant y guardar el token.
- No cambies nada más. Reinicia el servidor cuando termines.
