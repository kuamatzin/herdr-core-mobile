---
status: not_started
feature: "herdr-core-mobile"
story: "05"
layers:
  - "backend"
dependencies:
  - "story-04"
---

# Story 05: Gateway, terminal

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code que agrega una app móvil para herdr-core (`hc`). El gateway por servidor (`apps/herdr-gateway`, historia 4) ya expone health, snapshot y eventos, y tiene stubs `501` para las rutas de terminal. Esta historia implementa esas rutas en `routes/terminals.ts`: el stream WebSocket de una terminal (salida en vivo, teclado, cambio de tamaño, control de flujo con créditos), `send`/`keys`/`interrupt`, y abrir una pestaña nueva en un espacio. La app (historia 9) monta la terminal nativa sobre este stream.

## Dependencies

- Depends on: Story 04 — Gateway base (`apps/herdr-gateway/src/http.ts`: `Router`, `GatewayContext`, `sendJson`, `sendError`; `snapshot.ts`; stubs en `routes/terminals.ts`; tipos en `packages/herdr/src/gateway-api.ts`). Usa `HcConnection.attach()`/`call()` de `@herdr/core-client/hc` (historia 3).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Tamaño de la terminal?** → **Decision:** se ajusta **siempre** al teléfono. La app manda su tamaño en celdas al abrir y al rotar. En herdr-core, el último visor activo define el tamaño, así que el escritorio se reacomoda mientras el teléfono la usa y lo recupera cuando el escritorio vuelve a enfocarla.
4. **Q: ¿Seguridad del gateway?** → **Decision:** solo Tailscale. El gateway escucha únicamente en la IP de Tailscale del servidor (`tailscale ip -4`), nunca en `0.0.0.0`. Además, en cada conexión HTTP y WebSocket consulta `tailscale whois --json <ip>:<puerto>` del cliente y acepta solo si `UserProfile.LoginName` coincide con el dueño del propio nodo. Si no coincide, responde `403`. Ejemplo verificado: la MacBook del usuario resuelve a `kuamatzin@gmail.com`, igual que `mylinux`. Un dispositivo de las socias de Protact sería rechazado aunque vea el puerto.
5. **(plan) Gateway sin Effect.** El gateway y `packages/herdr` usan TypeScript plano con Promises y APIs de Node, no Effect, para no acoplarse a la evolución de las librerías del servidor T3. `packages/herdr` no importa nada de `@t3tools/*`.

## Affected Files

### Backend

- [ ] Pattern to follow: `apps/server/src/terminal/OutputProtocol.ts` y `withTerminalOutputWindow` (solo como referencia de ventanas de salida y control de flujo; no importar). El router y las ayudas de `apps/herdr-gateway/src/http.ts` (historia 4).
- [ ] Conventions to follow: TypeScript plano con `node:http`, `ws` y `node:string_decoder` (decisión 14); errores `{ "error": { "kind", "message" } }` con códigos HTTP explícitos (`sendError`); sin `enum`/`namespace`; pruebas `*.test.ts` junto al archivo, sin sleeps ni `Date.now()`.
- [ ] `apps/herdr-gateway/src/routes/terminals.ts` — reemplazar los stubs por la implementación.
- [ ] `apps/herdr-gateway/src/terminalStream.ts` — nuevo: puente entre el WebSocket y el `HcAttachStream`.
- [ ] `packages/herdr/src/gateway-api.ts` — solo agregar `TERMINAL_STREAM_CLOSE_CODES` (ver Step 1); ninguna otra historia de la ola 3 edita este archivo.
- [ ] Pruebas: `apps/herdr-gateway/src/terminalStream.test.ts`, `apps/herdr-gateway/src/routes/terminals.test.ts`.

## Implementation Steps

### Step 1: contrato

Del API Contract (binding):

**`WS /v1/terminals/:address/stream?cols=80&rows=24&history=2000`**: servidor → `{ "type": "output", "data": "…" }`, `{ "type": "exit", "reason": "exited", "exitCode": 0 }`; cliente → `{ "type": "input", "data": "…" }`, `{ "type": "resize", "cols": 60, "rows": 30 }`.

**`POST /v1/terminals/:address/send`** `{ "text": "…" }` → `{ "turn": 19, "started": true }`. **`/keys`** `{ "keys": ["ctrl+c"] }`. **`/interrupt`** `{}`.

**`POST /v1/tabs`** `{ "spaceId": "sp…", "argv": ["claude"] }` (`argv` opcional; sin él, shell) → `{ "address": "t9", "tabId": "tb…" }`. El gateway usa `terminal.run` con `cwd` = cwd del espacio, `keep: true` y las etiquetas `herdr.space=<spaceId>` y `herdr.tab=<id nuevo>`.

Todas las rutas pasan por la autorización de la decisión 11 que ya aplica `createGatewayServer` (historia 4); no repetirla.

Cierres del WebSocket cuando no se puede abrir el stream (agregar a `gateway-api.ts` como `export const TERMINAL_STREAM_CLOSE_CODES = { notFound: 4404, coreUnavailable: 4503, badRequest: 4400 } as const;`, la app los usa en la historia 9).

### Step 2: `terminalStream.ts`

`export function bridgeTerminalStream(opts: { socket: StreamSocket; stream: HcAttachStream; maxUndeliveredBytes?: number /* 1 MiB */ }): void` donde `StreamSocket` es la parte de `ws.WebSocket` que se usa (`send(data, cb)`, `bufferedAmount`, `on("message" | "close")`, `close(code, reason)`), para poder probarla con un socket falso.

- **Salida:** un `StringDecoder("utf8")` por stream. Por cada `stream.onData(bytes)`: `text = decoder.write(bytes)`; si no está vacío, `socket.send(JSON.stringify({ type: "output", data: text }), cb)`. Un carácter partido entre dos chunks se emite completo en el segundo (nunca `�` por un corte).
- **Créditos (cola explícita):** un contador `pendingCredit` acumula los **bytes crudos** de cada chunk cuando su `send` termina (`cb` sin error). Regla de liberación `flushCredit()`: si `socket.bufferedAmount <= maxUndeliveredBytes` (1 MiB), llamar `stream.consumed(pendingCredit)` y dejarlo en 0; si no, programar un nuevo intento en 50 ms (un solo temporizador a la vez) y repetir hasta que baje. `flushCredit()` se llama en cada `cb` de `send` y desde ese temporizador, así los créditos se liberan aunque no llegue otro `send`; el temporizador se cancela al cerrar. El core da a cada stream 256 KiB de crédito al abrirlo; este código nunca manda un `CREDIT` inicial. Un visor con más de 4 MiB de atraso recibe un repintado del core.
- **Entrada:** mensaje `{ type: "input", data: string }` (máximo 64 KiB por mensaje; uno más grande se ignora) → `stream.write(new TextEncoder().encode(data))`. Los mensajes WebSocket del cliente de más de 128 KiB cierran el socket con `1009`. `{ type: "resize", cols, rows }` con enteros entre 1 y 1000 → `stream.resize(cols, rows)` (decisión 10: el último visor activo define el tamaño; no hay modo espejo). Mensajes inválidos o de otro tipo se ignoran.
- **Cierre por el cliente:** `socket` `close` → `stream.close()` (manda `CLOSE`; el core contesta con `CLOSE` vacío).
- **Cierre por el core:** `stream.onClose(reason)` con razón (`{reason, exit_code, signal}`) → `socket.send({ type: "exit", reason: reason.reason, exitCode: reason.exit_code })` y `socket.close(1000)`. Con `onClose(null)` (respuesta a nuestro `CLOSE`) → nada más. Si `hc connect` muere, la historia 3 llama `onClose({ reason: "connection_lost", exit_code: null, signal: null })` en cada stream: en ese caso **no** se manda `exit`; se cierra el WebSocket con `4503` (`core_unavailable`) para que la app reconecte (la terminal sigue viva en el core).
- Al terminar, vaciar el decoder (`decoder.end()`) y mandarlo si quedó algo antes del `exit`.

### Step 3: `routes/terminals.ts`

Mantener la firma `export function registerTerminalRoutes(router: Router, ctx: GatewayContext): void`.

- `router.ws("/v1/terminals/:address/stream", ...)`: leer `cols`, `rows` (enteros 1–1000, obligatorios) e `history` (por defecto 2000, 0–100 000). Inválidos → `socket.close(4400, "bad_request")`. Sin `ctx.hc()` → `close(4503, "core_unavailable")`. `await hc.attach(address, { cols, rows, history })`; error `HcCallError` con `kind === "not_found"` → `close(4404, "not_found")`; otro error → `close(4503, ...)`. Si el socket se cerró mientras se esperaba el attach, cerrar el stream recién abierto. Luego `bridgeTerminalStream(...)`.
- Límites de entrada (todas las rutas): `address` debe cumplir `^[A-Za-z0-9._-]{1,64}$` (las direcciones `machine/t7` no se aceptan; fuera del alcance) o `400`.
- `router.post("/v1/terminals/:address/send", ...)`: cuerpo `{ text: string }` (máximo 64 KiB; si no, `400 bad_request`) → `hc.call("terminal.send", { terminal: address, text })` → `200 { turn: r.turn, started: r.started ?? null }` (ajustar `TerminalSendResult` a `started: boolean | null` si la historia 4 lo dejó como `boolean`).
- `router.post("/v1/terminals/:address/keys", ...)`: `{ keys: string[] }` (1–32 nombres, cada uno de 1–32 caracteres `[a-z0-9+_-]`; nombres de `hc`: `enter`, `esc`, `tab`, `up`, `ctrl+c`, `alt+x`, `f5`, …; `hc` valida el nombre y su error `bad_request`/`usage` se devuelve como `400`) → `hc.call("terminal.keys", { terminal: address, keys })` → `200 { turn }`.
- `router.post("/v1/terminals/:address/interrupt", ...)`: → `hc.call("terminal.interrupt", { terminal: address })` → `200 { outcome, key }`.
- `router.post("/v1/tabs", ...)`: `{ spaceId: string; argv?: string[] }`. Buscar el espacio en `ctx.snapshot.get()`; si no existe → `404 not_found`. `tabId = "tb" + randomBytes(16).toString("hex")` (id nuevo al estilo de Herdr: `tb` + 32 hex). `hc.call("terminal.run", { cwd: space.cwd, keep: true, labels: { "herdr.space": spaceId, "herdr.tab": tabId }, ...(argv?.length ? { argv } : {}) })` → `200 { address: r.address, tabId }`. Hecho verificado en la documentación de herdr-app: una terminal con las etiquetas `herdr.space=<id>` y `herdr.tab=<id>` aparece en Herdr aunque la arranque otro (`hc run --keep --label herdr.space=<id> --label herdr.tab=<id> -- claude`), y un id que Herdr no conoce se vuelve una pestaña nueva; por eso la pestaña aparece también en el escritorio. (`snapshot.ts` de la historia 4 además la asigna por estas etiquetas mientras el documento `herdr-app` no la incluya.)
- Errores de `hc`: `not_found` → `404`; conexión ausente → `503 core_unavailable`; otro `HcCallError` → `502 { error: { kind: <data.kind>, message } }`.

### Step 4: pruebas

- `terminalStream.test.ts` con socket y stream falsos: UTF-8 partido (`"é"` = `0xC3 0xA9` en dos chunks → un solo `output` con `"é"`), `input` → bytes correctos a `stream.write`, `resize` válido e inválido, cierre del socket → `stream.close()`, `CLOSE` del core con razón → `exit` con `exitCode` y cierre `1000`, créditos devueltos en el callback de `send`, retenidos mientras `bufferedAmount > 1 MiB` y liberados por el temporizador cuando baja sin otro `send` (temporizadores falsos); `onClose` con `connection_lost` → cierre `4503` sin `exit`.
- `routes/terminals.test.ts`: `POST /v1/tabs` arma `terminal.run` con `cwd` del espacio, `keep: true` y las dos etiquetas; espacio inexistente → `404`; `send` sin `text` → `400`; `tabId` cumple `^tb[0-9a-f]{32}$`; `keys` con 33 nombres → `400`.

### Step 5: comprobación en el escritorio (criterio de aceptación)

Con el gateway instalado en `mylinux` y la app de escritorio Herdr conectada a ese core: `POST /v1/tabs` con `argv` vacío y con `["claude"]`. La pestaña nueva **debe** aparecer en el escritorio, en ese espacio. Si no aparece, la historia no está terminada: revisar las etiquetas (`hc ls` / `hc get <t>`), el `spaceId` y el formato del id, sin recurrir al snapshot del gateway como sustituto. Anotar el resultado en `## Implementation Notes` del tracker (story-05).

## Out of Scope

- Archivos (historia 6), chat (historia 13), hooks (historia 12), push (historia 17).
- La UI de la terminal y la hoja de pestaña nueva en la app: historia 9.
- Escribir en la terminal en modo espejo sin ajustar el tamaño (decisión 10): no existe `observe` en la v1.
- Crear, renombrar o eliminar espacios; renombrar o cerrar pestañas: fuera del alcance (solo se abren pestañas nuevas).
- `AUTOMATIC_INPUT`, `FOCUS` y `FACTS`: no se mandan.
- Usar `ssh` desde el teléfono o el árbol de `hc machine`.

## Acceptance Criteria

- [ ] Dentro de `apps/herdr-gateway`: `vp test run src/terminalStream.test.ts src/routes/terminals.test.ts` pasa.
- [ ] Dentro de `apps/herdr-gateway` y de `packages/herdr`: `vp run typecheck` pasa.
- [ ] Las rutas de `routes/terminals.ts` ya no responden `501`.
- [ ] Ningún otro archivo del gateway cambia salvo `routes/terminals.ts`, `terminalStream.ts`, sus pruebas y la constante en `gateway-api.ts`.
- [ ] **Binding:** una pestaña creada con `POST /v1/tabs` aparece en la app de escritorio Herdr en ese espacio (Step 5), y el resultado está anotado en el tracker.
- [ ] `node scripts/herdr/check-patch-surface.ts` sale con código 0.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
