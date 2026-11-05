---
status: not_started
feature: "herdr-core-mobile"
story: "13"
layers:
  - "backend"
dependencies:
  - "story-11"
  - "story-12"
---

# Story 13: Gateway, sesión de chat

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code que agrega una app móvil para herdr-core, con un chat nativo sobre la misma sesión de Claude Code que corre en la terminal de herdr. Ya existen las tres fuentes: el proxy `herdr-tap` emite el texto y las herramientas palabra por palabra a un socket local (historia 11), los hooks llegan al receptor `127.0.0.1:7789` con el mapa sesión ↔ terminal y las decisiones pendientes (historia 12), y la transcripción `.jsonl` de Claude guarda el historial. Esta historia las une por terminal en un modelo de ítems de chat y lo expone en `WS /v1/agents/:address/chat`, con enviar, detener y aprobar.

## Dependencies

- Depends on: Story 11 — Proxy `herdr-tap` (líneas JSON en `$XDG_RUNTIME_DIR/herdr-gateway/tap.sock` con `{ sessionId, agentId, requestId, event, data }`; el tap se conecta, el gateway escucha).
- Depends on: Story 12 — Hooks de Claude Code (`ctx.hooks: HookHub` con `onEvent`, `onDecision`, `terminalForSession`, `sessionForTerminal`, `pending`, `resolve`, `setViewerCheck`; resultados de la comprobación en `docs/herdr/hooks.md` — leerlo antes de empezar).
- Usa de la historia 4: `routes/chat.ts` (stubs `501`), `Router`, `GatewayContext` (`hc()`, `snapshot`, `homeDir`, `hooks`), `sendJson`, `sendError`.

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Cómo se alimenta el chat de Claude Code con los mismos agentes del escritorio?** → **Decision:** dos fuentes oficiales, con el Claude Code real intacto en su terminal:
   - **Proxy `herdr-tap`** vía `ANTHROPIC_BASE_URL`: texto y herramientas palabra por palabra, correlacionados con el header `x-claude-code-session-id`.
   - **Hooks de Claude Code**: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`, `Notification`, `Stop`, `SessionEnd`.
   - **Transcripción** (`~/.claude/projects/<proyecto>/<session-id>.jsonl`): solo para el historial al abrir el chat, porque su formato es interno y puede cambiar.
   - **Envío desde el teléfono**: `terminal.send` de herdr-core, sobre la misma terminal.
4. **(plan) Gateway sin Effect.** El gateway y `packages/herdr` usan TypeScript plano con Promises y APIs de Node, no Effect, para no acoplarse a la evolución de las librerías del servidor T3. `packages/herdr` no importa nada de `@t3tools/*`.
5. **(plan) El proxy es un proceso aparte.** `herdr-tap` corre como su propio servicio systemd y reenvía a `https://api.anthropic.com` sin depender del gateway. Si el gateway se reinicia o cae, los agentes no se cortan. `herdr-tap` nunca guarda credenciales ni cuerpos de petición o respuesta; solo emite eventos por un socket Unix local, sin esperar respuesta.
6. **(plan) Aprobaciones sin bloquear al escritorio.** El hook solo espera la decisión del teléfono si hay al menos una app con el chat de ese agente abierto en ese momento. Si no, responde de inmediato sin decidir y Claude sigue su flujo normal en la terminal. La espera máxima es 120 s; al vencer, también sigue el flujo normal. Una espera también se libera sin decidir al llegar, para la misma sesión, cualquiera de estos eventos: `PostToolUse`, `PostToolUseFailure`, `PermissionDenied`, `Stop`, `UserPromptSubmit` o `SessionEnd` (nunca `Notification`). Respaldo aprobado: si la comprobación con Claude real muestra que responder en la terminal no termina la espera del hook, la espera máxima baja a 30 s y se anota en `docs/herdr/hooks.md` y en el tracker.
7. **(plan) Sugerencias de permiso solo informativas.** Las `permission_suggestions` de Claude Code se muestran en la tarjeta solo como información; desde el teléfono solo se responde `allow` o `deny`, porque el contrato de `/approve` no tiene cómo elegir una sugerencia.
8. **(plan) Aprobaciones solo con el chat abierto.** Una aprobación o pregunta llega al teléfono solo mientras el chat de ese agente está abierto en el teléfono. Una petición que se levantó antes de abrirlo, o mientras se ve en modo Terminal, se queda en la terminal. Es el comportamiento buscado y se prueba.

## Affected Files

### Backend

- [ ] Pattern to follow: el modelo de ítems del API Contract (abajo). El router y el contexto de `apps/herdr-gateway/src/http.ts`; `HookHub` de `apps/herdr-gateway/src/hooks.ts`; el formato de líneas del tap en `apps/herdr-gateway/src/tap/sse.ts` (`toTapEvent`) y `docs/herdr/tap.md`.
- [ ] Conventions to follow: TypeScript plano con `node:net`, `node:fs`, `node:readline` (decisión 14); errores `{ "error": { "kind", "message" } }` con códigos HTTP explícitos; lector de transcripción tolerante (entradas desconocidas se ignoran); sin `enum`/`namespace`; pruebas `*.test.ts` junto al archivo con secuencias grabadas, sin sleeps.
- [ ] `apps/herdr-gateway/src/routes/chat.ts` — reemplazar los stubs.
- [ ] `apps/herdr-gateway/src/chat/session.ts` — nuevo: modelo de chat por terminal.
- [ ] `apps/herdr-gateway/src/chat/claudeTranscript.ts` — nuevo: historial desde el `.jsonl`.
- [ ] `apps/herdr-gateway/src/chat/tapSocket.ts` — nuevo: escucha del socket del tap.
- [ ] `apps/herdr-gateway/src/main.ts` — dueño del socket del tap: lo crea al arrancar, lo conecta al chat y lo cierra al salir (ninguna otra historia de la ola 4 toca el gateway fuera de esta).
- [ ] `packages/herdr/src/gateway-api.ts` — tipos de chat.
- [ ] Pruebas: `apps/herdr-gateway/src/chat/session.test.ts`, `apps/herdr-gateway/src/chat/claudeTranscript.test.ts`, `apps/herdr-gateway/src/chat/tapSocket.test.ts`.

## Implementation Steps

### Step 1: contrato y tipos (`gateway-api.ts`)

Del API Contract (binding):

**`WS /v1/agents/:address/chat`** → primer mensaje `{ "type": "snapshot", "source": "claude" | "codex", "items": [ChatItem], "historyUnavailable": false, "streaming": true }`; luego `{ "type": "item", "item": ChatItem }` (alta o reemplazo por `id`), `{ "type": "delta", "id": "…", "text": "…" }` y `{ "type": "turnEnded" }`.

`ChatItem` es uno de (el gateway trunca `input` y `output` de herramientas a 4 KB):

- `{ "kind": "user", "id", "text", "atMs" }`
- `{ "kind": "assistant", "id", "text", "streaming": boolean }`
- `{ "kind": "tool", "id", "name", "input": object, "status": "running" | "done" | "error", "output": string | null }`
- `{ "kind": "approval", "id": requestId, "tool": string, "input": object, "suggestions": [...], "questions": [...] | null, "state": "pending" | "resolved" | "expired" }`

**`POST /v1/agents/:address/chat/send`** `{ "text": "…" }`. **`/interrupt`** `{}`. **`/approve`** `{ "requestId": "…", "decision": "allow" | "deny", "answers": { … } | null }` → `409 {"error":{"kind":"not_pending"}}` si ya se resolvió o venció.

Exportar: `ChatSource = "claude" | "codex"`, `ChatUserItem`, `ChatAssistantItem`, `ChatToolItem`, `ChatApprovalItem`, `ChatItem`, `ChatServerMessage` (unión de `snapshot`, `item`, `delta`, `turnEnded`), `ChatSendRequest`, `ChatApproveRequest` (`answers: Record<string, string> | null`), y `CHAT_CLOSE_CODES = { notFound: 4404, unsupportedAgent: 4415 } as const`.

Validation & Error Handling (binding): chat sin datos del tap (por ejemplo, sesiones con `claude-glm`): el chat funciona con hooks y transcripción, sin palabra por palabra, y lo indica (`streaming: false`; la app muestra "Sin streaming en vivo"). Aprobación respondida en la terminal antes que en el teléfono: la tarjeta pasa a `resolved` y el botón ya no envía nada (`409`).

### Step 2: `chat/tapSocket.ts`

`startTapSocket(path: string, onLine: (m: TapLine) => void): Promise<{ close(): Promise<void> }>`: `mkdir -p` de la carpeta con modo `0700` (y `chmod 0700` si ya existía); si ya hay algo en `path`, intentar conectarse: si responde, otro gateway lo está usando → error claro (no se borra); si no responde (socket viejo), borrarlo. `net.createServer` en `path` y `chmod 0600` del socket en cuanto escucha; `close()` cierra el servidor y borra el socket. leer líneas JSON (un objeto por `\n`, ver la desviación de la historia 11 en `docs/herdr/tap.md`), ignorar líneas inválidas. `path` = `$XDG_RUNTIME_DIR/herdr-gateway/tap.sock` (si no hay `XDG_RUNTIME_DIR`, `/run/user/<uid>`). `tapSocket.test.ts`: con un socket temporal, una línea partida en dos escrituras llega entera; una línea inválida se ignora; el directorio queda 0700 y el socket 0600; un socket viejo sin dueño se reemplaza y uno vivo provoca error.

### Step 3: `chat/claudeTranscript.ts`

`readClaudeTranscript(path: string, opts?: { limit?: number /* 200 */ }): Promise<ChatItem[]>`:

- Leer solo la cola del archivo (últimos 8 MiB, descartando la primera línea parcial) y procesar línea por línea; `JSON.parse` en `try`.
- Registros conocidos (formato interno de Claude Code, puede cambiar): `{ type: "user", uuid, timestamp, message: { role: "user", content: string | Block[] } }` y `{ type: "assistant", uuid, message: { content: Block[] } }`. Saltar los que tengan `isMeta: true` o `isSidechain: true`.
- `user` con `content` string (o bloques `text`) → `ChatUserItem` (`id = uuid`, `atMs = Date.parse(timestamp)`); bloques `tool_result` → completan el `ChatToolItem` con el mismo `tool_use_id` (`output` como texto, truncado a 4 KB; `status: "error"` si `is_error`).
- `assistant`: bloques `text` → `ChatAssistantItem` (`id = uuid + ":" + índice`, `streaming: false`); bloques `tool_use` → `ChatToolItem` (`id = block.id`, `name`, `input` —si su JSON pasa de 4 KB, se reemplaza por `{ "truncated": "<primeros 4 KB del JSON>" }`—, `status: "done"`, `output: null` hasta su resultado); `thinking` se ignora.
- Cualquier otro registro o bloque se ignora. Devuelve los últimos `limit` ítems. Si el archivo no existe o no se puede leer, lanza (la sesión marca `historyUnavailable: true`).
- **Ubicar el archivo:** solo con `hooks.session(sessionId).transcriptPath`, el `transcript_path` que dio un hook de esa sesión (el mapa de sesiones se guarda en `sessions.json` y sobrevive a reinicios del gateway, historia 12). **Nunca** se adivina por cwd ni por el archivo más reciente. Sin `transcriptPath` → `historyUnavailable: true`.
- `claudeTranscript.test.ts` con un fixture grabado a mano en `apps/herdr-gateway/src/chat/fixtures/claude-transcript.jsonl` (líneas representativas: prompt, texto, `tool_use`, `tool_result`, un registro desconocido, una línea rota, un `isSidechain`).

### Step 4: `chat/session.ts`

`createChatHub(opts: { hooks: HookHub; snapshot: SnapshotStore; readTranscript?: typeof readClaudeTranscript; homeDir: string })` con:

- `open(address, listener: (m: ChatServerMessage) => void): Promise<() => void>` (cuenta visores por terminal), `viewers(address): number`, `onTapLine(line)`.
- Al crear la hub: `hooks.setViewerCheck((address) => viewers(address) > 0)` (decisión 17: el hook solo espera si hay una app con ese chat abierto).
- **Una sesión por terminal** (`address`), creada al primer `open`: `sessionId = hooks.sessionForTerminal(address)`, historial con `readTranscript` (sin `transcriptPath` o error → `historyUnavailable: true`), primer mensaje `snapshot` con `source: "claude"`, `items`, `historyUnavailable`, `streaming` (verdadero si ya llegó alguna línea del tap con ese `sessionId`).
- **Chat abierto antes de que haya sesión:** si la terminal aún no tiene sesión mapeada (Claude recién abierto o sin hooks todavía), el primer `snapshot` va con `items: []` e `historyUnavailable: true`; cuando `hooks.onSessionMapped` asocia una sesión a esa terminal (o la cambia), se lee la transcripción de esa sesión y se manda un `snapshot` nuevo.
- **Hooks** (`hooks.onEvent`), resueltos a la terminal con `e.address ?? hooks.terminalForSession(e.sessionId)`:
  - `SessionStart` (nueva sesión en la misma terminal, por ejemplo `/clear`) → reiniciar ítems, releer la transcripción y mandar `snapshot`.
  - `UserPromptSubmit` → `ChatUserItem` (`id = "u-" + randomUUID()`, `text = payload.prompt`, `atMs`), y el turno queda abierto.
  - `PreToolUse` → `ChatToolItem` (`id = payload.tool_use_id`, `name`, `input`, `status: "running"`, `output: null`), alta o reemplazo.
  - `PostToolUse` → mismo ítem con `status: "done"` (o `"error"` si `tool_response.is_error`) y `output` (texto, truncado a 4 KB). `PostToolUseFailure` → `status: "error"`.
  - `Stop` → cerrar el turno (ver regla abajo), `turnEnded`.
  - `Notification`, `SessionEnd` → sin ítems (en `SessionEnd`, el siguiente `open` vuelve a buscar la sesión).
- **Decisiones** (`hooks.onDecision(p, state)`) → `ChatApprovalItem` `{ kind: "approval", id: p.requestId, tool: p.toolName, input: p.toolInput, suggestions: p.suggestions, questions: p.questions, state }`, alta o reemplazo. Al abrir un chat, las `hooks.pending(address)` vigentes entran al `snapshot`.
- **Tap sin terminal todavía (carrera):** si `hooks.terminalForSession(line.sessionId)` es `null` (la primera petición del tap puede llegar antes que el primer hook), la línea se guarda en un búfer por `sessionId`, como máximo 30 s o 500 líneas por sesión (lo primero que se cumpla descarta las más viejas o el búfer entero al vencer). Cuando `hooks.onSessionMapped(sessionId, …)` llega, el búfer se reproduce en orden por las reglas de abajo; si no llega a tiempo, se descarta.
- **Versión de Claude:** de `message_start.data.clientVersion` → `hooks.setClaudeVersion(sessionId, version)` cuando no es `null`.
- **Tap** (`onTapLine`), resuelto con `hooks.terminalForSession(line.sessionId)`; se ignora fuera de un turno abierto:
  - `message_start` → `ChatAssistantItem` `{ id: "a-" + requestId, text: "", streaming: true }`.
  - `content_block_delta` con `text` → anexar y mandar `{ type: "delta", id, text }` (sin reenviar el ítem completo).
  - `content_block_start` con `type: "tool_use"` → `ChatToolItem` con `id` y `name` del bloque, `input: {}`, `status: "running"`; `partialJson` se acumula y en `content_block_stop` se parsea a `input` (si falla, se deja `{}`; si pasa de 4 KB, `{ "truncated": "…" }`) y se manda `item`.
  - **Unión de herramientas por id:** el `id` de un bloque `tool_use` del tap es el mismo `tool_use_id` que traen `PreToolUse`/`PostToolUse` (ambos son el id de Anthropic, `toolu_…`). Puede llegar primero cualquiera de los dos: el primero crea el ítem; el segundo lo completa sin duplicarlo —el tap aporta `name` e `input`, los hooks aportan `status` y `output`—, y un `status` `done`/`error` nunca vuelve a `running`.
  - `thinking` se ignora.
  - La primera línea del tap para un `sessionId` pone `streaming: true`; si había chats abiertos con `streaming: false`, se les manda un `snapshot` nuevo.
- **Regla de cierre de turno (`Stop`):** Claude Code hace peticiones secundarias (títulos, etc.) con el mismo `sessionId`. Al `Stop`: (1) de los ítems `assistant` del tap creados en este turno, se quitan los de peticiones que no tuvieron ningún bloque `tool_use` y no son la última petición del turno; (2) el último ítem `assistant` toma `text = payload.last_assistant_message` si viene (si no hubo tap en el turno, se crea uno con ese texto); (3) todos quedan `streaming: false`. Como el contrato no tiene mensaje de borrado, si se quitó algún ítem se manda un `snapshot` completo; si no, los `item` cambiados. Luego `turnEnded`.
- **Liberación de memoria:** una sesión sin visores se descarta 10 minutos después de cerrarse el último.

### Step 5: `routes/chat.ts`

Cambiar la firma a `export function registerChatRoutes(router: Router, ctx: GatewayContext): ChatHub`: crea `createChatHub({ hooks: ctx.hooks, snapshot: ctx.snapshot, homeDir: ctx.homeDir })`, registra las rutas y devuelve el hub.

**`main.ts` (dueño del socket):** `const chat = registerChatRoutes(router, ctx)`; luego `const tap = await startTapSocket(path, (line) => chat.onTapLine(line))`. Si `startTapSocket` falla (otro gateway vivo, permisos), registrar el error en el log y reintentar cada 30 s mientras el proceso vive; el chat funciona sin tap (`streaming: false`) mientras tanto. En `SIGTERM`/`SIGINT`, `await tap.close()` antes de salir.

- `ws /v1/agents/:address/chat`: la terminal debe existir en el snapshot (si no → cerrar con `4404`); su `app` debe ser `claude-code` (otro → `4415`; la historia 16 agrega `codex`). Mandar cada mensaje como JSON; al cerrar el socket, quitar el visor.
- `post /v1/agents/:address/chat/send` `{ text }` → `hc.call("terminal.send", { terminal: address, text })` → `200 { turn, started }` (sin `hc` → `503 core_unavailable`; sin `text` → `400`).
- `post /v1/agents/:address/chat/interrupt` → `hc.call("terminal.interrupt", { terminal: address })` → `200 { outcome, key }`.
- `post /v1/agents/:address/chat/approve` `{ requestId, decision, answers }` → `hooks.resolve(requestId, { decision, answers })`; `"not_pending"` → `409 {"error":{"kind":"not_pending"}}`; `"ok"` → `200 {}`. `decision` distinto de `allow`/`deny` → `400`.

### Step 6: pruebas (`session.test.ts`)

Con un `HookHub` falso (o el real de `createHookHub` con temporizadores falsos), un snapshot falso y `readTranscript` inyectado, reproduciendo secuencias grabadas de tap y hooks:

- Turno simple: `UserPromptSubmit` → tap `message_start` + 3 `text` → `Stop` con `last_assistant_message` → mensajes `item`(user), `item`(assistant), 3 `delta`, `item`(assistant final, `streaming: false`), `turnEnded`.
- Herramienta: tap `tool_use` con `partialJson` partido → `PreToolUse` → `PostToolUse` → ítem `tool` `running` → `done` con `output`.
- Petición secundaria del tap (solo texto, no la última) se quita al `Stop` y se manda `snapshot`.
- Aprobación: `onDecision(pending)` → ítem `approval` `pending`; `approve` → `200` y `resolved`; segundo `approve` → `409`.
- Sin tap: `streaming: false` en el `snapshot`; la primera línea del tap manda un `snapshot` con `streaming: true`.
- Transcripción que falla, o sesión sin `transcriptPath` → `historyUnavailable: true` (sin buscar por cwd).
- Carrera tap/hooks: líneas del tap de una sesión sin mapear se guardan y se reproducen al llegar `onSessionMapped`; pasados 30 s (temporizador falso) o 500 líneas, se descartan.
- Chat abierto antes de la sesión: `snapshot` vacío con `historyUnavailable: true` y luego un `snapshot` con el historial cuando se mapea.
- Herramienta por id: `PreToolUse` antes que el bloque del tap y al revés → un solo ítem; `input`/`output` de más de 4 KB truncados.
- `viewers()` alimenta `setViewerCheck`: con un chat abierto devuelve verdadero; al cerrarlo, falso.

## Out of Scope

- Fuente Codex (`codexRollout.ts`, `source: "codex"`): historia 16 (aquí `codex` se rechaza con `4415`).
- Pantalla de chat, editor, tarjeta de aprobación y selector en la app: historias 14 y 15.
- Cambios al proxy o a los hooks (`tap/*`, `hooks.ts`, scripts): historias 11 y 12; si la comprobación de `docs/herdr/hooks.md` exige ajustar `hooks.ts`, anotarlo en el tracker en lugar de rediseñar.
- Elegir y guardar una `permission_suggestion` como regla permanente: el contrato solo lleva `allow`/`deny`; las sugerencias se muestran como información (decisión 19).
- Mandar al teléfono una aprobación levantada antes de abrir el chat o con el agente en modo Terminal: se queda en la terminal (decisión 20).
- Chat para agentes que no sean Claude Code o Codex: fuera del alcance.
- Notificaciones push: historias 17 y 18.

## Acceptance Criteria

- [ ] Dentro de `apps/herdr-gateway`: `vp test run src/chat/session.test.ts src/chat/claudeTranscript.test.ts src/chat/tapSocket.test.ts` pasa.
- [ ] Dentro de `apps/herdr-gateway` y de `packages/herdr`: `vp run typecheck` pasa.
- [ ] Las rutas de `routes/chat.ts` ya no responden `501`; `main.ts` crea el socket del tap (0700/0600) y lo cierra al salir.
- [ ] Con el gateway, el tap y los hooks instalados en `mylinux`: abrir el WS de chat de un Claude Code real (`websocat` o un `node -e` con `ws`), mandar un prompt con `POST …/chat/send` y ver `item` del usuario, `delta` palabra por palabra y `turnEnded`; anotar el resultado en el tracker.
- [ ] `node scripts/herdr/check-patch-surface.ts` sale con código 0.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
