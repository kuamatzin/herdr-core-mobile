---
status: not_started
feature: "herdr-core-mobile"
story: "10"
layers:
  - "frontend"
dependencies:
  - "story-06"
  - "story-07"
---

# Story 10: App, archivos

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code cuya app móvil abre en Herdr y se conecta a un gateway por servidor. El gateway ya expone archivos limitados al `$HOME` del servidor (historia 6). Esta historia hace la pantalla de archivos de un espacio: navegar desde el cwd del espacio, previsualizar texto y código con el visor de T3 e imágenes con el visor de imágenes de T3, descargar al teléfono (hoja de compartir del sistema) y subir desde el teléfono (selector de documentos). En iPad, el mismo navegador aparece en el inspector junto al agente.

## Dependencies

- Depends on: Story 06 — Gateway, archivos (`GET /v1/files`, `/v1/files/content`, `/v1/files/download`, `POST /v1/files/upload`; tipos `FileEntry`, `FileListResult`, `FileContentResult`, `FileUploadResult`, `GatewayError` en `@herdr/core-client/gateway-api`).
- Depends on: Story 07 — App, conexión e inicio (`createGatewayClient(baseUrl)` con `getJson`/`postJson`/`GatewayHttpError`; `useHerdrHosts()`; ruta `HerdrFiles` `{ hostId, path? }` con la pantalla temporal `HerdrFilesRoute.tsx`; componente temporal `HerdrFilesInspector` `{ hostId, path }`, que la historia 9 monta en `HerdrAgentRoute`).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Alcance de la v1?** → **Decision:** espacios, pestañas, terminal en vivo, agentes por prioridad, archivos (navegar, ver, descargar, subir) y avisos con la app abierta. Las notificaciones push van al final y bloqueadas (decisión 13). El chat va justo después de la base (decisión 8).
4. **(plan) Archivos sin pasar por `hc`.** El gateway corre en el servidor, así que lee y escribe el disco con `node:fs`, limitado a rutas dentro de `$HOME` después de resolver enlaces (`realpath`).

## Affected Files

### Frontend

- [ ] Pattern to follow: `apps/mobile/src/features/files/FileTreeBrowser.tsx` (aspecto de la lista de archivos; recibe `ProjectEntry[]` de T3, así que se usa como referencia visual, no directamente), `apps/mobile/src/features/files/SourceFileSurface.tsx` (`SourceFileSurface({ contents, path, embedded?, selectable?, onRefresh? })`: visor de texto y código con resaltado, se reutiliza tal cual), `apps/mobile/src/features/files/WorkspaceFileImagePreview.tsx` (`WorkspaceFileImagePreview({ accessibilityLabel, uri })`: visor de imágenes, se reutiliza con un `file://` local). Inspector de iPad: `Inspector({ content: { panes: [{ key, render }], active }, revealKey? })` exportado por `apps/mobile/src/features/layout/AdaptiveWorkspaceLayout.tsx` (tipo `InspectorContent` en `features/layout/inspector-panes.ts`).
- [ ] Conventions to follow: estado con átomos de `@effect/atom-react` o hooks de React con estado local donde baste; `PascalCase.tsx` para componentes, `camelCase.ts` para lógica; estilos con clases Uniwind y `useThemeColor`; símbolos con `AppSymbol`; texto con `AppText`; `ListRow`, `EmptyState`, `ErrorBanner`, `Button` de `components/ui`; pruebas `*.test.ts` junto al archivo. Dependencias ya presentes en `apps/mobile/package.json`: `expo-file-system`, `expo-sharing`, `expo-document-picker`.
- [ ] `apps/mobile/src/features/herdr/HerdrFilesRoute.tsx` — reemplazar la pantalla temporal.
- [ ] `apps/mobile/src/features/herdr/state/useHerdrFiles.ts` — nuevo.
- [ ] `apps/mobile/src/features/herdr/HerdrFilePreview.tsx` — nuevo.
- [ ] `apps/mobile/src/features/herdr/HerdrFilesInspector.tsx` — reemplazar el componente temporal.
- [ ] Prueba: `apps/mobile/src/features/herdr/state/useHerdrFiles.test.ts`.

## Implementation Steps

### Step 1: contrato

Del API Contract (binding):

**`GET /v1/files?path=/abs/dir`** → `{ "path": "/abs/dir", "entries": [{ "name": "src", "dir": true, "size": null, "mtimeMs": 0 }] }` (máximo 500 entradas; `"truncated": true` si hay más).
**`GET /v1/files/content?path=`** → `{ "text": "…", "truncated": false }` (máximo 1 MiB; `415 {"error":{"kind":"binary"}}` si tiene bytes nulos).
**`GET /v1/files/download?path=`** → bytes con `content-disposition: attachment`.
**`POST /v1/files/upload?dir=/abs&name=a.png`** (cuerpo: bytes) → `{ "path": "/abs/a (1).png" }`.

Errores: `403 outside_home`, `404 not_found`, `415 binary`. Subida interrumpida: el gateway no deja `.part`; la app muestra el error.

### Step 2: `state/useHerdrFiles.ts`

- Funciones puras exportadas: `parentPath(path)` (`/a/b` → `/a`; `/` → `null`), `joinPath(dir, name)`, `previewKind(name): "image" | "text" | "other"` (imagen por extensión: `png`, `jpg`, `jpeg`, `gif`, `webp`, `heic`, `bmp` → `image`; `svg` es **texto**, no imagen —no se pasa SVG remoto al visor de imágenes—; el resto se intenta como texto y el gateway responde `415` si es binario).
- `useHerdrDirectory(hostId, path)` → `{ listing: FileListResult | null; error: GatewayErrorKind | null; loading; refresh() }` con `getJson("/v1/files?path=" + encodeURIComponent(path))`.
- `useHerdrFileText(hostId, path)` → `{ content: FileContentResult | null; error; loading }`.
- `downloadToCache(hostId, path): Promise<string>`: descarga `GET /v1/files/download?path=` con `expo-file-system` a la carpeta de caché de la app (nombre = basename) y devuelve el `file://` local. Se usa para imágenes y para compartir.
- `shareFile(hostId, path)`: `downloadToCache` + `Sharing.shareAsync(uri)` (`expo-sharing`).
- `uploadFromDevice(hostId, dir): Promise<string | null>`: `DocumentPicker.getDocumentAsync({ copyToCacheDirectory: true })`; si se canceló → `null`; si no, sube el archivo elegido con `uploadAsync` de `expo-file-system/legacy` (`uploadAsync(url, asset.uri, { httpMethod: "POST", uploadType: FileSystemUploadType.BINARY_CONTENT, headers: { "content-type": "application/octet-stream" } })`: el módulo nativo lee el archivo en stream, sin cargarlo en memoria de JS) a `/v1/files/upload?dir=<dir>&name=<nombre>` (ambos con `encodeURIComponent`); respuesta con `status` distinto de 200 → lanzar con el `error.kind` del cuerpo y devuelve `path` de la respuesta (puede ser `nombre (1).ext`: el gateway nunca sobrescribe). Error → lanza con el `kind` del gateway.
- `useHerdrFiles.test.ts`: `parentPath`, `joinPath` y `previewKind` (incluido `logo.svg` → `text`).

### Step 3: `HerdrFilesRoute.tsx`

Mantener `export function HerdrFilesRoute(props: StaticScreenProps<{ hostId: string; path?: string }>)`.

- Carpeta inicial = `path` de la ruta (el cwd del espacio, que pasan las historias 8 y 9); si falta, `$HOME` (llamar `/v1/files` sin `path`). La carpeta actual siempre es el `path` que devuelve el gateway (ruta real, ya resuelta), y ".." usa `parentPath` de esa ruta real; si el padre queda fuera de `$HOME`, el gateway responde `outside_home` y la fila ".." se oculta en esa carpeta.
- Lista: carpeta "..", carpetas primero y luego archivos (el gateway ya ordena), tamaño y fecha; si `truncated`, una fila final "Hay más archivos (se muestran 500)". Tocar carpeta → entrar (estado local de navegación, con "atrás" a la carpeta padre); tocar archivo → `HerdrFilePreview`.
- Acción del encabezado "Subir" → `uploadFromDevice(hostId, carpetaActual)`, luego refrescar y mostrar `Toast`/mensaje con el nombre con el que quedó; error → `ErrorBanner` con el mensaje.
- Errores: `outside_home` → "Fuera de la carpeta personal"; `not_found` → "No existe"; red → "Sin conexión" con botón para reintentar.
- Extraer el contenido (lista + vista previa) a un componente `HerdrFilesBrowser({ hostId, initialPath, embedded })` en este mismo archivo, para reutilizarlo en el inspector.

### Step 4: `HerdrFilePreview.tsx`

`HerdrFilePreview({ hostId, path, embedded? })`:

- `image` → `downloadToCache` y `WorkspaceFileImagePreview({ accessibilityLabel: name, uri })` (con `uri: null` mientras descarga).
- `text` → `useHerdrFileText` y `SourceFileSurface({ contents: text, path, embedded })`; si `truncated`, aviso "Archivo truncado a 1 MiB". Si el gateway responde `415 binary` → "Vista previa no disponible" con el botón "Descargar".
- Acción "Descargar" siempre visible → `shareFile` (hoja de compartir del sistema).

### Step 5: `HerdrFilesInspector.tsx` (iPad)

Mantener `export function HerdrFilesInspector(props: { hostId: string; path: string | null }): ReactNode`. Si `path` no es nulo, renderizar `<Inspector content={{ panes: [{ key: "herdr-files", render: () => <HerdrFilesBrowser hostId={hostId} initialPath={path} embedded /> }], active: "herdr-files" }} />`. `Inspector` registra el panel solo mientras la pantalla que lo monta (`HerdrAgentRoute`, historia 9) tiene el foco y en teléfono no dibuja nada (el inspector es una hoja que solo se abre a pedido), así que en teléfono no cambia nada. Memoizar `content` para no re-registrar en cada render.

## Out of Scope

- Borrar, renombrar o mover archivos: fuera del alcance.
- Editar archivos de texto desde la app: no está en la v1.
- Rutas fuera de `$HOME`: el gateway las rechaza (historia 6).
- Cambiar `HerdrAgentRoute.tsx` (historia 9) o el árbol de espacios (historia 8): ya navegan a `HerdrFiles` y montan `HerdrFilesInspector`.
- Editar archivos de T3 (`features/files/*`, layout): se reutilizan sin cambios.

## Acceptance Criteria

- [ ] Dentro de `apps/mobile`: `vp test run src/features/herdr/state/useHerdrFiles.test.ts` pasa.
- [ ] Dentro de `apps/mobile`: `vp run typecheck` pasa.
- [ ] Ningún archivo fuera de `apps/mobile/src/features/herdr/` cambia; `node scripts/herdr/check-patch-surface.ts` sale con código 0.
- [ ] Con un gateway real (Android en `mylinux`): abrir Archivos de un espacio muestra su cwd; un `.ts` se ve con resaltado; un `.png` se ve como imagen; "Descargar" abre la hoja de compartir; subir una imagen con el nombre de un archivo existente deja `nombre (1).ext` y el original no cambia. Anotar el resultado en el tracker (iPad: verificación manual en la Mac del usuario).

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
