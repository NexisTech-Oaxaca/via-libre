# Requirements Document

## Introduction

ViaLibre es una aplicación web que centraliza y visualiza bloqueos carreteros en tiempo real en el estado de Oaxaca, México. La comunidad reporta, confirma y sigue incidentes viales, reemplazando la información dispersa en redes sociales con datos verificados comunitariamente. El MVP utiliza datos estáticos (seed data con incidentes reales de Oaxaca) para demostrar la funcionalidad completa sin depender de base de datos activa.

---

## Glossary

- **ViaLibre**: La aplicación web descrita en este documento.
- **Incidente**: Un bloqueo, manifestación, derrumbe u otro evento vial que afecta la circulación en Oaxaca.
- **Confirmación**: Validación comunitaria que indica que un Incidente sigue ocurriendo.
- **Negación**: Indicación comunitaria de que un Incidente reportado es falso o ya no existe.
- **Confirmación_Neta**: Resultado de `confirmaciones_count − denegaciones_count` para un Incidente.
- **Mapa_Interactivo**: Visualización cartográfica basada en Leaflet + OpenStreetMap centrada en Oaxaca.
- **Usuario_Básico**: Persona registrada con verificación de correo electrónico.
- **Usuario_Verificado**: Persona con identidad o trayectoria confirmada cuyas confirmaciones tienen peso adicional.
- **Moderador**: Usuario con permisos para editar, validar o archivar cualquier Incidente.
- **Zona**: Región geográfica del estado de Oaxaca (ej. Valles Centrales, Istmo, Sierra Norte).
- **Seed_Data**: Conjunto de datos estáticos de incidentes reales de Oaxaca cargados para el MVP/demo.
- **Estado_Incidente**: Valor del ciclo de vida de un Incidente: `programado`, `activo`, `finalizado`, `cancelado`, `no_verificado`, `rechazado`.
- **Suscripción**: Seguimiento activo de un Incidente o Zona para recibir alertas de cambios.

---

## Requirements

### Requisito 1: Landing Page

**User Story:** Como conductor o viajero que transita Oaxaca, quiero encontrar una página de inicio clara y atractiva, para entender rápidamente qué es ViaLibre y acceder al mapa o reportar un incidente.

#### Criterios de Aceptación

1. THE ViaLibre SHALL mostrar una sección hero con el tagline de la aplicación, un botón "Ver bloqueos ahora" que navega a `/mapa` y un botón "Reportar un bloqueo" que navega al formulario de reporte.
2. THE ViaLibre SHALL incluir en la landing una sección "El problema" con una descripción empática de máximo 3 líneas sobre la situación de bloqueos en Oaxaca.
3. THE ViaLibre SHALL mostrar una sección "Cómo funciona" con los 4 pasos: ver el mapa, confirmar o reportar, suscribirse y recibir alertas.
4. THE ViaLibre SHALL presentar una sección de diferenciadores que destaque bloqueos programados, verificación comunitaria y rutas alternativas.
5. THE ViaLibre SHALL mostrar métricas de confianza visibles: número total de confirmaciones registradas, cantidad de moderadores activos y número de usuarios verificados.
6. THE ViaLibre SHALL incluir un CTA final antes del footer que repita las opciones de ver el mapa y reportar un incidente.
7. THE ViaLibre SHALL renderizar la landing page con los colores de la paleta oaxaqueña: terracota/rojo óxido, verde oaxaqueño, amarillo mostaza/ocre, morado grana cochinilla como acento, y fondo hueso/papel.
8. THE ViaLibre SHALL aplicar en títulos de la landing una tipografía con carácter visual diferenciado y en los textos de datos una tipografía sans-serif humanista.

---

### Requisito 2: Mapa Interactivo en Tiempo Real

**User Story:** Como conductor que está por salir de viaje, quiero ver un mapa de Oaxaca con todos los incidentes activos marcados visualmente, para decidir qué ruta tomar antes de salir.

#### Criterios de Aceptación

1. THE Mapa_Interactivo SHALL renderizarse con Leaflet y OpenStreetMap centrado en las coordenadas del estado de Oaxaca (latitud 17.0732° N, longitud 96.7266° O) con zoom inicial que muestre el estado completo.
2. WHEN el Mapa_Interactivo carga, THE Mapa_Interactivo SHALL mostrar todos los Incidentes del Seed_Data como marcadores posicionados en sus coordenadas geográficas reales.
3. THE Mapa_Interactivo SHALL diferenciar los marcadores por color según el Estado_Incidente: terracota/rojo óxido para `activo`, ocre/amarillo para `programado`, y verde apagado para `finalizado`.
4. WHEN el usuario hace clic en un marcador, THE Mapa_Interactivo SHALL mostrar un panel o popup con: título del Incidente, motivo/tipo, zona, carretera afectada, hora de reporte, Confirmación_Neta y descripción de ruta alternativa si existe.
5. THE ViaLibre SHALL proveer filtros que permitan al usuario mostrar u ocultar Incidentes por Estado_Incidente y por Zona.
6. WHEN el usuario aplica un filtro, THE Mapa_Interactivo SHALL actualizar los marcadores visibles en menos de 500 milisegundos sin recargar la página.
7. THE Mapa_Interactivo SHALL funcionar correctamente dentro de un componente `<ClientOnly>` de Nuxt para evitar errores de SSR por el uso de `window` en Leaflet.
8. WHEN la lista de Incidentes se actualiza en tiempo real via WebSocket, THE Mapa_Interactivo SHALL reflejar el nuevo estado del Incidente actualizado sin recargar la página completa.

---

### Requisito 3: Seed Data de Incidentes Reales de Oaxaca

**User Story:** Como usuario del MVP demo, quiero ver incidentes reales de Oaxaca en el mapa desde el primer momento, para evaluar la utilidad real de ViaLibre sin necesidad de cargar datos manualmente.

#### Criterios de Aceptación

1. THE Seed_Data SHALL contener un mínimo de 8 Incidentes con ubicaciones geográficas reales del estado de Oaxaca, incluyendo carreteras identificadas por nombre oficial (ej. Carretera Federal 190, Carretera 175 Oaxaca-Puerto Ángel).
2. THE Seed_Data SHALL incluir al menos un Incidente de cada tipo: `bloqueo_sindical`, `bloqueo_comunitario`, `manifestacion`, `derrumbe` y `obra`.
3. THE Seed_Data SHALL incluir Incidentes con Estado_Incidente `activo`, `programado` y `finalizado` para demostrar la diferenciación visual en el mapa.
4. THE Seed_Data SHALL asignar a cada Incidente una ruta alternativa textual cuando exista una vía alterna conocida para esa carretera.
5. THE Seed_Data SHALL ser accesible como un archivo de datos estáticos TypeScript o JSON importable por el frontend sin necesidad de petición HTTP a un backend.

---

### Requisito 4: Formulario de Reporte de Incidente

**User Story:** Como miembro de la comunidad que acaba de encontrar un bloqueo, quiero reportarlo fácilmente desde mi teléfono, para que otros conductores puedan enterarse de inmediato.

#### Criterios de Aceptación

1. THE Formulario_Reporte SHALL solicitar los campos obligatorios: ubicación (coordenadas via pin en mapa o texto de autocompletar), tipo de Incidente, y confirmación de si el evento es programado o no.
2. THE Formulario_Reporte SHALL solicitar el campo opcional: descripción de texto libre de hasta 500 caracteres.
3. THE Formulario_Reporte SHALL permitir adjuntar una fotografía opcional en formatos JPEG o PNG con un tamaño máximo de 5 MB.
4. THE Formulario_Reporte SHALL permitir el envío sin cuenta de usuario, indicando visualmente que el reporte quedará en estado `no_verificado` hasta revisión por Moderador.
5. WHEN el usuario está autenticado y envía el formulario, THE Formulario_Reporte SHALL registrar el reporte asociado a su cuenta y sumar la acción a su historial de reputación.
6. IF el usuario omite un campo obligatorio, THEN THE Formulario_Reporte SHALL mostrar un mensaje de error específico junto al campo faltante sin limpiar los demás campos completados.
7. WHEN el formulario se envía exitosamente, THE Formulario_Reporte SHALL mostrar una confirmación con el mensaje "Tu reporte fue enviado. Estará visible cuando la comunidad lo confirme." y un enlace para ver el Incidente en el mapa.
8. THE Formulario_Reporte SHALL ser usable en dispositivos móviles con pantallas de 320px de ancho mínimo y campos suficientemente grandes para interacción táctil.

---

### Requisito 5: Sistema de Confirmaciones Comunitarias

**User Story:** Como conductor que pasó por un punto de bloqueo, quiero confirmar o negar un incidente reportado, para que la información del mapa sea más confiable para todos.

#### Criterios de Aceptación

1. THE Sistema_Confirmaciones SHALL mostrar en cada Incidente dos acciones visibles: "Confirmo que sigue así" y "Ya no está".
2. THE Sistema_Confirmaciones SHALL mostrar el conteo de Confirmación_Neta actualizado junto a los botones de acción.
3. WHEN un usuario autenticado envía una Confirmación o Negación, THE Sistema_Confirmaciones SHALL registrarla y actualizar el contador visible en menos de 2 segundos.
4. WHEN un usuario autenticado intenta confirmar un Incidente que ya confirmó previamente, THE Sistema_Confirmaciones SHALL reemplazar la Confirmación anterior con la nueva en lugar de crear una duplicada.
5. WHEN la Confirmación_Neta de un Incidente en estado `no_verificado` alcanza 3 o más, THE Sistema_Confirmaciones SHALL cambiar el Estado_Incidente a `activo` automáticamente.
6. WHEN la Confirmación_Neta de un Incidente en estado `no_verificado` llega a −3 o menos, THE Sistema_Confirmaciones SHALL cambiar el Estado_Incidente a `rechazado` automáticamente.
7. THE Sistema_Confirmaciones SHALL permitir que usuarios no autenticados vean los contadores de confirmaciones pero no envíen confirmaciones, mostrando una invitación a registrarse al intentar confirmar.
8. THE Sistema_Confirmaciones SHALL calcular la Confirmación_Neta como: `confirmaciones_count − denegaciones_count`.

---

### Requisito 6: Suscripciones y Notificaciones

**User Story:** Como conductor que viaja frecuentemente la Carretera 190, quiero suscribirme a alertas de esa zona, para recibir una notificación antes de salir cuando haya un bloqueo activo.

#### Criterios de Aceptación

1. THE Sistema_Suscripciones SHALL permitir a un Usuario_Básico suscribirse a un Incidente individual para recibir alertas de cambios de Estado_Incidente.
2. THE Sistema_Suscripciones SHALL permitir a un Usuario_Básico suscribirse a una Zona completa para recibir alertas de nuevos Incidentes en esa zona.
3. WHEN el Estado_Incidente cambia (ej. de `programado` a `activo`), THE Sistema_Notificaciones SHALL enviar una notificación a todos los usuarios suscritos a ese Incidente o a su Zona dentro de los primeros 30 segundos del cambio.
4. THE Sistema_Notificaciones SHALL soportar entrega de notificaciones por al menos uno de los canales: push (WebSocket en sesión activa) o correo electrónico.
5. THE Sistema_Notificaciones SHALL registrar cada notificación enviada en la entidad Notificacion con los campos: `usuario_id`, `incidente_id`, `tipo`, `titulo`, `cuerpo` y `created_at`.
6. WHEN el usuario lee una notificación, THE Sistema_Notificaciones SHALL registrar la fecha y hora de lectura en el campo `leida_en` de la entidad Notificacion.
7. IF un usuario está suscrito a una Zona e individualmente a un Incidente de esa misma Zona, THEN THE Sistema_Notificaciones SHALL enviar una sola notificación por evento para ese usuario, no dos.

---

### Requisito 7: Cuentas de Usuario y Roles

**User Story:** Como persona que usa ViaLibre regularmente, quiero crear una cuenta para que mis reportes y confirmaciones tengan más peso y credibilidad en la comunidad.

#### Criterios de Aceptación

1. THE Sistema_Cuentas SHALL permitir el registro con correo electrónico y contraseña, enviando un correo de verificación antes de activar la cuenta.
2. WHEN un usuario intenta iniciar sesión con credenciales incorrectas, THE Sistema_Cuentas SHALL mostrar el mensaje "Correo o contraseña incorrectos" sin especificar cuál de los dos es incorrecto.
3. THE Sistema_Cuentas SHALL asignar el rol `basico` por defecto a todo usuario recién registrado.
4. WHEN un Moderador accede al panel de moderación, THE Panel_Moderacion SHALL mostrar todos los Incidentes en estado `no_verificado` con la opción de marcarlos como `rechazado` o promoverlos a `activo`.
5. THE Panel_Moderacion SHALL permitir a un Moderador editar el título, descripción, tipo, zona y carretera de cualquier Incidente en cualquier Estado_Incidente no terminal.
6. IF un Moderador marca un Incidente como `rechazado`, THEN THE Sistema_Incidentes SHALL actualizar el Estado_Incidente a `rechazado` y notificar al usuario que lo reportó.
7. THE Sistema_Cuentas SHALL almacenar la contraseña del usuario utilizando un algoritmo de hash seguro (bcrypt con costo mínimo 10) y nunca almacenarla en texto plano.

---

### Requisito 8: Ciclo de Vida de Incidentes

**User Story:** Como moderador de ViaLibre, quiero que los incidentes tengan un ciclo de vida claro con transiciones bien definidas, para mantener el mapa limpio y la información confiable.

#### Criterios de Aceptación

1. THE Sistema_Incidentes SHALL aceptar solo las transiciones de estado válidas: `programado → activo`, `programado → cancelado`, `activo → finalizado`, `no_verificado → activo`, `no_verificado → rechazado`.
2. IF el sistema intenta realizar una transición de estado no válida (ej. `finalizado → activo`), THEN THE Sistema_Incidentes SHALL rechazar la operación y devolver un error descriptivo sin modificar el Estado_Incidente.
3. WHEN un Incidente programado llega a su `inicio_estimado` sin haber sido cancelado, THE Sistema_Incidentes SHALL cambiar automáticamente su estado a `activo`.
4. WHEN un Incidente `activo` llega a su `fin_estimado`, THE Sistema_Incidentes SHALL notificar a los moderadores para que confirmen si el Incidente debe marcarse como `finalizado`.
5. THE Sistema_Incidentes SHALL registrar la fecha y hora exacta de cada cambio de estado en el campo `updated_at` del Incidente.
6. THE Sistema_Incidentes SHALL publicar un evento WebSocket en el canal `incidentes.{id}` con tipo `IncidenteActualizado` cada vez que el Estado_Incidente cambie.

---

### Requisito 9: Accesibilidad y Rendimiento en Redes Lentas

**User Story:** Como conductor en una zona rural de Oaxaca con conexión 3G, quiero que ViaLibre cargue rápido y sea usable, para poder consultar el mapa antes de tomar una carretera remota.

#### Criterios de Aceptación

1. THE ViaLibre SHALL lograr un Time to First Contentful Paint (FCP) menor a 3 segundos en una conexión simulada de 3G (1.6 Mbps bajada) medido con Lighthouse.
2. THE ViaLibre SHALL servir las imágenes en formato WebP con compresión, y las fotografías de Incidentes adjuntas por usuarios no deberán superar 200 KB al mostrarse en el mapa.
3. THE ViaLibre SHALL funcionar con JavaScript deshabilitado para las rutas de solo lectura (landing y mapa en modo estático), mostrando al menos los datos de los Incidentes del Seed_Data.
4. THE ViaLibre SHALL cumplir con un nivel mínimo de accesibilidad WCAG 2.1 AA, incluyendo: etiquetas ARIA en controles de formulario, contraste de color mínimo 4.5:1 en texto normal y navegación por teclado en el mapa.
5. THE ViaLibre SHALL mostrar todos los textos de interfaz en español mexicano coloquial oaxaqueño, usando un tono cercano y directo (ej. "Antes de salir, échale un ojo a ViaLibre").

---

### Requisito 10: Rutas Alternativas

**User Story:** Como viajero que encuentra un bloqueo activo en el mapa, quiero ver una ruta alternativa sugerida, para poder llegar a mi destino sin perder tiempo buscando en otro lado.

#### Criterios de Aceptación

1. THE Sistema_RutasAlternativas SHALL mostrar la ruta alternativa en el popup del Incidente cuando el campo `ruta_alternativa` del Incidente no esté vacío.
2. THE Sistema_RutasAlternativas SHALL describir la ruta alternativa en texto libre indicando la vía, sentido o instrucciones básicas (ej. "Tomar la desviación por Tlacolula hacia la carretera 190D").
3. WHERE el Incidente tiene ruta alternativa definida, THE Mapa_Interactivo SHALL mostrar un ícono diferenciador en el marcador del Incidente para indicar que hay ruta alternativa disponible.
4. THE Seed_Data SHALL incluir rutas alternativas textuales para al menos 5 de los Incidentes de ejemplo, basadas en vías reales del estado de Oaxaca.

