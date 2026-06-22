# Portar los últimos cambios funcionales a la versión Ubuntu

Esta guía documenta los cambios realizados **después del commit Tahoe `575b745`**. La lógica es independiente del tema visual y puede reaplicarse si el portfolio vuelve a la versión Ubuntu.

El código actual está concentrado en `index.html`.

## Cambios incluidos

1. Las ventanas minimizadas ya no generan badges independientes.
2. El punto del Dock indica si una sección está abierta o minimizada.
3. El icono del Dock enfoca una ventana abierta o restaura una minimizada.
4. Explore muestra el perfil de GitHub dentro de una ventana del portfolio.
5. El menú del gestor de archivos solo aparece cuando una carpeta está maximizada.
6. Los proyectos sin imagen muestran su emoji en la carpeta y el detalle, pero no en la preview lateral.

## 1. Minimizado gestionado únicamente desde el Dock

### Comportamiento

- No mostrar botones o badges adicionales para las ventanas minimizadas.
- Mantener la ventana en el array `minimized`.
- Añadir la clase `running` al icono correspondiente mientras haya una ventana abierta o minimizada.
- Al pulsar un icono del Dock:
  1. enfocar una ventana visible de esa sección;
  2. si no hay una visible, restaurar la última minimizada;
  3. si no existe ninguna, abrir la ventana principal.

### Piezas que deben conservarse

- `openWindows` y `minimized` como fuentes de estado.
- `windowBelongsTo(target, win)` para relacionar cada icono con sus ventanas.
- `syncNav()` para actualizar las clases `active` y `running`.
- La búsqueda inversa con `openWindows.slice().reverse().find(...)` y `minimized.slice().reverse().find(...)`.
- `renderTaskbar()` vacío y `.taskbar { display: none; }`, o eliminar por completo el taskbar y sus llamadas.

En Ubuntu, el indicador puede conservar el color naranja del tema:

```css
.desktop-icon.running::after {
  background: #e95420;
}
```

## 2. Perfil de GitHub integrado

GitHub no permite incrustar `https://github.com/marinucs` en un `iframe`. Sus respuestas incluyen `X-Frame-Options: deny` y una política CSP con `frame-ancestors 'none'`.

La alternativa implementada es una ventana interna que consume la API pública:

```text
https://api.github.com/users/marinucs
https://api.github.com/users/marinucs/repos?sort=updated&per_page=12
```

### HTML necesario

- Convertir Explore de enlace externo a botón con `data-open="github"`.
- Añadir la ventana `#win-github` con los mismos controles de cerrar, minimizar y maximizar que el resto.
- Mantener `#githubProfile` como contenedor del contenido dinámico.
- El enlace externo debe ser secundario, no la acción principal del icono Explore.

### JavaScript necesario

- `loadGitHubProfile()` carga el perfil y los repositorios una sola vez.
- `renderGitHubProfile(user, repos)` construye la vista integrada.
- `escapeHtml()` debe aplicarse a todos los campos recibidos de GitHub.
- Si la API falla o alcanza su límite, mostrar un mensaje y un enlace externo de fallback.
- Añadir `win-github` a `windowBelongsTo()` y a la selección activa de `syncNav()`.

La API pública tiene límite para peticiones sin autenticar. La variable `githubLoaded` evita solicitudes repetidas durante una misma sesión.

### Adaptación visual para Ubuntu

Reutilizar la estructura, pero sustituir el Liquid Glass por superficies GNOME:

```css
#win-github .window-body {
  background: #f7f7f7;
}

.github-addressbar,
.github-repo {
  background: #ffffff;
  border-color: #d6d1d5;
}

.github-repo-name,
.github-external,
.github-open-profile {
  color: #e95420;
}
```

## 3. Menú del gestor de archivos condicionado al maximizado

En Tahoe, la barra muestra `Finder`. En Ubuntu debe mostrarse el equivalente, por ejemplo:

```html
<span class="menu-app">Files</span>
```

La lógica no debe comprobar nombres concretos de ventanas. Debe comprobar sus clases:

```js
const fileManagerVisible = openWindows.some(function (win) {
  return win.classList.contains('is-folder') &&
    win.classList.contains('maximized');
});
```

Aplicar después una clase al `body`. En una futura limpieza conviene renombrar `finder-active` a `file-manager-active`:

```css
.menu-left {
  opacity: 0;
  visibility: hidden;
}

body.file-manager-active .menu-left {
  opacity: 1;
  visibility: visible;
}
```

`toggleMaximize()`, `openWindow()`, `closeWindow()`, `minimizeWindow()` y `restoreWindow()` deben terminar actualizando este estado mediante `syncNav()`.

## 4. Emojis de proyectos sin imagen

Cada proyecto sin `img` conserva su propiedad `emoji` en `PROJECTS`.

### Carpeta Projects

`renderFolderGrid()` debe seguir usando el emoji como glyph cuando no existe una imagen:

```js
item.glyph || item.emoji || '📄'
```

### Ventana de detalle

`openFolderContent()` debe conservar el placeholder `.proj-thumb.ph` con `item.emoji`.

### Preview lateral

No debe renderizarse ningún placeholder para Projects:

```js
else if (key !== 'projects' && !item.previewRows) {
  // Placeholder permitido para otras carpetas, no para Projects.
}
```

Así se evita mostrar el mismo emoji dos veces: permanece en el icono de la carpeta y en el detalle, pero desaparece de la preview.

## Orden recomendado para el port

1. Recuperar o reconstruir el tema Ubuntu.
2. Mantener las estructuras `PROJECTS`, `FOLDERS`, `openWindows` y `minimized`.
3. Portar primero `windowBelongsTo()` y la restauración desde el Dock.
4. Ocultar o eliminar el antiguo taskbar de minimizados.
5. Añadir la ventana integrada de GitHub y sus funciones.
6. Adaptar `Finder` a `Files` y renombrar la clase de estado si se desea.
7. Aplicar la condición de preview de Projects.
8. Rehacer únicamente los colores y superficies CSS con la paleta Ubuntu.

## Verificación manual

- Abrir, minimizar y restaurar About, Contact, Projects y Explore desde el Dock.
- Confirmar que solo aparece un punto y que no se crea ningún badge flotante.
- Maximizar About o Projects: debe aparecer `Files` y su menú.
- Desmaximizar o minimizar la carpeta: el menú debe desaparecer.
- Abrir Explore: debe cargar avatar, perfil y repositorios sin abandonar la página.
- Simular un fallo de red: debe aparecer el fallback externo.
- Seleccionar Gameskins, FilmApp, Android Chat, RPC To-Do List y Tic Tac Toe:
  - emoji visible en la carpeta;
  - preview lateral sin emoji;
  - emoji visible al abrir el detalle.

## Comprobaciones rápidas

```bash
git diff --check
rg -n "windowBelongsTo|githubLoaded|finder-active|key !== 'projects'" index.html
```

