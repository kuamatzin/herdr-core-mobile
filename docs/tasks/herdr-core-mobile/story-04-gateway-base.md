---
status: not_started
feature: "herdr-core-mobile"
story: "04"
layers:
  - "backend"
dependencies:
  - "story-03"
---

# Story 04: Gateway base

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code que agrega una app móvil para herdr-core (`hc`). Cada servidor del usuario (`mylinux`, `gx10`) corre un **gateway** que habla con `hc connect` localmente y expone un API HTTP/WebSocket solo en la red de Tailscale, solo para el dueño. Esta historia crea el gateway: proceso `hc connect` supervisado, autorización por Tailscale, `GET /v1/health`, `GET /v1/snapshot`, `WS /v1/events` con el modelo de máquinas → carpetas → espacios → pestañas → terminales, el registro de rutas con stubs para las historias 5, 6 y 13, los tipos del API en `packages/herdr/src/gateway-api.ts` y el instalador systemd.

## Dependencies

- Depends on: Story 03 — Cliente de `hc` (`@herdr/core-client/hc`: `HcConnection`, `ByteTransport`, tipos `Terminal`, `Status`, `HelloResult`, `SubscribeResult`).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Un gateway central o uno por servidor?** → **Decision:** uno por servidor. Cada servidor corre su gateway, su `herdr-tap` y sus hooks. La app guarda una lista de gateways y se conecta a cada uno. Ejemplo: si `gx10` se apaga, `mylinux` sigue visible y `gx10` aparece como desconectado.
4. **Q: ¿Seguridad del gateway?** → **Decision:** solo Tailscale. El gateway escucha únicamente en la IP de Tailscale del servidor (`tailscale ip -4`), nunca en `0.0.0.0`. Además, en cada conexión HTTP y WebSocket consulta `tailscale whois --json <ip>:<puerto>` del cliente y acepta solo si `UserProfile.LoginName` coincide con el dueño del propio nodo. Si no coincide, responde `403`. Ejemplo verificado: la MacBook del usuario resuelve a `kuamatzin@gmail.com`, igual que `mylinux`. Un dispositivo de las socias de Protact sería rechazado aunque vea el puerto.
5. **(plan) Gateway sin Effect.** El gateway y `packages/herdr` usan TypeScript plano con Promises y APIs de Node, no Effect, para no acoplarse a la evolución de las librerías del servidor T3. `packages/herdr` no importa nada de `@t3tools/*`.

## Affected Files

### Backend

- [ ] Pattern to follow: ninguno de T3 (decisión 14). Usar `node:http` y la librería `ws` (en el catálogo: `ws: catalog:`, `@types/ws: catalog:`). Para el empaquetado, la sección `pack` de `apps/server/vite.config.ts` (`vp pack`, tsdown). Para el protocolo de `hc`: `@herdr/core-client/hc` (historia 3) y `https://github.com/herdrdev/herdr-core/blob/main/schema/README.md`.
- [ ] Conventions to follow: TypeScript plano, solo tipos que se borran (sin `enum` ni `namespace`), errores como `{ "error": { "kind", "message" } }` con códigos HTTP explícitos; sin imports de `@t3tools/*`; pruebas `*.test.ts` junto al archivo con `vite-plus/test`; nada de `Date.now()`/sleeps en pruebas (inyectar reloj y temporizadores).
- [ ] `apps/herdr-gateway/package.json`, `apps/herdr-gateway/tsconfig.json`, `apps/herdr-gateway/vite.config.ts` — nuevos (configuración).
- [ ] `apps/herdr-gateway/src/main.ts` — nuevo: arranque, supervisor de `hc connect`, servidor.
- [ ] `apps/herdr-gateway/src/http.ts` — nuevo: router mínimo, contexto y ayudas de respuesta (registro común que usan las historias 5, 6, 12 y 13).
- [ ] `apps/herdr-gateway/src/auth.ts` — nuevo: decisión 11.
- [ ] `apps/herdr-gateway/src/snapshot.ts` — nuevo: modelo y eventos.
- [ ] `apps/herdr-gateway/src/routes/terminals.ts`, `routes/files.ts`, `routes/chat.ts`, `routes/devices.ts` — nuevos, stubs `501`.
- [ ] `apps/herdr-gateway/src/push/notifier.ts` — nuevo, stub `startPushNotifier(ctx): { close(): void }` que no hace nada (la historia 17 lo reemplaza sin tocar `main.ts`).
- [ ] `apps/herdr-gateway/scripts/install.sh` — nuevo.
- [ ] `packages/herdr/src/gateway-api.ts` — tipos del API Contract (sin chat).
- [ ] Pruebas: `apps/herdr-gateway/src/auth.test.ts`, `apps/herdr-gateway/src/snapshot.test.ts`, `apps/herdr-gateway/src/http.test.ts`.

## Implementation Steps

### Step 1: paquete

- `apps/herdr-gateway/package.json`: `"name": "@herdr/gateway"`, `"private": true`, `"type": "module"`, `"version": "0.1.0"`; scripts `"typecheck": "effect-tsgo tsrs --noEmit"`, `"test": "vp test run"`, `"build": "vp pack"`; dependencies `"@herdr/core-client": "workspace:*"`, `"ws": "catalog:"`; devDependencies `"@types/node": "catalog:"`, `"@types/ws": "catalog:"`, `"vite-plus": "catalog:"`. `vp i` en la raíz (cambia `pnpm-lock.yaml`, permitido).
- `tsconfig.json` como `packages/tailscale/tsconfig.json` (`extends ../../tsconfig.base.json`, `types: ["node"]`, `include: ["src"]`).
- `vite.config.ts`: `mergeConfig(baseConfig, defineConfig({ pack: { entry: ["src/main.ts"], outDir: "dist", platform: "node", target: "node22", clean: true, ... } }))` importando `../../vite.config.ts` como hace `apps/server/vite.config.ts`. Configurar el bundle para incluir **todas** las dependencias (`ws`, `@herdr/core-client`) en un solo archivo JS (opción `noExternal` de tsdown); verificar que el archivo resultante corre con `node` fuera del repo, sin `node_modules`.

### Step 2: tipos del contrato (`packages/herdr/src/gateway-api.ts`)

Solo `interface`/`type`. Contrato (binding):

Base: `http://<ip-tailscale-del-servidor>:7788`. Todas las rutas exigen la autorización de la decisión 11 (`403 {"error":{"kind":"forbidden"}}` si falla). JSON en UTF-8.

**`GET /v1/health`** → `{ "gatewayVersion": "0.1.0", "hc": { "version": "0.1.18", "generation": 1 }, "machine": { "coreId": "2918…", "name": "mylinux" } }`

**`GET /v1/snapshot`** →

```json
{
  "seq": 42,
  "machine": { "coreId": "2918…", "name": "mylinux" },
  "folders": [
    { "id": "f1", "name": "Protact", "color": "ansi4", "folded": false, "spaceIds": ["sp…"] }
  ],
  "spaces": [
    {
      "id": "sp…",
      "name": "odoo",
      "cwd": "/home/u/…/odoo",
      "folderId": "f1",
      "tabs": [{ "id": "tb…", "name": "1", "panes": [{ "paneId": "p…", "terminal": "t1" }] }]
    }
  ],
  "terminals": [
    {
      "address": "t1",
      "spaceId": "sp…",
      "tabId": "tb…",
      "cwd": "/home/u/…",
      "title": null,
      "turn": 18,
      "app": "claude-code",
      "status": { "state": "blocked", "kind": "permission", "msg": "Bash(npm test)" },
      "changedAtMs": 1793900000000
    }
  ]
}
```

`status` es `null` en terminales que no reportan estado. Los ids son los de herdr-core, sin prefijo de máquina.

**`WS /v1/events`** → primer mensaje `{ "type": "snapshot", ...igual que /v1/snapshot }`; luego `{ "type": "terminal", "terminal": {…} }`, `{ "type": "terminalRemoved", "address": "t3" }`, `{ "type": "layout", "spaces": [...], "folders": [...] }` y `{ "type": "heartbeat", "seq": 50 }` cada 15 s.

**`WS /v1/terminals/:address/stream?cols=80&rows=24&history=2000`**: servidor → `{ "type": "output", "data": "…" }`, `{ "type": "exit", "reason": "exited", "exitCode": 0 }`; cliente → `{ "type": "input", "data": "…" }`, `{ "type": "resize", "cols": 60, "rows": 30 }`.

**`POST /v1/terminals/:address/send`** `{ "text": "…" }` → `{ "turn": 19, "started": true }`. **`/keys`** `{ "keys": ["ctrl+c"] }`. **`/interrupt`** `{}`.

**`POST /v1/tabs`** `{ "spaceId": "sp…", "argv": ["claude"] }` (`argv` opcional; sin él, shell) → `{ "address": "t9", "tabId": "tb…" }`.

**`GET /v1/files?path=/abs/dir`** → `{ "path": "/abs/dir", "entries": [{ "name": "src", "dir": true, "size": null, "mtimeMs": 0 }] }` (máximo 500 entradas; `"truncated": true` si hay más).
**`GET /v1/files/content?path=`** → `{ "text": "…", "truncated": false }` (máximo 1 MiB; `415 {"error":{"kind":"binary"}}` si tiene bytes nulos).
**`GET /v1/files/download?path=`** → bytes con `content-disposition: attachment`.
**`POST /v1/files/upload?dir=/abs&name=a.png`** (cuerpo: bytes) → `{ "path": "/abs/a (1).png" }`.

**`POST /v1/devices`** (historia 17) `{ "platform": "ios" | "android", "token": "…", "environment": "sandbox" | "production" }` → `204`.
**`DELETE /v1/devices/:token`** (historia 17; `token` codificado en la URL, máximo 4096 caracteres) → `204`; `404 {"error":{"kind":"not_found"}}` si no está registrado.

**Compatibilidad (fail-closed):** si `hc` no habla la generación 1, `GET /v1/health` responde `503 {"error":{"kind":"core_incompatible"}}`. Si el documento `herdr-app` no tiene la forma esperada, `GET /v1/snapshot` responde `503 {"error":{"kind":"layout_unsupported"}}`.

Exportar: `GatewayHealth`, `GatewayMachine`, `GatewaySnapshot`, `GatewayFolder`, `GatewaySpace`, `GatewayTab`, `GatewayPane`, `GatewayTerminal` (con `spaceId: string | null`, `tabId: string | null` para terminales que no están en ninguna pestaña), `AgentState = "blocked" | "done" | "working" | "idle" | (string & {})`, `GatewayTerminalStatus { state; kind: string | null; msg: string | null }`, `GatewayEventMessage` (unión de los cuatro mensajes de `/v1/events`), `TerminalStreamServerMessage`, `TerminalStreamClientMessage`, `TerminalSendRequest`, `TerminalSendResult`, `TerminalKeysRequest`, `NewTabRequest`, `NewTabResult`, `FileEntry`, `FileListResult`, `FileContentResult`, `FileUploadResult`, `DeviceRegistration { platform: "ios" | "android"; token: string; environment: "sandbox" | "production" }` (lo usan las historias 17 y 18; aquí se define para que la 17 no edite este archivo), `GatewayErrorKind = "forbidden" | "not_found" | "core_unavailable" | "core_incompatible" | "layout_unsupported" | "not_implemented" | "outside_home" | "binary" | "bad_request" | "not_pending" | (string & {})`, `GatewayError { error: { kind: GatewayErrorKind; message?: string } }`. Los tipos de chat los agrega la historia 13.

### Step 3: `http.ts` (registro común)

```ts
export interface GatewayRequest {
  params: Record<string, string>;
  query: URLSearchParams;
  raw: IncomingMessage;
  json<T>(): Promise<T>;
}
export type HttpHandler = (req: GatewayRequest, res: ServerResponse) => Promise<void> | void;
export type WsHandler = (socket: WebSocket, req: GatewayRequest) => Promise<void> | void;
export interface Router {
  get(path: string, h: HttpHandler): void;
  post(path: string, h: HttpHandler): void;
  delete(path: string, h: HttpHandler): void;
  ws(path: string, h: WsHandler): void;
}
export class BadRequestError extends Error {}
export interface GatewayContext {
  hc(): HcConnection | null;
  snapshot: SnapshotStore;
  homeDir: string;
}
export function sendJson(res: ServerResponse, status: number, body: unknown): void;
export function sendError(
  res: ServerResponse,
  status: number,
  kind: GatewayErrorKind,
  message?: string,
): void;
export function createRouter(): Router & { matchHttp(method, pathname); matchWs(pathname) };
export function createGatewayServer(opts: {
  host: string;
  port: number;
  router;
  authorize(remoteAddress: string, remotePort: number): Promise<boolean>;
}): http.Server;
```

- Patrones con segmentos `:param` (`/v1/terminals/:address/stream`), decodificando con `decodeURIComponent`.
- `createGatewayServer`: en cada petición HTTP y en cada `upgrade` llama `authorize(socket.remoteAddress, socket.remotePort)` **antes** de cualquier otra cosa; si es falso → `403 {"error":{"kind":"forbidden"}}` (en `upgrade`, escribir la respuesta HTTP 403 al socket y destruirlo). Ruta inexistente → `404 {"error":{"kind":"not_found"}}`. Excepción no controlada en un handler → `500 {"error":{"kind":"internal"}}` sin filtrar detalles. WebSocket con `new WebSocketServer({ noServer: true })` y `handleUpgrade` solo si la ruta `ws` existe.
- `json<T>()` limita el cuerpo a 1 MiB y **lanza** `BadRequestError` si no es JSON o es más grande; el router convierte esa excepción en `400 {"error":{"kind":"bad_request"}}`. Las rutas que leen el cuerpo como stream (subida de archivos, historia 6) usan `req.raw` directamente.
- Los handlers `ws` pueden ser asíncronos (la historia 5 hace `await hc.attach(...)`); una excepción en uno cierra el socket con `1011`.

### Step 4: `auth.ts` (decisión 11)

- `export function createAuthorizer(opts: { whois(ipPort: string): Promise<string | null>; owner: string; ttlMs?: number; now?: () => number })` → `(remoteAddress, remotePort) => Promise<boolean>`.
- Normalizar `::ffff:100.x.y.z` a `100.x.y.z`. Llamar `whois("<ip>:<puerto>")` (`tailscale whois --json` acepta `<ip>:<puerto>` o la IP sola, verificado); aceptar solo si el `LoginName` devuelto es exactamente `owner`. Cachear el resultado por IP durante 60 s (`ttlMs` por defecto 60 000). Si `whois` falla (error o JSON inválido) → rechazar y **no** cachear.
- `export async function tailscaleWhois(bin: string, target: string): Promise<string | null>`: `execFile(bin, ["whois", "--json", target], { timeout: 5000 })`, parsea y devuelve `UserProfile.LoginName` (verificado: devuelve `kuamatzin@gmail.com` para la MacBook y para `mylinux`).
- `export async function tailscaleIPv4(bin): Promise<string>`: primera línea de `tailscale ip -4`.
- `export async function tailscaleOwner(bin): Promise<string>`: dueño del nodo con `tailscale status --self --json` → `User[String(Self.UserID)].LoginName`. Se lee al arrancar, antes de escuchar; si Tailscale no responde, reintentar cada 5 s sin escuchar. Una vez leído se conserva mientras el proceso vive. `tailscale ip -4` puede devolver varias líneas: se usa la primera IPv4.

### Step 5: `snapshot.ts`

`export function createSnapshotStore(opts: { homeDir: string; foldersFile?: string /* default ~/.config/herdr-space-folders.json */; gitRootOf?(cwd: string): string | null; now?(): number })` con:

- `attach(conn: HcConnection): Promise<void>` — en este orden (el README de `hc` pide suscribirse antes de leer documentos): `conn.subscribe()` (sus `terminals` son la lista inicial, equivalente a `terminal.list`), luego `conn.call("doc.get", { name: "herdr-app" })`, construir el modelo y emitir `{ type: "snapshot", ... }` a los oyentes.
- `detach()` — al morir `hc connect`; `ready()` pasa a falso.
- `close()` — deja de vigilar el archivo de carpetas (`fs.unwatchFile`) y suelta la suscripción; `main.ts` lo llama al salir. `attach` repetido (reconexión de `hc`) reutiliza el mismo vigilante, nunca crea otro.
- **Fail-closed del documento `herdr-app`:** el lector valida los campos que usa (`value.spaces` arreglo; cada espacio con `id` string, `name` string, `tabs` arreglo; cada pestaña con `id` string y `root` objeto; nodos con `type` string y, si `type === "block"`, `content.terminal` string). `value === null` (documento nunca escrito) es válido y da cero espacios. Cualquier otra forma inesperada: `get()` devuelve un error `layout_unsupported` y `/v1/snapshot` responde `503 {"error":{"kind":"layout_unsupported"}}` (y `/v1/events` no manda snapshot) en lugar de adivinar; se registra una línea en el log con la ruta del campo que falló.
- `get(): GatewaySnapshot | null`, `onMessage(listener: (m: GatewayEventMessage) => void): () => void`, `seq` propio del gateway (contador monotónico que sube con cada mensaje emitido; el `seq` de `hc` se reinicia con cada proceso del core y no se expone).
- **Eventos de `hc`** (notificación `event` con `params.type`), aplicados al registro `Terminal` como dice el README: `created`/`restarted` → agregar/reemplazar; `lost`/`finished` → actualizar (un `finished` quita la terminal del modelo → `terminalRemoved`); `removed` → `terminalRemoved`; `status` (`{status, records, reported}`), `turn` (`{turn}`), `title` (`{title}`), `labels`, `size`, `foreground` (`{pid,name,argv,cwd}` o null), `keep` → actualizar y emitir `{ type: "terminal", terminal }` solo si cambió algún campo de `GatewayTerminal`. `doc` con `data.name === "herdr-app"` → `doc.get` y emitir `{ type: "layout", spaces, folders }` (y `terminal` para las que cambiaron de `spaceId`/`tabId`). `heartbeat` → nada (el gateway manda su propio heartbeat). `missed` → rehacer todo: usar `params.terminals`, volver a leer `doc.get`, y emitir un `{ type: "snapshot", ... }` completo (la app lo reemplaza sin perder la navegación).
- **Mapeo `Terminal` → `GatewayTerminal`:** `address`; `spaceId`/`tabId` del documento (la pestaña cuyo árbol contiene un bloque con `content.terminal === address`); si el documento no la incluye, de sus etiquetas `labels["herdr.space"]` y `labels["herdr.tab"]` (las pone `POST /v1/tabs`, historia 5); si tampoco, `null`. Una terminal con `herdr.tab` que aún no está en el documento aparece además como pestaña sintética `{ id: <herdr.tab>, name: String(n), panes: [{ paneId: <address>, terminal: <address> }] }` al final de su espacio, hasta que el documento la incluya; `cwd = foreground?.cwd ?? cwd`; `title = title || null`; `turn = turn ?? 0`; `app = status?.app ?? null`; `status = status ? { state, kind: kind ?? null, msg: msg ?? null } : null`; `changedAtMs = status_ms ?? created_ms`. Las terminales con `state === "finished"` no se incluyen.
- **Documento `herdr-app`** (forma real verificada con `hc connect` en `mylinux`): `{ value: { spaces: [{ id: "sp…", name: "", folder: "/home/u" | "", tabs: [{ id: "tb…", name: "", root: { id, type: "row", children: [{ id: "p…", type: "block", content: { terminal: "t1" } }] } }] }] }, version: 4 }`. Los nodos del árbol pueden anidarse (`children`); los paneles son los nodos `type: "block"` con `content.terminal`, en orden de recorrido en profundidad; `paneId` = `id` del nodo. `folder` del documento es una **ruta**, no una carpeta de space-folders. Es un documento interno de Herdr sin contrato público: toda su lectura vive en este archivo, tolerante a campos faltantes.
- **Pestañas:** `name || String(índice + 1)`.
- **cwd del espacio** (`place()` del mod space-folders): el cwd (`foreground.cwd ?? cwd`) de la primera terminal de sus pestañas que tenga cwd; si no, `folder` del documento; si no, `homeDir`.
- **Nombre del espacio vacío** (como Herdr): si `cwd === homeDir` → `"~"`; si no, el nombre de la carpeta raíz del repo git que contiene el cwd (`gitRootOf`: subir desde el cwd buscando una entrada `.git` con `node:fs`, sin ejecutar `git`); si no hay repo, `basename(cwd)`.
- **Carpetas:** si existe `~/.config/herdr-space-folders.json` (`{ list: [{ id, name, folded, path?, match?, color? }], of: { [spaceId]: folderId }, at }`), cada carpeta sale como `{ id, name, color: color ?? null, folded: Boolean(folded), spaceIds }` en el orden de `list`. Asignación: explícita en `of` (`""` = fuera de toda carpeta; un id que ya no existe = ninguna); si no hay asignación, la carpeta cuya regla `match` alcanza más profundo en el cwd del espacio. Portar este código del mod `space-folders` (`jankeesvw/herdr-space-folders`, commit `63a78b2`, `mods/space-folders/main.js`, licencia MIT; poner el crédito en un comentario):
  ```js
  export function rule(pattern) {
    const p = pattern.trim().replace(/\/+$/, "");
    if (!p) return null;
    const body = p
      .replace(/^~/, "\u0000")
      .split("*")
      .map((part) => part.replace(/[.+?^${}()|[\]\\]/g, "\\$&"))
      .join("[^/]*")
      .replace("\u0000", "(?:/home/[^/]+|/Users/[^/]+|/root)");
    return new RegExp(`${/^[/~]/.test(p) ? "^" : "(?:^|/)"}${body}(?:/|$)`);
  }
  const reach = (f, path) =>
    !path
      ? 0
      : Math.max(
          0,
          ...(f.match ?? "").split(",").map((p) => {
            const m = rule(p)?.exec(path);
            return m ? 1 + m.index + m[0].replace(/\/$/, "").length : 0;
          }),
        );
  // La carpeta cuya regla llega más profundo; la primera en un empate.
  function deepest(list, score) {
    let best = null,
      most = 0;
    for (const f of list) {
      const n = score(f);
      if (n > most) {
        best = f;
        most = n;
      }
    }
    return best?.id ?? null;
  }
  function folderOf(spaceId, { list, of }, place) {
    const id = of[spaceId];
    if (id !== undefined) return list.some((f) => f.id === id) ? id : null;
    return deepest(list, (f) => reach(f, place));
  }
  ```
  Releer el archivo al reconstruir el layout y vigilarlo con `fs.watchFile(path, { interval: 5000 })` → emitir `layout` si cambió.

### Step 6: `main.ts`

- Variables: `HERDR_GATEWAY_PORT` (por defecto `7788`), `HERDR_HC_BIN` (por defecto `hc`), `HERDR_TAILSCALE_BIN` (por defecto `tailscale`).
- Escucha **solo** en la IP de `tailscale ip -4`, nunca en `0.0.0.0` ni `::`. Si Tailscale no responde al arrancar, reintentar cada 5 s sin escuchar en otra interfaz.
- **Supervisor de `hc connect`:** `spawn(HERDR_HC_BIN, ["connect"], { stdio: ["pipe", "pipe", "inherit"] })`, adaptar `stdin`/`stdout` a `ByteTransport` (un `Buffer` es un `Uint8Array`), `HcConnection.open(...)`; si `hello.generation !== 1`, cerrar esa conexión, no adjuntarla, y responder `/v1/health` con `503 {"error":{"kind":"core_incompatible"}}` hasta el siguiente intento; si es 1, `snapshot.attach(conn)`. Si el hijo muere o el transporte se cierra: `snapshot.detach()` y reintentar con espera creciente 1, 2, 4, 8, 16, 30, 30… s; volver a 1 s tras un `hello` exitoso.
- **Rutas propias:** `GET /v1/health` → `200` con `{ gatewayVersion: "0.1.0" (de package.json), hc: { version: hello.version, generation: hello.generation }, machine: { coreId: hello.core.id, name: hello.core.name || os.hostname() } }`, o `503 {"error":{"kind":"core_unavailable"}}` mientras `hc connect` no está conectado. `GET /v1/snapshot` → el snapshot o `503 core_unavailable`. `WS /v1/events` → primer mensaje `{ type: "snapshot", ... }` (en cuanto haya uno), luego cada mensaje del store, y `{ type: "heartbeat", seq }` cada 15 s.
- **Registro de rutas:** `registerTerminalRoutes(router, ctx)` de `routes/terminals.ts`, `registerFileRoutes(router, ctx)` de `routes/files.ts`, `registerChatRoutes(router, ctx)` de `routes/chat.ts`, `registerDeviceRoutes(router, ctx)` de `routes/devices.ts`, y `startPushNotifier(ctx)` de `push/notifier.ts` (cerrado al salir). Así las historias 5, 6, 13 y 17 solo editan sus propios archivos.
- **Stubs** (`501 {"error":{"kind":"not_implemented"}}`; los `ws` responden HTTP 501 en el upgrade):
  - `routes/terminals.ts`: `ws /v1/terminals/:address/stream`, `post /v1/terminals/:address/send`, `post /v1/terminals/:address/keys`, `post /v1/terminals/:address/interrupt`, `post /v1/tabs`.
  - `routes/files.ts`: `get /v1/files`, `get /v1/files/content`, `get /v1/files/download`, `post /v1/files/upload`.
  - `routes/chat.ts`: `ws /v1/agents/:address/chat`, `post /v1/agents/:address/chat/send`, `post /v1/agents/:address/chat/interrupt`, `post /v1/agents/:address/chat/approve`.
  - `routes/devices.ts`: `post /v1/devices`, `delete /v1/devices/:token`.

### Step 7: `scripts/install.sh`

`bash`, `set -euo pipefail`, idempotente. Con `--uninstall` deshace todo.

1. Verificar `node` (≥ 22), `hc`, `tailscale` y `vp` con `command -v`; guardar sus rutas absolutas.
2. Compilar: `(cd apps/herdr-gateway && vp pack)` desde la raíz del repo (resolver la raíz desde la ubicación del script). Copiar el archivo único resultante a `~/.local/share/herdr-gateway/herdr-gateway.mjs`.
3. Escribir `~/.config/systemd/user/herdr-gateway.service`:
   ```ini
   [Unit]
   Description=herdr gateway (Tailscale)
   [Service]
   ExecStart=<ruta absoluta de node> %h/.local/share/herdr-gateway/herdr-gateway.mjs
   Environment=HERDR_HC_BIN=<ruta absoluta de hc>
   Environment=HERDR_TAILSCALE_BIN=<ruta absoluta de tailscale>
   EnvironmentFile=-%h/.config/herdr-gateway/push.env
   Restart=always
   RestartSec=5
   [Install]
   WantedBy=default.target
   ```
   (Un servicio de usuario no hereda el `PATH` del shell: por eso rutas absolutas. El `-` de `EnvironmentFile=` lo hace opcional: el archivo de credenciales de push lo crea el usuario cuando se desbloquee la historia 17. Nunca poner `HERDR_GATEWAY_DEBUG` en este unit.)
4. `systemctl --user daemon-reload`, `systemctl --user enable herdr-gateway` y `systemctl --user restart herdr-gateway` (toma el build nuevo).
5. Si `loginctl show-user "$USER" -p Linger` dice `Linger=no`, imprimir el aviso `loginctl enable-linger $USER` (sin ejecutarlo).
6. Compatibilidad: `hc schema` comparado con `packages/herdr/fixtures/hc-schema.json` (por ejemplo `hc schema | diff -q - <fixture>`); si difiere, imprimir un aviso con la versión de `hc` instalada y la del fixture (`0.1.18`), sin abortar (el gateway igual falla cerrado si la generación no es 1).
7. Imprimir `http://$(tailscale ip -4 | head -1):${HERDR_GATEWAY_PORT:-7788}`.
8. `--uninstall`: `systemctl --user disable --now herdr-gateway`, borrar el unit y `~/.local/share/herdr-gateway/`, `daemon-reload`.
9. Organizar el script en funciones (`install_gateway`, `uninstall_gateway`) y dejar un bloque marcado `# --- herdr-tap y hooks (historias 11 y 12) ---` donde la historia 11 agrega lo suyo.

### Step 8: pruebas

- `auth.test.ts`: whois falso con reloj inyectado: dueño aceptado; otro `LoginName` → rechazado; caché de 60 s (no vuelve a llamar `whois` antes, sí después); falla de `whois` → rechazado y sin caché; `::ffff:` normalizado.
- `snapshot.test.ts` con un `HcConnection` falso (objeto con `subscribe`/`call` que reproduce un `SubscribeResult` y eventos grabados): snapshot con el documento real de arriba; nombres de espacio (`~`, raíz git con `gitRootOf` falso, basename); pestaña sin nombre → `"1"`; paneles anidados; carpetas por `of`, por `match` más profundo y `""` = ninguna; evento `status` → mensaje `terminal`; `finished` → `terminalRemoved`; `doc` → `layout`; `missed` → `snapshot` completo con las terminales nuevas; documento con forma inesperada (por ejemplo `tabs` no arreglo) → error `layout_unsupported`; `value: null` → cero espacios.
- En `http.test.ts`, con la función de salud extraída de `main.ts` y un `HcConnection` falso cuyo `hello.generation` es 2, `/v1/health` responde `503 core_incompatible`.
- `http.test.ts`: con `createGatewayServer` en `127.0.0.1` y puerto `0` (solo en la prueba) y `authorize` falso: `403 forbidden` en HTTP y en upgrade WebSocket; un stub responde `501`; ruta inexistente `404`.

## Out of Scope

- Implementar el stream de terminal, `send`/`keys`/`interrupt` y `POST /v1/tabs`: historia 5 (aquí solo stubs `501` en `routes/terminals.ts`).
- Implementar archivos: historia 6 (stubs en `routes/files.ts`).
- Implementar chat: historia 13 (stubs en `routes/chat.ts`); el receptor de hooks en `127.0.0.1:7789`: historia 12; `herdr-tap`: historia 11.
- Dispositivos push y notificador (`POST`/`DELETE /v1/devices`, APNs, FCM): historia 17 (aquí solo stubs y la línea `EnvironmentFile=`).
- Mostrar el árbol de máquinas de `hc machine` (máquinas que un core alcanza por SSH): fuera del alcance; terminales con dirección `machine/t7` se ignoran.
- Crear, renombrar o eliminar espacios, o renombrar/cerrar pestañas: fuera del alcance.
- Escribir en `~/.config/herdr-space-folders.json`: el gateway solo lo lee.

## Acceptance Criteria

- [ ] Dentro de `apps/herdr-gateway`: `vp test run src/auth.test.ts src/snapshot.test.ts src/http.test.ts` pasa.
- [ ] Dentro de `apps/herdr-gateway` y de `packages/herdr`: `vp run typecheck` pasa.
- [ ] `grep -rn "0\.0\.0\.0\|\"::\"" apps/herdr-gateway/src` no encuentra nada; el servidor real solo escucha en la IP de `tailscale ip -4`.
- [ ] `grep -rn "from \"effect\|@t3tools/" apps/herdr-gateway/src packages/herdr/src` no encuentra nada.
- [ ] Dentro de `apps/herdr-gateway`, `vp pack` produce un solo archivo JS que arranca con `node` copiado fuera del repo (sin `node_modules`).
- [ ] En `mylinux`, `apps/herdr-gateway/scripts/install.sh` deja `systemctl --user is-active herdr-gateway` en `active`, imprime `http://100.…:7788`, y correrlo dos veces no cambia nada; desde un nodo del dueño `curl http://<ip>:7788/v1/health` responde `200` con `hc.version`; `--uninstall` lo quita.
- [ ] `node scripts/herdr/check-patch-surface.ts` sale con código 0.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
