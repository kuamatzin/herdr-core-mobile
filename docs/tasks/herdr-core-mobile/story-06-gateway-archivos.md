---
status: not_started
feature: "herdr-core-mobile"
story: "06"
layers:
  - "backend"
dependencies:
  - "story-04"
---

# Story 06: Gateway, archivos

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code que agrega una app móvil para herdr-core. El gateway por servidor (`apps/herdr-gateway`, historia 4) tiene stubs `501` para las rutas de archivos. Esta historia las implementa en `routes/files.ts` y `files.ts`: listar una carpeta, leer un archivo de texto, descargar bytes y subir un archivo sin sobrescribir nunca uno existente, todo limitado al `$HOME` del usuario del servidor. La app (historia 10) navega, previsualiza, descarga y sube sobre estas rutas.

## Dependencies

- Depends on: Story 04 — Gateway base (`apps/herdr-gateway/src/http.ts`: `Router`, `GatewayContext` con `homeDir`, `sendJson`, `sendError`; stubs en `routes/files.ts`; tipos `FileEntry`, `FileListResult`, `FileContentResult`, `FileUploadResult` en `packages/herdr/src/gateway-api.ts`).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Seguridad del gateway?** → **Decision:** solo Tailscale. El gateway escucha únicamente en la IP de Tailscale del servidor (`tailscale ip -4`), nunca en `0.0.0.0`. Además, en cada conexión HTTP y WebSocket consulta `tailscale whois --json <ip>:<puerto>` del cliente y acepta solo si `UserProfile.LoginName` coincide con el dueño del propio nodo. Si no coincide, responde `403`. Ejemplo verificado: la MacBook del usuario resuelve a `kuamatzin@gmail.com`, igual que `mylinux`. Un dispositivo de las socias de Protact sería rechazado aunque vea el puerto.
4. **(plan) Gateway sin Effect.** El gateway y `packages/herdr` usan TypeScript plano con Promises y APIs de Node, no Effect, para no acoplarse a la evolución de las librerías del servidor T3. `packages/herdr` no importa nada de `@t3tools/*`.
5. **(plan) Archivos sin pasar por `hc`.** El gateway corre en el servidor, así que lee y escribe el disco con `node:fs`, limitado a rutas dentro de `$HOME` después de resolver enlaces (`realpath`).

Access Control (binding): los procesos locales del mismo usuario son de confianza (ya tienen acceso de shell). Las carreras TOCTOU con enlaces simbólicos hechas por procesos del mismo usuario quedan fuera del alcance; aun así, los archivos se abren con `O_NOFOLLOW` en el último componente donde Node lo permite.

## Affected Files

### Backend

- [ ] Pattern to follow: la lógica de nombres libres del mod `files-download` del usuario (`stem (n).ext`: si `a.png` existe, se guarda `a (1).png`, luego `a (2).png`; sube a un `.part` oculto y renombra al final sin pisar nunca un nombre tomado). El router y las ayudas de `apps/herdr-gateway/src/http.ts` (historia 4).
- [ ] Conventions to follow: TypeScript plano con `node:fs/promises`, `node:path`, `node:stream` (decisión 14); errores `{ "error": { "kind", "message" } }` con códigos HTTP explícitos (`sendError`); sin `enum`/`namespace`; pruebas `*.test.ts` junto al archivo, sobre una carpeta temporal que hace de `$HOME`.
- [ ] `apps/herdr-gateway/src/files.ts` — nuevo: lógica de archivos, sin HTTP.
- [ ] `apps/herdr-gateway/src/routes/files.ts` — reemplazar los stubs.
- [ ] Prueba: `apps/herdr-gateway/src/files.test.ts`.

## Implementation Steps

### Step 1: contrato

Del API Contract (binding):

**`GET /v1/files?path=/abs/dir`** → `{ "path": "/abs/dir", "entries": [{ "name": "src", "dir": true, "size": null, "mtimeMs": 0 }] }` (máximo 500 entradas; `"truncated": true` si hay más).
**`GET /v1/files/content?path=`** → `{ "text": "…", "truncated": false }` (máximo 1 MiB; `415 {"error":{"kind":"binary"}}` si tiene bytes nulos).
**`GET /v1/files/download?path=`** → bytes con `content-disposition: attachment`.
**`POST /v1/files/upload?dir=/abs&name=a.png`** (cuerpo: bytes) → `{ "path": "/abs/a (1).png" }`.

Errores (Validation & Error Handling, binding): rutas fuera de `$HOME` → `403 {"error":{"kind":"outside_home"}}`. Inexistente → `404 {"error":{"kind":"not_found"}}`. Archivo binario en `/content` → `415`. Más de 1 MiB → se trunca con `truncated: true`. Subida interrumpida: no queda el `.part`.

### Step 2: `files.ts`

- `export async function resolveInsideHome(homeDir: string, path: string): Promise<string>`: `path` debe ser absoluto (si no → `FilesError("bad_request")`); `realHome = await realpath(homeDir)`; `real = await realpath(path)` (si `ENOENT` → `FilesError("not_found")`); aceptar solo si `real === realHome` o `real.startsWith(realHome + "/")`; si no → `FilesError("outside_home")`. Esto cubre `..` y enlaces simbólicos que salen de `$HOME`.
- `export class FilesError extends Error { constructor(readonly kind: "outside_home" | "not_found" | "binary" | "bad_request" | "not_a_directory" | "is_directory") }` (sin `enum`).
- `listDirectory(homeDir, path): Promise<FileListResult>`: carpeta resuelta; `readdir(..., { withFileTypes: true })`; por cada entrada, `stat` (sigue enlaces; si falla —enlace roto— `dir: false, size: null, mtimeMs: 0`). Orden: carpetas primero, luego por nombre (`localeCompare`). Máximo 500 entradas; `truncated: true` si había más. `path` devuelto = ruta real. `size: null` en carpetas.
- `openInsideHome(homeDir, real, flags)`: abre la ruta ya resuelta con `fs.promises.open(real, flags | constants.O_NOFOLLOW)` y **revalida el descriptor**: `realpath("/proc/self/fd/" + fd)` debe seguir dentro de `$HOME` (si no → cerrar y `outside_home`). Toda lectura usa este descriptor, no la ruta.
- `readTextFile(homeDir, path): Promise<FileContentResult>`: archivo resuelto (carpeta → `is_directory`, `400`); leer como máximo 1 MiB del descriptor; si hay un byte `0x00` → `FilesError("binary")`; decodificar con `StringDecoder("utf8")` y `write()` sin `end()`: así se devuelven todos los caracteres completos y solo se omiten los bytes de un carácter incompleto cortado por el límite (nunca `\uFFFD` por el corte); `truncated = size > 1 MiB`.
- `openDownload(homeDir, path): Promise<{ stream: ReadStream; size: number; name: string }>`: stream creado desde el descriptor de `openInsideHome` (`fd.createReadStream()`), tamaño de `fd.stat()`.
- `freeName(dir: string, name: string, exists: (p: string) => Promise<boolean>): Promise<string>`: si `name` está libre, `name`; si no, `stem (n).ext` con `n = 1, 2, …`, donde `ext = extname(name)` (solo la última extensión: `archive.tar.gz` → `archive.tar (1).gz`; sin extensión: `Makefile (1)`; archivo oculto sin extensión: `.env (1)`).
- `uploadFile(homeDir, dir, name, body: Readable): Promise<{ path: string }>`: `dir` resuelto dentro de `$HOME` (y debe ser carpeta). `name` válido: no vacío, máximo 255 bytes en UTF-8, sin `/`, `\\`, `\0` ni caracteres de control (`< 0x20`), distinto de `.` y `..`; se usa tal cual llega (sin normalizar Unicode) (si no → `bad_request`). Si `final` existe y es una carpeta, cuenta como ocupado igual que un archivo. Escribir a `join(dir, "." + name + ".part-" + randomUUID())` con `pipeline(body, createWriteStream(part, { flags: "wx" }))`. Al terminar: `final = await freeName(...)` y `await link(part, final)` (el `.part` está en la misma carpeta que `final`, así que `link` nunca cruza sistemas de archivos; falla con `EEXIST` si alguien lo tomó entre medio → volver a calcular con el siguiente número), luego `unlink(part)`. Si la subida se corta o falla en cualquier punto → `unlink(part)` (ignorar `ENOENT`) y relanzar. Nunca sobrescribe.

### Step 3: `routes/files.ts`

Mantener `export function registerFileRoutes(router: Router, ctx: GatewayContext): void`, con `ctx.homeDir`.

- `GET /v1/files?path=` → `listDirectory`. Sin `path` → listar `ctx.homeDir`.
- `GET /v1/files/content?path=` → `readTextFile`.
- `GET /v1/files/download?path=` → `200` con `content-type: application/octet-stream`, `content-length`, `content-disposition: attachment; filename*=UTF-8''<encodeURIComponent(name)>` y el stream del archivo.
- `POST /v1/files/upload?dir=&name=` → `uploadFile(ctx.homeDir, dir, name, req.raw)`; si el cliente corta (`req.raw` emite `aborted`/`error`), el `.part` se borra y no se responde nada más.
- Mapeo de `FilesError.kind` a HTTP: `outside_home` → `403`, `not_found` → `404`, `binary` → `415`, `bad_request`/`not_a_directory`/`is_directory` → `400`.

### Step 4: pruebas (`files.test.ts`)

Sobre una carpeta temporal creada por la prueba como `homeDir`:

- Salida de `$HOME` por `..` (`<home>/../x`) → `outside_home`; por enlace simbólico dentro de home que apunta afuera → `outside_home`; enlace que apunta adentro → aceptado.
- Inexistente → `not_found`.
- Nombres inválidos (`a/b`, `a\\b`, `..`, 256 bytes, con `\n`) → `bad_request`.
- `readTextFile` con un carácter de varios bytes cortado justo en 1 MiB: el texto termina en el último carácter completo.
- Conflicto de nombre: con `a.png` existente, subir `a.png` → `a (1).png`, otra vez → `a (2).png`; el original no cambia (mismo contenido).
- Subida cortada (un `Readable` que emite error a la mitad) → rechaza y no queda ningún archivo `.part` en la carpeta.
- `readTextFile`: archivo con `\0` → `binary`; archivo de 1 MiB + 10 bytes → `truncated: true`.
- `listDirectory` con 501 archivos → 500 entradas y `truncated: true`; carpetas antes que archivos.

## Out of Scope

- Carreras TOCTOU hechas por procesos locales del mismo usuario: fuera del alcance (modelo de confianza); se mitigan con `O_NOFOLLOW` y la revalidación del descriptor.
- Borrar archivos desde la app (fuera del alcance del plan): no hay ruta de borrado, ni renombrar, ni mover.
- Leer archivos a través de `hc` (`file.read`): decisión 16, se usa `node:fs`.
- La UI de archivos y el inspector de iPad: historia 10.
- Terminal (historia 5), chat (historia 13).
- Límites de tamaño de subida: no hay en la v1.

## Acceptance Criteria

- [ ] Dentro de `apps/herdr-gateway`: `vp test run src/files.test.ts` pasa con todos los casos del Step 4.
- [ ] Dentro de `apps/herdr-gateway`: `vp run typecheck` pasa.
- [ ] Las rutas de `routes/files.ts` ya no responden `501`.
- [ ] `grep -n "writeFile\|rename(" apps/herdr-gateway/src/files.ts` muestra que la escritura final usa `link` sobre el `.part` (nunca `rename` sobre un nombre que pueda existir).
- [ ] Ningún otro archivo del gateway cambia salvo `files.ts`, `routes/files.ts` y su prueba.
- [ ] `node scripts/herdr/check-patch-surface.ts` sale con código 0.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

S
