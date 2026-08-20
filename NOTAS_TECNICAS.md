# Notas técnicas — Monitor DCA

## Arquitectura

Esta app no tiene backend propio. Se apoya en dos piezas separadas:

- **`dca`** (este repo, público o privado según se configure en GitHub Pages):
  contiene solo código. `index.html` es la app entera (HTML + CSS + JS
  autocontenidos, sin build step) y se despliega tal cual en GitHub Pages.
- **`dca-data`** (repo aparte, **privado**): contiene únicamente
  `data.json` con los datos reales del usuario (activos, niveles,
  operativas, historial de ventas, muestras de precio). La app lee y
  escribe ese archivo directamente desde el navegador contra la API REST
  de GitHub.

**Por qué separar código y datos en dos repos:** el código de `dca` puede
ser público o compartirse sin problema; los datos (cartera, operativas)
son información personal y no deben mezclarse nunca con el repo de
código, aunque ese repo también fuera privado. Un token con acceso solo
al repo de datos limita el daño si se filtra: nunca da acceso al código.

## Persistencia sin servidor propio

No hay backend: la propia app hace `fetch` contra
`https://api.github.com/repos/{owner}/{repo}/contents/data.json` desde el
navegador del usuario.

- **Lectura (`GET`)**: descarga `data.json`, decodifica el `content`
  (base64, con soporte UTF-8 vía `TextEncoder`/`TextDecoder` para que
  tildes y ñ no se rompan) y puebla `assets`, `positions`, `trades` y
  `priceHistory`.
- **Escritura (`PUT`)**: sube el JSON codificado en base64 junto con el
  `sha` del archivo anterior (obligatorio en la API de GitHub para
  actualizar un archivo existente sin pisar una escritura concurrente).
  Si el `sha` está desactualizado, la API responde `409`; la app relee el
  `sha` actual y reintenta una sola vez.
- Un `data.json` inexistente (404 en el primer uso) se trata como "no hay
  nada que perder": se permite guardar y el archivo se crea en el primer
  guardado real.

## Autenticación: token en localStorage, nunca en el código

El menú "Sincronización" (icono ☁/⚠/✓ en la cabecera) pide:

- Usuario/organización y nombre del repo de datos.
- Un **token fine-grained de GitHub** con permiso limitado a
  **"Contents: Read and write"** y acceso restringido solo al repo de
  datos (nunca a todos los repos de la cuenta).

El token se guarda exclusivamente en `localStorage` del navegador
(`monitor_dca_gh_token`), igual que el resto de configuración de
sincronización (`monitor_dca_gh_owner`, `monitor_dca_gh_repo`). Nunca se
hardcodea en el código ni se sube a ningún repo — de hecho `data.json`
está en `.gitignore` de este repo por si alguna vez se genera una copia
local.

Antes de poder guardar la configuración hay que pulsar **"Probar
conexión"**: hace un `GET` a `/repos/{owner}/{repo}` con el token y solo
si responde bien se habilita el botón de guardar (`syncTestOk` en el
estado del componente). Esto evita persistir una configuración rota sin
que el usuario se entere.

## Guardado automático: solo ante cambios reales, con debounce

Requisito clave: la sincronización con GitHub **nunca** debe dispararse
por abrir la app, cambiar de pestaña/vista o pulsar una tecla en un
formulario a medio rellenar. Solo debe guardar cuando el usuario confirma
un cambio real: añadir/editar/borrar un activo, guardar una operativa,
registrar una venta o importar una copia de seguridad.

Por eso `scheduleGithubSave()` se llama únicamente desde los puntos donde
ya se persiste en `localStorage` tras una acción confirmada por el
usuario (`doSave`, `doSell`, `onDelete`, `importData`) — nunca desde
`refresh()` (poll de precios cada 2 min), ni desde cambios de pestaña,
tema, búsqueda o teclas sueltas en un formulario todavía sin guardar.

`scheduleGithubSave()` aplica un debounce de 1.5 s (`setTimeout` +
`clearTimeout` sobre el mismo temporizador) antes de llamar a
`saveToGithub()`, para no lanzar una petición por cada cambio si se
encadenan varias acciones seguidas.

## Por qué la carga inicial bloquea el guardado hasta que tiene éxito

Si la sincronización está configurada pero la carga inicial desde GitHub
falla (token caducado, sin red, repo inaccesible…), la app **no debe
guardar nada** hasta conseguir cargar bien. Si guardara de todas formas,
un estado local vacío o desactualizado podría sobrescribir los datos
reales del usuario en `dca-data`.

Esto se implementa con la bandera `ghLoadOk` en el estado del
componente:

- Arranca en `false` en cuanto hay configuración de sincronización.
- `loadFromGithub()` la pone a `true` solo si el `GET` tiene éxito (o si
  el archivo no existe todavía, caso en el que no hay nada que
  sobrescribir).
- `scheduleGithubSave()` y `saveToGithub()` comprueban `ghLoadOk` al
  principio y no hacen nada si es `false`.
- Mientras `ghLoadOk` es `false` con la sincronización configurada, la
  cabecera muestra un aviso ("Sincronización con GitHub pausada") con un
  botón "Reintentar" que vuelve a llamar a `loadFromGithub()`.

Los datos ya guardados en `localStorage` de una sesión anterior se siguen
mostrando mientras tanto (no se borran), así que el usuario nunca ve la
app vacía por un fallo de red — solo deja de sincronizar hasta que la
conexión se recupera.

## Despliegue

`index.html` no necesita build ni backend: sirve tal cual desde GitHub
Pages (rama del repo `dca` configurada como origen de Pages). Todo el
runtime de la app (fuentes, lógica) va embebido en ese único archivo.
