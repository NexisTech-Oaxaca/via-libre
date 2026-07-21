---
inclusion: manual
---

# Accesibilidad Web (a11y)

Guía de referencia para implementar accesibilidad en HTML/CSS/JS siguiendo los estándares WCAG 2.1 nivel AA.

---

## Elementos semánticos

Usa etiquetas HTML que describan su propósito. El lector de pantalla los anuncia correctamente sin necesidad de ARIA adicional.

```html
<header>        <!-- Cabecera del sitio o sección -->
<nav>           <!-- Navegación principal o secundaria -->
<main>          <!-- Contenido principal (único por página) -->
<article>       <!-- Contenido autónomo y reutilizable -->
<section>       <!-- Agrupación temática (requiere heading) -->
<aside>         <!-- Contenido complementario -->
<footer>        <!-- Pie del sitio o sección -->
<h1>–<h6>       <!-- Jerarquía de encabezados, sin saltarse niveles -->
<ul> / <ol>     <!-- Listas no ordenadas / ordenadas -->
<button>        <!-- Acciones; nunca usar <div> o <span> para botones -->
<a href="...">  <!-- Navegación; siempre con href válido -->
```

**Reglas clave:**
- Una sola etiqueta `<main>` por página.
- Los headings deben seguir jerarquía lógica (`h1` → `h2` → `h3`), sin saltos.
- Nunca usar `<table>` para layout; solo para datos tabulares.

---

## Atributos ARIA

Usa ARIA solo cuando el HTML semántico no sea suficiente. La regla es: **primero HTML nativo, ARIA como último recurso**.

### Roles más comunes
```html
role="alert"          <!-- Mensajes urgentes (anunciados automáticamente) -->
role="dialog"         <!-- Modales y popups -->
role="tablist" / "tab" / "tabpanel"  <!-- Componente de pestañas -->
role="menu" / "menuitem"             <!-- Menús de acción -->
role="status"         <!-- Mensajes de estado no urgentes -->
role="progressbar"    <!-- Barras de progreso -->
```

### Atributos de estado y propiedades
```html
aria-label="Cerrar modal"         <!-- Nombre accesible cuando no hay texto visible -->
aria-labelledby="id-del-titulo"   <!-- Nombre accesible apuntando a otro elemento -->
aria-describedby="id-descripcion" <!-- Descripción adicional -->
aria-expanded="true|false"        <!-- Estado abierto/cerrado (menús, acordeones) -->
aria-hidden="true"                <!-- Ocultar del árbol de accesibilidad (iconos decorativos) -->
aria-live="polite|assertive"      <!-- Regiones que se actualizan dinámicamente -->
aria-disabled="true"              <!-- Elemento deshabilitado semánticamente -->
aria-required="true"              <!-- Campo obligatorio -->
aria-invalid="true"               <!-- Campo con error de validación -->
aria-controls="id-panel"          <!-- Elemento que controla otro -->
aria-current="page|step|location" <!-- Elemento activo en navegación -->
```

### Ejemplo: botón que expande/colapsa
```html
<button
  aria-expanded="false"
  aria-controls="menu-principal"
>
  Menú
</button>
<ul id="menu-principal" hidden>
  <!-- items -->
</ul>
```

---

## Contraste de color (WCAG AA)

### Requisitos mínimos
| Tipo de texto | Ratio mínimo |
|---|---|
| Texto normal (< 18pt / < 14pt bold) | 4.5:1 |
| Texto grande (≥ 18pt / ≥ 14pt bold) | 3:1 |
| Componentes UI e iconos significativos | 3:1 |
| Texto decorativo / logos | Sin requisito |

### Herramientas de verificación
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/)
- DevTools de Chrome/Firefox (pestaña Accessibility)
- Extensión axe DevTools

### Ejemplo de paleta accesible
```css
/* ✅ Texto oscuro sobre fondo claro — ratio ~9:1 */
color: #1a1a1a;
background-color: #ffffff;

/* ✅ Texto blanco sobre azul primario — ratio ~4.6:1 */
color: #ffffff;
background-color: #1d4ed8;

/* ❌ Gris claro sobre blanco — ratio ~2.3:1 */
color: #9ca3af;
background-color: #ffffff;
```

---

## Focus visible

Todo elemento interactivo debe mostrar un indicador de foco claro cuando se navega con teclado.

```css
/* Nunca eliminar outline sin reemplazarlo */
/* ❌ Malo */
:focus {
  outline: none;
}

/* ✅ Bueno: outline personalizado que supera 3:1 de contraste */
:focus-visible {
  outline: 2px solid #1d4ed8;
  outline-offset: 2px;
  border-radius: 2px;
}

/* Ocultar solo para usuarios de ratón, mantener para teclado */
:focus:not(:focus-visible) {
  outline: none;
}
```

**Reglas clave:**
- Usa `:focus-visible` en lugar de `:focus` para no afectar a usuarios de ratón.
- El indicador de foco debe tener al menos ratio de contraste 3:1 contra el fondo adyacente.
- No uses `tabindex="-1"` en elementos que deben ser alcanzables con teclado.

---

## Labels asociados a inputs

Cada campo de formulario debe tener un label asociado explícitamente.

```html
<!-- ✅ Asociación explícita con for/id -->
<label for="email">Correo electrónico</label>
<input type="email" id="email" name="email" required>

<!-- ✅ Asociación implícita (input dentro del label) -->
<label>
  Contraseña
  <input type="password" name="password">
</label>

<!-- ✅ Cuando no hay label visible (buscadores) -->
<input
  type="search"
  aria-label="Buscar productos"
  placeholder="Buscar..."
>

<!-- ❌ Placeholder como único label — no es suficiente -->
<input type="text" placeholder="Nombre completo">
```

### Mensajes de error
```html
<label for="email">Correo electrónico</label>
<input
  type="email"
  id="email"
  aria-describedby="email-error"
  aria-invalid="true"
>
<span id="email-error" role="alert">
  Ingresa un correo válido.
</span>
```

---

## Texto alternativo en imágenes

```html
<!-- ✅ Imagen informativa: describe su contenido -->
<img src="mapa-ruta.png" alt="Mapa con la ruta desde la estación Norte hasta el parque central">

<!-- ✅ Imagen funcional (dentro de un link/button): describe la acción -->
<a href="/inicio">
  <img src="logo.svg" alt="Via Libre — Ir al inicio">
</a>

<!-- ✅ Imagen decorativa: alt vacío para que el lector la ignore -->
<img src="separador.png" alt="">

<!-- ✅ Iconos SVG decorativos -->
<svg aria-hidden="true" focusable="false">...</svg>

<!-- ✅ Iconos SVG significativos -->
<svg aria-label="Advertencia" role="img">...</svg>

<!-- ❌ Alt genérico o redundante -->
<img src="foto.jpg" alt="imagen">
<img src="foto.jpg" alt="foto de foto.jpg">
```

**Reglas clave:**
- Alt nunca debe describir la imagen técnicamente ("imagen PNG de..."), sino su significado.
- Imágenes de texto deben tener en alt el mismo texto que muestran.
- Para imágenes complejas (gráficas, infografías) considera una descripción larga con `aria-describedby` o un párrafo adyacente.

---

## Navegación por teclado

### Orden del foco
El foco debe seguir un orden lógico (generalmente el orden del DOM). Evita `tabindex` positivos.

```html
<!-- ✅ Orden natural del DOM -->
tabindex="0"   <!-- Incluir en el orden de tabulación -->
tabindex="-1"  <!-- Focusable por JS, no por Tab -->

<!-- ❌ Evitar: rompe el orden natural -->
tabindex="3"
tabindex="10"
```

### Componentes interactivos con teclado
```html
<!-- Botones: Enter y Space los activan -->
<button type="button" onclick="abrirModal()">Abrir</button>

<!-- Links: Enter los activa -->
<a href="/ruta">Ir a ruta</a>

<!-- Trampas de foco en modales -->
<!-- El foco debe quedar atrapado dentro del modal mientras esté abierto -->
```

### Ejemplo: skip link (saltar al contenido principal)
```html
<!-- Primer elemento del body — visible solo al recibir foco -->
<a href="#contenido-principal" class="skip-link">
  Saltar al contenido principal
</a>

<main id="contenido-principal">
  ...
</main>
```

```css
.skip-link {
  position: absolute;
  top: -100%;
  left: 0;
  padding: 8px 16px;
  background: #1d4ed8;
  color: #fff;
  text-decoration: none;
  z-index: 9999;
}

.skip-link:focus {
  top: 0;
}
```

### Gestión de foco en interacciones dinámicas
```js
// Al abrir un modal: mover foco al primer elemento interactivo
const modal = document.getElementById('modal');
modal.removeAttribute('hidden');
modal.querySelector('button, [href], input').focus();

// Al cerrar un modal: devolver foco al elemento que lo abrió
const triggerBtn = document.getElementById('abrir-modal');
modal.setAttribute('hidden', '');
triggerBtn.focus();

// Al cargar contenido dinámico: anunciar con live region
const status = document.getElementById('status');
status.textContent = '10 resultados encontrados';
```

---

## Checklist rápido

Antes de hacer PR, verifica:

- [ ] Todos los inputs tienen `<label>` asociado
- [ ] Imágenes tienen `alt` apropiado (vacío si decorativas)
- [ ] Contraste de texto supera 4.5:1 (normal) o 3:1 (grande)
- [ ] Focus visible en todos los elementos interactivos
- [ ] La página es navegable completamente con Tab y Enter/Space
- [ ] No hay `outline: none` sin reemplazo visible
- [ ] Headings en orden jerárquico sin saltos
- [ ] Modales atrapan el foco y lo devuelven al cerrarse
- [ ] Mensajes de error están asociados al campo con `aria-describedby`
- [ ] Contenido dinámico usa `aria-live` o `role="alert"`
