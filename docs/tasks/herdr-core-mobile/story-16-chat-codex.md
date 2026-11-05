---
status: not_started
feature: "herdr-core-mobile"
story: "16"
layers:
  - "backend"
dependencies:
  - "story-13"
---

# Story 16: Chat de Codex

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code que agrega una app móvil para herdr-core con un chat nativo para los agentes de las terminales de herdr. El gateway ya tiene el chat de Claude Code (historia 13). Esta historia agrega Codex: primero **comprueba** si Codex puede dar respuestas palabra por palabra a través de un `app-server` compartido sin romper la regla de herdr de que Codex corre con `--no-daemon`; luego implementa la **base garantizada**, que lee el registro de sesión (`rollout-*.jsonl`) de Codex y lo traduce a ítems de chat por bloques; y solo si la comprobación sale positiva, agrega palabra por palabra. Es una historia de riesgo: la base por bloques queda pase lo que pase.

## Dependencies

- Depends on: Story 13 — Gateway, sesión de chat (`apps/herdr-gateway/src/chat/session.ts` con `createChatHub` y la sesión por terminal; `routes/chat.ts`, que hoy rechaza `app !== "claude-code"` con `4415`; tipos `ChatItem`, `ChatServerMessage`, `ChatSource` en `@herdr/core-client/gateway-api`).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Codex?** → **Decision:** el chat de Codex entra al plan, con comprobación primero. Base garantizada: leer su registro de sesión (`~/.codex/sessions/**/rollout-*.jsonl`), compatible con `--no-daemon`, con respuestas por bloques. Palabra por palabra solo si se demuestra que una terminal de herdr con Codex y el gateway pueden compartir un `app-server` sin romper la regla de herdr de que Codex corre con `--no-daemon`. Si no, se queda la base y se documenta por qué.
4. **(plan) Gateway sin Effect.** El gateway y `packages/herdr` usan TypeScript plano con Promises y APIs de Node, no Effect, para no acoplarse a la evolución de las librerías del servidor T3. `packages/herdr` no importa nada de `@t3tools/*`.

## Affected Files

### Backend

- [ ] Pattern to follow: `packages/effect-codex-app-server/` (solo como referencia del protocolo del `app-server` de Codex; no importarlo: es Effect). El modelo de ítems de `chat/session.ts` y el lector tolerante de `chat/claudeTranscript.ts` (historia 13).
- [ ] Conventions to follow: TypeScript plano con `node:fs`, `node:path` (decisión 14); lector tolerante (registros desconocidos se ignoran); errores `{ "error": { "kind", "message" } }`; sin `enum`/`namespace`; pruebas `*.test.ts` con fixtures grabados.
- [ ] `apps/herdr-gateway/src/chat/codexRollout.ts` — nuevo: ubicar y traducir el rollout.
- [ ] `apps/herdr-gateway/src/chat/session.ts` — fuente Codex.
- [ ] `apps/herdr-gateway/src/routes/chat.ts` — aceptar `app === "codex"` (cambio de una condición).
- [ ] `packages/herdr/src/gateway-api.ts` — campo opcional `sessionUncertain?: boolean` en el mensaje `snapshot` del chat (ninguna otra historia de la ola 5 edita este archivo).
- [ ] `docs/herdr/codex.md` — nuevo: resultado de la comprobación.
- [ ] Pruebas: `apps/herdr-gateway/src/chat/codexRollout.test.ts` con `apps/herdr-gateway/src/chat/fixtures/codex-rollout.jsonl`.

## Implementation Steps

### Step 1: comprobación (criterio de aceptación)

En `mylinux` (Codex instalado; `~/.codex/sessions/` existe): documentar en `docs/herdr/codex.md` si `codex --remote` (o la opción equivalente de la versión instalada; revisar `codex --help` y `codex app-server --help`) contra un `app-server` compartido funciona **dentro de una terminal de herdr** sin violar la regla de herdr de que Codex corre con `--no-daemon`, y si el gateway podría suscribirse a los eventos de esa sesión para recibir texto palabra por palabra. Registrar: versión de Codex (0.166 en `mylinux`, con `--no-daemon`), comandos probados, qué pasó, y la conclusión. **Criterio de "viable" (binding), deben cumplirse los dos:** (1) Codex corre dentro de la terminal de herdr y sus comandos se ejecutan bajo hc (verificable con `hc ls`: los procesos que lanza aparecen bajo esa terminal, no en un `app-server` fuera de hc); y (2) el gateway recibe los deltas de los ítems sin que el usuario cambie cómo lanza Codex (sigue siendo `codex --no-daemon`, el comando de la pestaña nueva de la historia 9). Si falla cualquiera de los dos: **no viable**, queda solo la base y `codex.md` dice cuál falló. No cambiar cómo se lanza Codex en las terminales de herdr para hacer la prueba más allá de una terminal de prueba.

### Step 2: `chat/codexRollout.ts` (base garantizada)

- **Ubicar el archivo por proceso (principal):** `findCodexRolloutByProcess(foregroundPid: number, procRoot = "/proc"): Promise<string | null>`. Con `hc.call("terminal.get", { terminal: address })` (método de hc 0.1.18, en la prueba de contrato de la historia 3) se obtiene `foreground.pid` (líder del grupo de procesos en primer plano de la terminal). Desde ese PID, recorrer el árbol de procesos hacia abajo (hijos en `/proc/<pid>/task/<pid>/children`, o leyendo `PPid` de `/proc/*/status`) hasta encontrar un proceso cuyo `comm` o `cmdline` sea `codex` (el propio PID puede serlo); en él, leer los enlaces de `/proc/<pid>/fd/*` (`readlink`) y tomar el que apunte a un `rollout-*.jsonl` bajo `~/.codex/sessions/`. Sesión segura: `sessionUncertain: false`.
- **Respaldo (solo si no hay proceso o no tiene el archivo abierto):** `findCodexRolloutByCwd(opts: { home: string; cwd: string; createdAfterMs: number })`: recorrer `~/.codex/sessions/YYYY/MM/DD/` de la fecha de `createdAfterMs` en adelante (como mucho 7 días), leer **solo la primera línea** (`session_meta`) de cada `rollout-*.jsonl` y elegir el más reciente con `payload.cwd === cwd` y `payload.timestamp` (o el `mtime`) posterior a `createdAfterMs` (`created_ms` de `terminal.get`). Lo encontrado así se marca `sessionUncertain: true` en el `snapshot` del chat.
- **Traducir:** `readCodexRollout(path, opts?: { limit?: 200 }): Promise<ChatItem[]>` y `applyCodexRecord(items, record): { items; changed: ChatItem[]; turnEnded: boolean }` (para el seguimiento en vivo). Registros (formato interno de Codex, puede cambiar; cada línea es `{ timestamp, type, payload }`):
  - `response_item` / `payload.type === "message"`: `role: "user"` → `ChatUserItem` con el texto de los bloques `input_text`, **saltando** los que empiezan con `<environment_context>`, `<user_instructions>` o similares inyectados por Codex; `role: "assistant"` → `ChatAssistantItem` con los bloques `output_text` (`streaming: false`); `role: "developer"` y `system` → se ignoran. `id` = `payload.id` o `timestamp + índice`.
  - `response_item` / `custom_tool_call` (`name`, `input`, `call_id`) y `function_call` (`name`, `arguments`, `call_id`) → `ChatToolItem` (`id = call_id`, `input` = objeto o `{ input }`/`JSON.parse(arguments)` en `try`, `status: "running"`).
  - `response_item` / `custom_tool_call_output` y `function_call_output` (`call_id`, `output`) → completan el `ChatToolItem` (`status: "done"`, `output` como texto, truncado a 4 KB).
  - `response_item` / `reasoning` → se ignora (no se muestra el razonamiento en la v1).
  - `event_msg` / `task_started` → turno abierto; `event_msg` / `task_complete` → turno cerrado (`turnEnded: true`); `event_msg` / `item_completed` → si trae un `AgentMessage` y no hubo `message` de asistente equivalente, sirve como respaldo del texto final. `token_count`, `turn_context`, `world_state` y desconocidos → se ignoran.
- **Seguimiento en vivo:** `followCodexRollout(path, onRecord): () => void` con `fs.watch` del archivo (y un respaldo de `fs.watchFile` cada 1 s), leyendo solo los bytes nuevos desde el último offset y procesando líneas completas; la función devuelta cierra ambos vigilantes. Si el archivo se acorta (truncado o reemplazado), volver a leerlo desde el inicio y mandar un `snapshot`. Los ítems se deduplican por `id`.

### Step 3: `chat/session.ts` (fuente Codex)

- Para una terminal con `app === "codex"`: `source: "codex"`, historial con `findCodexRolloutByProcess` (y, si no, el respaldo por cwd con `sessionUncertain: true`) + `readCodexRollout` (si no se encuentra o falla → `historyUnavailable: true`, y reintentar ubicar el archivo cada 5 s mientras haya visores —un solo temporizador por sesión, cancelado al irse el último visor—, porque Codex crea el rollout al empezar la primera sesión; si el respaldo por cwd encontró uno y luego aparece por proceso, se cambia al de proceso y se manda un `snapshot` nuevo), `streaming: false` (base por bloques), y en vivo `followCodexRollout` → `item` por cada ítem cambiado y `turnEnded` al cerrar el turno.
- Enviar e interrumpir ya van por la terminal (`terminal.send` / `terminal.interrupt` en `routes/chat.ts`), igual que Claude.
- Sin aprobaciones para Codex en la v1 (los permisos de Codex se responden en la terminal).
- Solo si la comprobación del Step 1 salió **viable**: agregar el palabra por palabra por esa vía (`delta` sobre el ítem `assistant` en curso, `streaming: true`), sin cambiar cómo herdr lanza Codex. Si no fue viable, no se agrega nada y `codex.md` lo explica.

### Step 4: `routes/chat.ts`

Cambiar la condición de `4415` para aceptar `app === "claude-code"` o `app === "codex"`.

### Step 5: pruebas (`codexRollout.test.ts`)

Fixture grabado a mano (`chat/fixtures/codex-rollout.jsonl`) con: `session_meta` (con `cwd`), `turn_context`, mensaje `developer`, mensaje `user` con `<environment_context>` y otro real, `reasoning`, `custom_tool_call` + `custom_tool_call_output`, `function_call` + `function_call_output`, mensaje `assistant`, `event_msg` `task_started`/`item_completed`/`task_complete`, una línea rota y un tipo desconocido. Verificar los ítems resultantes, el orden, que se ignoran los inyectados y los desconocidos, `turnEnded`, `findCodexRolloutByProcess` sobre un `/proc` falso en una carpeta temporal (PID de la terminal → hijo `codex` → `fd/5` enlazado a un `rollout-*.jsonl`; y un caso sin `codex`), y `findCodexRolloutByCwd` sobre una carpeta temporal con tres rollouts (otro `cwd`, uno anterior a la terminal, el correcto). `followCodexRollout` se detiene (sin vigilantes vivos) al llamar la función devuelta.

## Out of Scope

- Cambiar cómo herdr lanza Codex (`--no-daemon`) o instalar un `app-server` permanente: solo se comprueba.
- Aprobaciones o preguntas de Codex desde el teléfono: no están en la v1.
- Mostrar el razonamiento (`reasoning`) en el chat.
- La app (pantalla de chat, selector): historias 14 y 15; ya funcionan con `source: "codex"`.
- Chat para agentes que no sean Claude Code o Codex: fuera del alcance.

## Acceptance Criteria

- [ ] Dentro de `apps/herdr-gateway`: `vp test run src/chat/codexRollout.test.ts src/chat/session.test.ts` pasa (las pruebas de Claude de la historia 13 siguen pasando).
- [ ] Dentro de `apps/herdr-gateway`: `vp run typecheck` pasa.
- [ ] **Comprobación** del Step 1 hecha y documentada en `docs/herdr/codex.md` con versión, comandos, resultado y conclusión (viable / no viable y por qué).
- [ ] En `mylinux`, el WS de chat de una terminal de Codex real devuelve un `snapshot` con `source: "codex"` y sus mensajes, y un mensaje nuevo enviado con `POST …/chat/send` aparece como ítems por bloques al terminar. Anotar el resultado en el tracker.
- [ ] `node scripts/herdr/check-patch-surface.ts` sale con código 0.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
