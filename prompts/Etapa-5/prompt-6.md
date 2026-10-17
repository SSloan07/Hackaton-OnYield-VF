# Aplicación Android e interfaz final

## Prompt

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
