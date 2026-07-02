# renyn-componente-arbol

Lee `docs/CONTEXTO.md` para el contexto completo del proyecto (qué es, arquitectura, tipos de nodo, estado de cada cámara, pendientes).

## Estructura de carpetas

- **raíz**: el componente (`index.html` + `renyn-asistente.php`, espejos) y la config/tooling (`package.json`, `biome.json`, `.gitignore`, `.editorconfig`, `CLAUDE.md`). ⚠️ `index.html` debe quedarse en la raíz: GitHub Pages sirve el demo desde ahí (rama `master`, carpeta raíz).
- **`docs/`**: documentación (`CONTEXTO.md` y las propuestas `PROPUESTA-*.txt`, estas últimas gitignored).
- **`salidas-maker/`**: material fuente (manuales); zona restringida (ver abajo).

## Reglas operativas

- **Linter/formatter**: Biome 2.5.1 — `npm run lint`. Indentación con tabs, comillas dobles.
- **Stack**: HTML/CSS/JS vanilla en la raíz (`index.html` + `renyn-asistente.php`, espejos). Sin frameworks, sin build step.
- **`salidas-maker/`**: zona restringida — pedir permiso explícito al usuario antes de leer o modificar cualquier archivo de esta carpeta.
- **`docs/PROPUESTA-*.txt`**: documentación **solo informativa** (propuestas del chatbot). NO leerla ni usarla como contexto al desarrollar o mejorar el componente/asistente; consultarla únicamente si el usuario lo pide de forma explícita.
