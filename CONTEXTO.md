# Resumen del proyecto: Asistente guiado para cámaras de fototrampeo

## Qué es
Un widget interactivo tipo "árbol de decisiones" (preguntas con botones, estilo encuesta guiada) para insertar en WordPress vía **WPCode** como snippet HTML independiente. Ayuda al cliente a instalar/configurar sus cámaras de fototrampeo sin necesidad de contactar con soporte, salvo que sea necesario.

## Stack técnico
- **Un único archivo HTML autocontenido** (HTML + CSS + JavaScript vanilla, sin frameworks ni dependencias externas).
- Sin llamadas a APIs externas, sin librerías de CDN, sin fuentes externas (se usa una pila de fuentes de sistema: `Poppins, Century Gothic, Futura, Verdana, sans-serif`).
- Los iconos son **SVG inline** (no `<img>` con URLs externas).
- Pensado para pegarse directamente en un snippet de tipo "HTML" en WPCode (las etiquetas `<!DOCTYPE html><html><head>` y `</body></html>` se pueden quitar si WPCode da problemas con ellas).

## Estética
Diseño dark mode: fondo `#0e0e0e`, bordes redondeados grandes (25px en el contenedor, 50px en botones tipo píldora), botones blancos con texto en negrita (`font-weight:700`), acentos en **rojo** (`#c0392b` / hover `#e74c3c`) para los hovers de botones, y naranja (`#ff6a3d`) reservado específicamente para las cajas de "rama pendiente de creación". Ancho fijo configurable vía variable CSS `--ancho-widget` (actualmente 520px, con `max-width:100%` para que sea responsive en móvil).

### Mejoras de estilo aplicadas
- **Animación fade-in + slide-up (200ms)** al navegar entre nodos (CSS puro, reflow forzado para que se repita en cada transición).
- **Chevron `›`** en los botones de pregunta, alineado a la derecha, que se desplaza al hacer hover.
- **Bloque de instrucciones de pasos** con fondo `#141414` y borde izquierdo gris, diferenciando visualmente el contenido instructivo.
- **Botón "Siguiente" más prominente** en nodos paso: padding extra, `font-size` ligeramente mayor y sombra sutil.
- **Breadcrumb con overflow oculto** (`text-overflow: ellipsis`) en flujos largos, en vez de scroll horizontal.

---

## Arquitectura del código (importante para seguir editando)

### Dos bloques `<script>` separados

El archivo está dividido en dos secciones claramente delimitadas:

#### Bloque 1 — DATOS (`<!-- BLOQUE 1 — DATOS Y CONFIGURACIÓN -->`)
Contiene todo lo que necesita edición frecuente:
- **`CONFIG`**: objeto con los datos de contacto y URLs (ver abajo).
- **`ARBOL`**: el árbol de decisiones completo.

**No necesitas tocar el Bloque 2 para añadir o modificar contenido del árbol.**

#### Bloque 2 — MOTOR (`<!-- BLOQUE 2 — MOTOR DEL ASISTENTE -->`)
Contiene la lógica del asistente:
- Estado de sesión (`historial`, `contexto`).
- Iconos SVG (`ICONOS`, función `icon()`).
- Funciones de navegación (`irANodo`, `volverAtras`, `reiniciar`).
- Mapa de renderizadores (`RENDERIZADORES`).
- Función principal `renderizar()`.

---

## CONFIG — datos de contacto centralizados

Todos los datos de contacto y URLs viven en un único lugar al inicio del Bloque 1:

```js
const CONFIG = {
  whatsapp:    "https://wa.me/34600000000",   // ← cambia por tu número real
  email:       "soporte@tudominio.com",        // ← cambia por tu email real
  manualesUrl: "https://tudominio.com/manuales" // ← cambia por tu URL real
};
```

El motor los referencia automáticamente en todos los puntos donde aparecen (bloque de contacto, enlace de soporte del header, etc.). **No busques y reemplaces manualmente**: con cambiar `CONFIG` es suficiente.

---

## Arquitectura de datos — el objeto ARBOL

Cada clave es el ID único de un nodo. Hay 4 tipos:

### 1. `tipo: "pregunta"`
El cliente elige entre botones. El campo `destino` de cada opción puede ser:
- **Un string**: ID del nodo destino. `destino: "mi_nodo"`
- **Una función**: recibe el objeto `contexto` de sesión y devuelve un string. Útil para enrutar de forma condicional según respuestas anteriores:
  ```js
  destino: ctx => ctx.instalar_modelo?.texto === "Cherokee"
                   ? "instalar_cherokee_hw"
                   : "pendiente_apache_instalar"
  ```

```js
{
  tipo: "pregunta",
  breadcrumbLabel: "texto corto",   // opcional
  pregunta: "Texto de la pregunta",
  incluirNinguna: true/false,
  ningunaDestino: "idNodo",         // solo si incluirNinguna: true
  opciones: [ { texto:"...", destino: "idNodo" | fn }, ... ]
}
```

### 2. `tipo: "paso"`
Instrucción intermedia. Muestra texto + botón "Siguiente". Sin pregunta de satisfacción (el proceso aún no ha terminado).

```js
{
  tipo: "paso",
  breadcrumbLabel: "texto corto",   // opcional
  texto: "HTML de instrucciones",
  imagen: "",                       // opcional
  manualUrl: "",                    // opcional
  manualTexto: "",                  // opcional
  siguiente: "idNodo",
  textoBoton: "Siguiente →"         // opcional, por defecto "Siguiente"
}
```

### 3. `tipo: "final"`
Hoja real del árbol. Muestra el resultado y pregunta "¿Has conseguido configurar/instalar tu cámara?" → Sí (despedida) / No (muestra manuales + contacto + botón de reinicio).

```js
{
  tipo: "final",
  breadcrumbLabel: "texto corto",   // opcional
  resultadoTexto: "HTML del resultado",
  imagen: "",                       // opcional
  manualUrl: "",                    // opcional
  manualTexto: ""                   // opcional
}
```

### 4. `tipo: "pendiente"`
Placeholder visual naranja para ramas sin contenido todavía. Para completarla: cambia `tipo` a `"paso"` o `"final"` y rellena los campos.

```js
{ tipo: "pendiente", breadcrumbLabel: "texto corto" }
```

### 5. `tipo: "errores"`
Panel interactivo de códigos de error. Muestra una cuadrícula de píldoras clicables; al pulsar cada una aparece un bloque con qué significa el error, posible causa y solución. No navega a otro nodo: el usuario puede explorar varios errores sin salir.

```js
{
  tipo: "errores",
  breadcrumbLabel: "texto corto",   // opcional
  errores: [
    {
      codigo: "100",           // número visible en la píldora
      nombre: "Nombre corto",
      que: "Qué significa este error",
      causas: "Posible causa o causas",
      solucion: "Qué debe hacer el usuario (admite HTML)"
    },
    // …más errores
  ]
}
```

---

## Estado de sesión

El motor mantiene dos variables de estado:

- **`historial`**: array de IDs de nodos visitados (pila). `push` al avanzar, `pop` al volver atrás.
- **`contexto`**: objeto `{ [nodoId]: { texto, destino } }` que registra la opción elegida en cada nodo `pregunta`. Se limpia al reiniciar y se actualiza al volver atrás. Se puede leer desde destinos condicionales (`destino: ctx => ...`).

---

## RENDERIZADORES — mapa extensible de tipos de nodo

En vez de un `if/else` encadenado, el motor usa un mapa:

```js
const RENDERIZADORES = {
  pregunta:  renderizarPregunta,
  paso:      renderizarPaso,
  pendiente: renderizarPendiente,
  final:     renderizarFinal
};
```

**Para añadir un tipo de nodo nuevo** (ej. `video`, `formulario`):
1. Define `function renderizarXxx(nodo){ ... }` en el Bloque 2.
2. Añade `xxx: renderizarXxx` en el mapa `RENDERIZADORES`.
3. Define nodos con `tipo: "xxx"` en `ARBOL` (Bloque 1).

---

## Funcionalidades de navegación
- Botón "Volver atrás": deshace el último paso del historial y limpia su entrada del contexto.
- Botón "Reiniciar": vuelve al nodo `inicio` y resetea historial y contexto.
- Botón de soporte fijo en la cabecera (WhatsApp), siempre visible. URL tomada de `CONFIG.whatsapp`.
- Migas de pan dinámicas (solo nodos con `breadcrumbLabel`). Overflow oculto con elipsis en flujos largos.

## Glosario
Términos técnicos (FAT32, clase 10, APN, etc.) usan `class="ftw-glosario"` para mostrar un subrayado discreto con enlace a Wikipedia. No se usan `<img>` con URLs externas.

---

## Flujo actual del árbol

- **Pantalla inicial**: 4 casos de uso (Instalar / Configurar conexión / Envío de fotos / Tengo un problema).
- Cada caso pregunta el **modelo**: Cherokee, Apache, Creek.
- **Cherokee**: completamente desarrollada en los 4 flujos (instalación, conexión, envío, problemas con panel de códigos de error 100–600).
- **Apache**: completamente desarrollada en los 3 flujos (instalación, conexión, envío). La rama "Problemas" va directamente a `soporte_directo`.
- **Creek**: completamente desarrollada en los 3 flujos. Conexión y envío llevan a un final informativo (sin SIM ni app). Instalación tiene 3 pasos + final. La rama "Problemas" va directamente a `soporte_directo`.

---

## Pendiente de completar

No hay pendientes conocidos en este momento.

---

## Especificaciones por modelo

> Esta sección recoge las particularidades de cada cámara que afectan directamente al árbol de decisiones. Lo que no se especifica aquí se comporta igual que en la Cherokee (modelo de referencia).

---

### Cherokee

- Modo de configuración: **SETUP** (nombre del interruptor físico).
- Ranura SIM: lateral del cuerpo interno. Formato **Mini-SIM** (el más grande de los troquelados en el tarjetón portador).
- Ranura SD: parte inferior del cuerpo interno. Formato **Full SD** (SD estándar, el más grande, ~32×24 mm).
- Navegación en menú: `MENU → pestaña CAM → Instalar`.
- Prueba de envío: al pulsar OK en modo SETUP aparece **"Enviando por FTP..."** seguido de **"Enviado con éxito"**. Eso confirma que la conexión funciona.
- Indicador de éxito: mensaje "Enviado con éxito" en pantalla.

---

### Apache

#### Hardware
- Modo de configuración: **TEST** (mismo concepto que SETUP en Cherokee, pero con distinto nombre en el interruptor físico).
- Ranura SIM: lateral del cuerpo interno. Formato **Nano-SIM** (el más pequeño de los troquelados en el tarjetón portador).
- Ranura SD: parte inferior del cuerpo interno. Formato **Micro SD** (~15×11 mm).

#### Menú
- Al pulsar **MENU** en modo TEST, se entra directamente en **Ajustes** (no hay pestañas como CAM). El botón MENU también sirve para volver un paso atrás mientras se navega por los ajustes.
- Ruta de vinculación: `MENU → Instalar`.

#### Indicadores en pantalla (modo TEST)
| Icono | Significado |
|---|---|
| Barras de cobertura 4G | SIM conectada a la red |
| Icono de SIM en rojo | La cámara no detecta ninguna SIM |
| Interrogación (?) sobre las barras | SIM detectada pero sin conexión (inactiva o APN incorrecto) |
| **Nube azul con tick blanco** | Cámara sincronizada con la app Renyn y con cobertura activa → todo correcto |

#### Sin mensajes FTP
La Apache **no muestra** "Enviando por FTP..." ni "Enviado con éxito" al pulsar OK. El único indicador de que la configuración está completa es la **nube azul con tick blanco**.

#### Flujo de instalación completo
1. Abrir carcasa.
2. Insertar nano-SIM en ranura lateral.
3. Insertar micro SD en ranura inferior.
4. Colocar pilas AA respetando polaridad.
5. Cerrar carcasa y montar antena.
6. Descargar app Renyn (App Store / Google Play).
7. Poner interruptor en **TEST** → `MENU → Instalar` → escanear QR desde la app.
8. Comprobar indicadores de cobertura en pantalla.
9. Una vez confirmada la cobertura, **volver a escanear el QR** (siempre necesario tras resolver cualquier incidencia de SIM para que la cámara quede registrada).
10. Verificar que aparece la **nube azul con tick blanco**.

#### Diagnóstico de SIM (icono rojo)
- Retirar y reinsertar la nano-SIM comprobando que el tamaño sea el correcto.
- Si sigue en rojo tras reinserción → configurar APN.

#### Diagnóstico de SIM (interrogación)
1. ¿Está activa la SIM? (tarifa en vigor con datos móviles). Si no → activar o adquirir SIM Renyn.
2. Si está activa → ¿es de Renyn?
   - **No (otra operadora)** → configurar APN manualmente.
   - **Sí** → ¿tipo?
     - **Solo Movistar** → configurar APN.
     - **Multioperadora** → reiniciar cámara + revincular por QR → verificar nube.

#### SIM Renyn — tipos
- **Solo Movistar**: opera únicamente con red Movistar. Requiere configuración manual del APN si da problemas.
- **Multioperadora**: selecciona automáticamente la mejor red disponible. Si hay problemas, reiniciar y revincular por QR suele resolverlo.

---

### Creek

- Sin tarjeta SIM ni antena — funciona únicamente en local.
- Ranura SD: parte inferior del cuerpo interno. Formato **Micro SD** (igual que Apache).
- Las fotos y vídeos se guardan en la SD y se recuperan manualmente (retirar la SD e introducirla en móvil u ordenador).
- No es compatible con la app Renyn Systems.
- No tiene flujo de conexión ni de envío — el nodo de conexión y el de envío llevan a un final informativo.

---

## Cómo se instala en WordPress
1. Crear un snippet nuevo en WPCode, tipo **"HTML Snippet"**.
2. Pegar el contenido completo del archivo.
3. Insertar donde se quiera mediante shortcode o inserción automática.
