---
status: not_started
feature: "herdr-core-mobile"
story: "14"
layers:
  - "frontend"
dependencies:
  - "story-09"
  - "story-13"
---

# Story 14: App, pantalla de chat

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code cuya app móvil abre en Herdr y muestra la terminal de cada agente (historia 9). El gateway ya expone el chat de un agente de Claude Code como ítems (`WS /v1/agents/:address/chat`, historia 13). Esta historia hace la pantalla de chat en la app: un hook que mantiene los ítems en vivo, y una lista propia con los componentes de presentación de T3 (`HerdrChatFeed`, el camino planeado). Antes, una comprobación acotada decide si `ThreadFeed` de T3 se puede reutilizar tal cual; con cualquier falla se queda `HerdrChatFeed`. Es una historia de riesgo con **regla de salida aprobada de antemano**: se documenta qué camino se tomó y por qué.

## Dependencies

- Depends on: Story 09 — App, terminal (`HerdrAgentRoute`, que la historia 15 envolverá con el selector Chat/Terminal; `createGatewayClient` con `wsUrl`/`postJson` de la historia 7).
- Depends on: Story 13 — Gateway, sesión de chat (`WS /v1/agents/:address/chat`, `POST …/chat/send|interrupt|approve`; tipos `ChatItem`, `ChatServerMessage`, `ChatSource`, `ChatApproveRequest`, `CHAT_CLOSE_CODES` en `@herdr/core-client/gateway-api`).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Cómo se alimenta el chat de Claude Code con los mismos agentes del escritorio?** → **Decision:** dos fuentes oficiales, con el Claude Code real intacto en su terminal:
   - **Proxy `herdr-tap`** vía `ANTHROPIC_BASE_URL`: texto y herramientas palabra por palabra, correlacionados con el header `x-claude-code-session-id`.
   - **Hooks de Claude Code**: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`, `Notification`, `Stop`, `SessionEnd`.
   - **Transcripción** (`~/.claude/projects/<proyecto>/<session-id>.jsonl`): solo para el historial al abrir el chat, porque su formato es interno y puede cambiar.
   - **Envío desde el teléfono**: `terminal.send` de herdr-core, sobre la misma terminal.
4. **Q: ¿Cuándo va el chat?** → **Decision:** justo después de la base (olas 3 a 6 del breakdown). Las historias 11 y 12 comprueban primero, con Claude real, que el proxy y los hooks funcionan.

## Affected Files

### Frontend

- [ ] Pattern to follow: `apps/mobile/src/features/threads/ThreadFeed.tsx` (`export const ThreadFeed`, props `ThreadFeedProps` en la línea ~343: `feed: ReadonlyArray<ThreadFeedEntry>` más `environmentId`, `threadId`, `detailRef`, `threadTitle`, `contentPresentation`, `latestRun`, `listRef`, `freeze`, `contentInsetEndAdjustment`, `keyboardOffset`, `queuedMessages` y más; además lee átomos de la app: `useThreadSelection`, `assetEnvironment`, `projectEnvironment`, `serverEnvironment`, `environmentThreadShells`) y `apps/mobile/src/features/threads/ThreadDetailScreen.tsx` (cómo se monta). `ThreadFeedEntry` está en `apps/mobile/src/lib/threadActivity.ts:192`. Para la lista propia: `SelectableMarkdownText` (`apps/mobile/src/native/SelectableMarkdownText.tsx`, markdown con bloques de código), `ThreadDisclosureChevron` y `ShimmeringWorkContent` (`apps/mobile/src/features/threads/thread-work-log.tsx`), `AppText`, `AppSymbol`.
- [ ] Conventions to follow: estado con átomos de `@effect/atom-react` o hooks de React con estado local donde baste; `PascalCase.tsx` para componentes, `camelCase.ts` para lógica; estilos con clases Uniwind y `useThemeColor`; listas largas con el mismo virtualizador que usa `ThreadFeed` (LegendList) o `FlatList`; pruebas `*.test.ts` junto al archivo; sin animaciones que repinten continuamente.
- [ ] `apps/mobile/src/features/herdr/chat/useHerdrChat.ts` — nuevo.
- [ ] `apps/mobile/src/features/herdr/chat/herdrFeedEntries.ts` — nuevo.
- [ ] `apps/mobile/src/features/herdr/chat/HerdrChatScreen.tsx` — nuevo.
- [ ] `apps/mobile/src/features/herdr/chat/HerdrChatFeed.tsx` — nuevo: lista propia (camino planeado; solo se omite si la comprobación del Step 4 pasa).
- [ ] Pruebas: `apps/mobile/src/features/herdr/chat/useHerdrChat.test.ts`, `apps/mobile/src/features/herdr/chat/herdrFeedEntries.test.ts`.

## Implementation Steps

### Step 1: contrato

Del API Contract (binding):

**`WS /v1/agents/:address/chat`** → primer mensaje `{ "type": "snapshot", "source": "claude" | "codex", "items": [ChatItem], "historyUnavailable": false, "streaming": true }`; luego `{ "type": "item", "item": ChatItem }` (alta o reemplazo por `id`), `{ "type": "delta", "id": "…", "text": "…" }` y `{ "type": "turnEnded" }`.

`ChatItem` es uno de (el gateway trunca `input` y `output` de herramientas a 4 KB):

- `{ "kind": "user", "id", "text", "atMs" }`
- `{ "kind": "assistant", "id", "text", "streaming": boolean }`
- `{ "kind": "tool", "id", "name", "input": object, "status": "running" | "done" | "error", "output": string | null }`
- `{ "kind": "approval", "id": requestId, "tool": string, "input": object, "suggestions": [...], "questions": [...] | null, "state": "pending" | "resolved" | "expired" }`

**`POST /v1/agents/:address/chat/send`** `{ "text": "…" }`. **`/interrupt`** `{}`. **`/approve`** `{ "requestId": "…", "decision": "allow" | "deny", "answers": { … } | null }` → `409 {"error":{"kind":"not_pending"}}` si ya se resolvió o venció.

Cierres del WS: `4404` la terminal no existe, `4415` agente sin chat.

### Step 2: `chat/useHerdrChat.ts`

- Reducer puro exportado `applyChatMessage(state: HerdrChatState, m: ChatServerMessage): HerdrChatState` con `HerdrChatState = { source: ChatSource | null; items: ChatItem[]; historyUnavailable: boolean; streaming: boolean; turnActive: boolean }`: `snapshot` reemplaza todo; `item` hace alta o reemplazo por `id` (conserva la posición si ya existía); `delta` anexa `text` al ítem `assistant` con ese `id` (si no existe, lo crea con `streaming: true`); `turnEnded` pone `turnActive: false` y todos los `assistant` en `streaming: false`. Un ítem `user` nuevo o un `assistant` con `streaming: true` pone `turnActive: true`.
- `useHerdrChat(hostId, address)` → `{ state; connection: "connecting" | "live" | "reconnecting" | "unsupported" | "gone"; send(text): Promise<void>; interrupt(): Promise<void>; approve(requestId, decision, answers): Promise<"ok" | "not_pending"> }`. WebSocket a `wsUrl("/v1/agents/" + encodeURIComponent(address) + "/chat")`, reconexión con espera creciente 1–30 s (cada reconexión recibe un `snapshot` nuevo); `4404` → `gone`, `4415` → `unsupported`, sin reconectar. Agrupar `delta` por cuadro para no renderizar por token. `approve` traduce `409 not_pending` a `"not_pending"`. (La historia 15 usa `send`, `interrupt` y `approve` desde la UI.)
- `useHerdrChat.test.ts`: `applyChatMessage` para `snapshot`, alta y reemplazo de `item`, `delta` sobre ítem existente y nuevo, `turnEnded`, y `snapshot` que reemplaza (caso de quitar ítems del gateway).

### Step 3: `chat/herdrFeedEntries.ts`

`toThreadFeedEntries(items: ChatItem[], ids: { environmentId: string; threadId: string }): ThreadFeedEntry[]` con identificadores sintéticos `environmentId = "herdr:<hostId>"` y `threadId = <address>`:

- `user`/`assistant` → `{ type: "message", id, createdAt, message: ThreadFeedMessage }` con `role`, `text`, `streaming`, `attachments: []`, `runId: null`, `sourceThreadId`, `createdAt`/`updatedAt` ISO (de `atMs`, o de la posición si no hay), y el `visibility` que use un mensaje normal.
- `tool` → la entrada de actividad que use `ThreadFeed` para herramientas (`activity`/`activity-group`); `ThreadFeedActivity` exige `projectedItem: OrchestrationV2ProjectedTurnItem` (de `@t3tools/contracts`), así que hay que construir uno mínimo válido o concluir que no se puede (ver regla de salida).
- `approval` → no va al feed (la tarjeta la pinta la historia 15 fuera de la lista).
- Los tipos de ids de T3 pueden ser _branded_ (`MessageId`, `ThreadId`, `RunId`): usar sus constructores de `@t3tools/contracts` si existen, nunca `as any`.
- `herdrFeedEntries.test.ts`: orden, ids estables, `streaming` y texto, y que `approval` no aparece.

### Step 4: comprobación acotada de `ThreadFeed` y `HerdrChatScreen.tsx`

`export function HerdrChatScreen(props: { hostId: string; address: string })`.

**Comprobación (al inicio de la historia, antes de construir la pantalla; como máximo una iteración de prueba):** montar `ThreadFeed` de T3 con `feed = toThreadFeedEntries(fixture)`, donde el fixture tiene una entrada de usuario, una de asistente y una de herramienta, `environmentId = "herdr:<hostId>"`, `threadId = <address>` y los demás props con valores neutros. `ThreadFeed` se usa **solo si se cumplen las tres**:

1. Renderiza las tres entradas del fixture (usuario, asistente y herramienta) en Android (montándolo de forma temporal en una pantalla herdr, cambio local que se revierte antes del commit; nunca en un archivo de T3).
2. Lo hace usando solo providers y átomos que ya monta toda la app (los de `App.tsx`/`RootStackLayout`); no hace falta envolverlo en ningún provider nuevo ni simular un entorno T3 conectado. Sus átomos (`useThreadSelection`, `assetEnvironment`, `projectEnvironment`, `serverEnvironment`, `environmentThreadShells`) no pueden exigir un entorno real.
3. No requiere editar ningún archivo de T3 fuera de `docs/herdr/patch-surface.txt`, ni construir un `projectedItem` (`OrchestrationV2ProjectedTurnItem`) falso más allá de sus campos requeridos con valores honestos.
   **Cualquier** provider o átomo requerido que falte, cualquier error en tiempo de ejecución o cualquier entrada que no aparezca = se usa `HerdrChatFeed`. No se intenta arreglarlo.

**`HerdrChatFeed` (camino planeado):** lista propia con los componentes de presentación de T3:

- `user` → burbuja con el texto (`AppText`, alineada como los mensajes de usuario de T3).
- `assistant` → `SelectableMarkdownText` (markdown y bloques de código); mientras `streaming`, sin animaciones continuas (como mucho el indicador estático o `ShimmeringWorkContent` si no repinta en bucle).
- `tool` → fila plegada con `AppSymbol`, nombre de la herramienta y un resumen del `input` (por ejemplo `command` o `file_path`, una línea, máximo 200 caracteres), estado (`running`/`done`/`error`) y, al desplegar con `ThreadDisclosureChevron`, `input` (JSON con sangría, armado con un `JSON.stringify` en `try`; si falla, `String(input)`) y `output` en monoespaciado y seleccionables. El gateway ya los trunca a 4 KB (historia 13); la app no vuelve a truncar.
- Lista virtualizada que sigue el final mientras llega texto, salvo que el usuario haya subido.
- **Documentar** en `## Implementation Notes` del tracker (sección `story-14`) qué camino se tomó y cuál de los tres criterios falló, si fue el caso.
- Encabezado del chat: si `historyUnavailable`, una línea "Historial no disponible"; si `streaming` es falso, una línea **"Sin streaming en vivo"** (sesiones con `claude-glm`, etc.); si `connection` es `reconnecting`, "Reconectando…"; `gone` → "La terminal ya no existe"; `unsupported` → "Este agente no tiene chat".
- Espacio inferior reservado para el editor (historia 15): `HerdrChatScreen` acepta `props.footer?: ReactNode` y `props.renderApproval?: (item: ChatApprovalItem) => ReactNode` que la historia 15 llena; aquí quedan vacíos.

## Out of Scope

- Editor para escribir y detener, tarjeta de aprobación y preguntas, y el selector Chat/Terminal en `HerdrAgentRoute`: historia 15 (esta historia no edita `HerdrAgentRoute.tsx`).
- Chat de Codex en el gateway: historia 16 (la pantalla muestra lo que llegue con `source: "codex"` sin código especial).
- Modificar `ThreadFeed.tsx`, `threadActivity.ts` u otros archivos de T3 que no estén en `docs/herdr/patch-surface.txt`: prohibido; para eso existe la regla de salida.
- Chat para agentes que no sean Claude Code o Codex: fuera del alcance.

## Acceptance Criteria

- [ ] Dentro de `apps/mobile`: `vp test run src/features/herdr/chat/useHerdrChat.test.ts src/features/herdr/chat/herdrFeedEntries.test.ts` pasa.
- [ ] Dentro de `apps/mobile`: `vp run typecheck` pasa; ningún `as any` ni `@ts-expect-error` en `src/features/herdr/chat/`.
- [ ] Ningún archivo fuera de `apps/mobile/src/features/herdr/` cambia; `node scripts/herdr/check-patch-surface.ts` sale con código 0.
- [ ] El camino tomado (`ThreadFeed` o `HerdrChatFeed`) y su motivo están anotados en el tracker (`story-14`).
- [ ] El resultado de la comprobación del Step 4 (pasa / qué criterio falló) está anotado en el tracker. La verificación nativa de la pantalla completa con un Claude Code real se hace en la historia 15, que monta `HerdrChatScreen` en `HerdrAgentRoute`; aquí no se monta en ninguna ruta.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
