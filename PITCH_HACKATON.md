# Pago al Toque — guion de pitch

Propuesta para 5 minutos, incluida la demo. Preparada a partir del código y la documentación del repositorio; no se ha ejecutado una nueva prueba en dispositivos. La imagen mencionada por el equipo no está disponible en los archivos revisados, por lo que esta estructura queda pendiente de ajustarse a ella.

## Mensaje central

**Pago al Toque acerca los pagos digitales al comercio de barrio: una tarjeta NFC para pagar, un Android para cobrar y Open Payments para conectar el flujo.**

La historia se centra en Rosa, una persona con baja visión, y José, un comerciante con un Android compatible con NFC. Rosa toma sus propias decisiones y administra su dinero; la barrera que exploramos es poder comprobar un cobro en una interfaz que le cuesta leer. La baja visión no implica poca familiaridad digital. Son personajes ficticios para ilustrar una situación de uso, no testimonios ni resultados de investigación. Esta experiencia también puede ser relevante para personas con poca familiaridad digital, un grupo con necesidades distintas.

## 1. Problema y personas — 0:00 a 0:40

**En pantalla:** “¿Cuánto voy a pagar?”. Rosa en una compra cotidiana; mostrar la interacción y el producto.

**Guion:**

“Imaginen esta escena. Rosa tiene baja visión. Sale a comprar pan y café a la tienda de José, como cualquier mañana. Al pagar, el monto aparece en una pantalla que le cuesta leer. Se acerca y pregunta: «¿Cuánto dice ahí?». [Pausa.] Antes de confirmar, quiere comprobar por sí misma cuánto va a pagar. Piensen en su próxima compra: ¿cómo confirmarían el monto si no pudieran leer esa pantalla? De esa pregunta nace Pago al Toque.”

No agregar porcentajes de exclusión o entrevistas que el equipo no haya documentado.

## 2. Solución — 0:40 a 1:20

**En pantalla:** “Acercar → escuchar → confirmar”. Foto o captura de la tarjeta y del modo Cobrar.

**Guion:**

“Volvamos a la tienda. José introduce el monto en su Android. Rosa acerca su tarjeta registrada y escucha el valor y el nombre del comercio. Tiene esa información antes de decidir si vuelve a acercarla para confirmar. Eso es lo que construimos: acercar, escuchar y confirmar. Previamente, el titular de la wallet aprueba un permiso con un límite semanal. En esta compra, Rosa no necesita navegar una app en su propio celular. Ahora les mostramos el recorrido en nuestro entorno de pruebas.”

El registro y la aprobación inicial siguen siendo necesarios. La voz y el doble toque son decisiones de diseño; su utilidad con usuarios debe validarse. No afirmar que toda la experiencia es accesible: falta evaluar también el registro, el consentimiento, la ubicación de la tarjeta, los errores y la confirmación final.

## 3. Demo — 1:20 a 2:50

**En pantalla:** dispositivo o video del flujo de tarjeta. Etiqueta visible: “Entorno de pruebas de Open Payments”.

**Transición:** “Esta es la compra de Rosa. Escuchen qué información recibe antes de confirmar”. Usar el monto y la moneda reales de la demo; si aparecen USD de prueba, indicarlo sin presentarlos como pesos colombianos.

1. **15 s — Autorización previa:** mostrar el permiso aprobado y el tope. Si se usa una grabación, incluir el consentimiento en la wallet. Explicar que ese límite corresponde al permiso, no al saldo bancario.
2. **15 s — Solicitud:** José ingresa un monto pequeño y pulsa Cobrar.
3. **20 s — Primer toque:** Rosa acerca la tarjeta; dejar que se escuche el monto y el comercio, sin hablar encima de la voz.
4. **15 s — Confirmación:** retirar y acercar la misma tarjeta otra vez.
5. **15 s — Resultado:** mostrar el pago confirmado y el movimiento correspondiente, idealmente también en el test wallet.
6. **10 s — Interpretación:** “Rosa pudo escuchar el monto antes del segundo toque. Ahora veamos qué hizo posible procesar ese pago”.

Preparar un video de respaldo del flujo completo. Usar como demo principal tarjeta → Android: la documentación registra pruebas de ese recorrido. El recorrido celular → celular figura pendiente en el contexto del 15 de octubre; actualizar esa afirmación solo si el equipo tiene una prueba posterior.

## 4. Papel de Open Payments — 2:50 a 3:40

**En pantalla:** Tarjeta NFC → Android → servidor cliente → wallets de prueba. Debajo: incoming payment → quote → outgoing payment → consulta de recepción.

**Guion:**

“Para que ese gesto se convierta en una instrucción de pago, usamos Open Payments. Nuestro servidor crea una solicitud de recepción para el comercio, obtiene una cotización y solicita el pago desde la wallet del comprador usando el permiso aprobado. Después consulta la recepción para informar el resultado. NFC conecta la interacción presencial con ese flujo. Diseñamos sobre una API abierta; la demo utiliza dos cuentas del mismo entorno de pruebas. Integrarnos con otros proveedores compatibles es un siguiente paso que todavía debemos validar.”

Si preguntan por Interledger: Open Payments expresa las instrucciones y autorizaciones; las entidades que mantienen las cuentas ejecutan el movimiento. No afirmar que se implementó un conector ILP propio.

## 5. Aprendizajes e impacto a validar — 3:40 a 4:30

**En pantalla:** “Comprensión · Autonomía · Control”. Un aprendizaje y tres métricas propuestas.

**Guion:**

“Construir este recorrido nos llevó a separar dos momentos: autorizar un permiso y confirmar una compra. Nuestra hipótesis es que escuchar el monto y el comercio antes del segundo toque puede ayudar a personas como Rosa a decidir con más autonomía. Ahora queremos probarlo junto con personas con baja visión y comerciantes: observar si comprenden el cobro, si logran completar la compra y qué apoyos prefieren. Esa validación es nuestro siguiente paso.”

Propuesta de piloto, todavía no realizado: cinco comercios y diez personas, con transacciones de prueba. Incluir personas con baja visión que deseen participar. Registrar errores, asistencia requerida, comprensión y preferencias; no presentar estas métricas como resultados obtenidos. Evaluar ruido en la tienda, privacidad del audio, localización del lector NFC, posibilidad de desistir y certeza de la confirmación final. La voz por sí sola no cubre todas las discapacidades; otras necesidades requieren soluciones y pruebas específicas.

## 6. Cierre — 4:30 a 5:00

**En pantalla:** “Escuchar el monto. Decidir. Confirmar”. Nombre del equipo y enlace al repositorio/demo.

**Guion:**

“Volvamos a Rosa y a esa pregunta: «¿Cuánto dice ahí?». Queremos que pueda escuchar el cobro y decidir por sí misma si lo confirma. Hoy mostramos un prototipo; el siguiente paso es construir esa experiencia junto con quienes la van a usar. Con Open Payments, queremos acercar los pagos abiertos a las compras de todos los días. Somos Pago al Toque: acercar, escuchar y confirmar.”

## Cómo contar la historia

- Presentar a Rosa como un personaje ilustrativo con una vida cotidiana y capacidad de decisión. Hablar con naturalidad, sin imitar una discapacidad ni usar un tono de lástima.
- Hacer una pausa breve después de «¿Cuánto dice ahí?». Mirar al público al plantear cómo comprobaría el monto. La pregunta es retórica: no pedir que cierre los ojos ni representar una simulación de discapacidad.
- Mantener a Rosa y José durante problema, solución, demo y cierre. Evitar interrumpir la historia con una lista de todas las funciones de la app.
- Durante la demo, dejar oír la voz de la aplicación. La narración debe describir exactamente lo que se oye y se observa, sin atribuirle al prototipo mensajes que no emite.
- No convertir a un familiar en quien decide por Rosa por el hecho de tener una discapacidad. El consentimiento corresponde al titular de la wallet.
- Usar una imagen cotidiana y respetuosa si se ilustra la escena. No presentar una foto de archivo como una participante real ni inventar una cita testimonial.

## Preguntas del jurado

- **¿Lo han validado con personas con discapacidad?** No hay evidencia de esa validación en los materiales revisados. Es una hipótesis de diseño que queremos probar junto con personas con baja visión. La demo demuestra el flujo técnico, no demuestra todavía accesibilidad integral.
- **¿Sirve para cualquier discapacidad?** El caso presentado se centra en baja visión. Hay necesidades distintas, por ejemplo auditivas o motrices, que requieren otras opciones de interacción y validación.

- **¿Qué aporta frente a una tarjeta bancaria?** Explora una interfaz presencial para wallets compatibles con Open Payments y permite cobrar con un Android compatible. No equivale a una terminal certificada para tarjetas bancarias.
- **¿Funciona sin internet?** La tarjeta no necesita conectividad, pero el teléfono que cobra y el servidor necesitan conexión para procesar el pago.
- **¿Rosa necesita celular?** No durante la compra con tarjeta. Sí debe existir una wallet vinculada y un proceso previo de registro y autorización.
- **¿El dinero está en la tarjeta?** No. La tarjeta sirve como medio registrado para identificar la cuenta en el prototipo.
- **¿Es seguro por usar NFC?** NFC por sí solo no demuestra autenticidad. El límite del permiso lo aplica la infraestructura de autorización; el doble toque y la biometría del modo celular son controles de la aplicación. Antes de un uso productivo hay que reforzar autenticación y protección frente a clonación y repetición.
- **¿Han probado distintos bancos o proveedores?** La evidencia del repositorio corresponde a cuentas del mismo test wallet. La compatibilidad entre proveedores requiere pruebas adicionales.
- **¿Cuál sería el modelo de negocio?** Es una hipótesis por validar: ofrecer la solución a comercios o integrarla con proveedores de wallets. No afirmar ingresos, ahorro porcentual ni comisiones garantizadas.

## Enfoque ante el jurado

Mi recomendación es priorizar inclusión y mostrarla en una acción concreta: pagar con una tarjeta registrada sin manejar una app durante la compra. La innovación se sostiene en la combinación de experiencia presencial y permisos de Open Payments, no en afirmar que NFC es una tecnología nueva. La fiabilidad debe demostrarse con el flujo completo y evidencia del resultado.

Los criterios oficiales incluyen problema, Open Payments, experiencia, calidad técnica, presentación, inclusión, innovación e impacto. La página del evento de Medellín destaca premios de inclusión, impacto e innovación. Esto respalda el enfoque sugerido; no permite predecir las preferencias individuales del jurado.

## Fuentes y evidencia

- [Criterios oficiales y duración del pitch](https://interledger.org/hackathon/submission-judging/).
- [Hackatón Medellín, octubre de 2026](https://interledger.org/hackathon/2026-hackathon-medellin/).
- [Misión de Interledger Foundation](https://interledger.org/about-us/).
- [Estado documentado del prototipo](NFC/CONTEXTO_APP.md).
- [Guía de pruebas y demo](NFC/PRUEBA_DATAFONO.md).
- [Análisis de arquitectura](NFC/reports/Arquitectura%20Open%20Payments%20Pago%20al%20Toque.md).
- Implementación: `NFC/servidor/op.mjs` y `NFC/servidor/servidor.mjs`.
