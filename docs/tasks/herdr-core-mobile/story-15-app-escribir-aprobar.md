---
status: not_started
feature: "herdr-core-mobile"
story: "15"
layers:
  - "frontend"
dependencies:
  - "story-14"
---

# Story 15: App, escribir y aprobar

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code cuya app móvil abre en Herdr, muestra la terminal de cada agente (historia 9) y ya tiene una pantalla de chat que lee los ítems en vivo del gateway (historia 14). Esta historia completa el chat: un editor para mandar mensajes a la misma terminal y detener el turno, una tarjeta para aprobar o denegar permisos y responder preguntas (`AskUserQuestion`) desde el teléfono, y el selector Chat / Terminal en cada agente (Chat por defecto para Claude Code y Codex; los demás agentes solo tienen terminal).

## Dependencies

- Depends on: Story 14 — App, pantalla de chat (`HerdrChatScreen({ hostId, address, footer?, renderApproval? })`; `useHerdrChat(hostId, address)` con `state` (`items`, `turnActive`, `streaming`, `source`), `send(text)`, `interrupt()`, `approve(requestId, decision, answers)` → `"ok" | "not_pending"`; el camino tomado —`ThreadFeed` o `HerdrChatFeed`— está anotado en el tracker, leerlo).
- Usa de la historia 9: `HerdrAgentRoute.tsx` con la paginación de paneles y `HerdrTerminalPane`; de la historia 7: `useHerdrTerminal(hostId, address)` (campo `app` de la terminal).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Cómo se alimenta el chat de Claude Code con los mismos agentes del escritorio?** → **Decision:** dos fuentes oficiales, con el Claude Code real intacto en su terminal:
   - **Proxy `herdr-tap`** vía `ANTHROPIC_BASE_URL`: texto y herramientas palabra por palabra, correlacionados con el header `x-claude-code-session-id`.
   - **Hooks de Claude Code**: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`, `Notification`, `Stop`, `SessionEnd`.
   - **Transcripción** (`~/.claude/projects/<proyecto>/<session-id>.jsonl`): solo para el historial al abrir el chat, porque su formato es interno y puede cambiar.
   - **Envío desde el teléfono**: `terminal.send` de herdr-core, sobre la misma terminal.
4. **(plan) Aprobaciones sin bloquear al escritorio.** El hook solo espera la decisión del teléfono si hay al menos una app con el chat de ese agente abierto en ese momento. Si no, responde de inmediato sin decidir y Claude sigue su flujo normal en la terminal. La espera máxima es 120 s; al vencer, también sigue el flujo normal. Una espera también se libera sin decidir al llegar, para la misma sesión, cualquiera de estos eventos: `PostToolUse`, `PostToolUseFailure`, `PermissionDenied`, `Stop`, `UserPromptSubmit` o `SessionEnd` (nunca `Notification`). Respaldo aprobado: si la comprobación con Claude real muestra que responder en la terminal no termina la espera del hook, la espera máxima baja a 30 s y se anota en `docs/herdr/hooks.md` y en el tracker.
5. **(plan) Sugerencias de permiso solo informativas.** Las `permission_suggestions` de Claude Code se muestran en la tarjeta solo como información; desde el teléfono solo se responde `allow` o `deny`, porque el contrato de `/approve` no tiene cómo elegir una sugerencia.
6. **(plan) Aprobaciones solo con el chat abierto.** Una aprobación o pregunta llega al teléfono solo mientras el chat de ese agente está abierto en el teléfono. Una petición que se levantó antes de abrirlo, o mientras se ve en modo Terminal, se queda en la terminal. Es el comportamiento buscado y se prueba.

## Affected Files

### Frontend

- [ ] Pattern to follow: `apps/mobile/src/features/threads/ThreadComposer.tsx` (editor, botón de enviar y de detener) sobre `ComposerEditor` (`apps/mobile/src/components/ComposerEditor.tsx`, envuelve el módulo nativo `t3-composer-editor`; acepta `draftKey`, `environmentId?` y los props nativos); `apps/mobile/src/features/threads/PendingApprovalCard.tsx` y `PendingUserInputCard.tsx` (aspecto de la tarjeta de permiso y de preguntas); `SegmentedControl` de `apps/mobile/src/components/ui/SegmentedControl.tsx` para el selector.
- [ ] Conventions to follow: estado con átomos de `@effect/atom-react` o hooks de React con estado local donde baste; `PascalCase.tsx` para componentes; estilos con clases Uniwind y `useThemeColor`; símbolos con `AppSymbol`; texto con `AppText`; `Button`, `Chip` de `components/ui`; pruebas `*.test.ts` junto al archivo para la lógica pura.
- [ ] `apps/mobile/src/features/herdr/chat/HerdrChatComposer.tsx` — nuevo.
- [ ] `apps/mobile/src/features/herdr/chat/HerdrApprovalCard.tsx` — nuevo.
- [ ] `apps/mobile/src/features/herdr/chat/approvalAnswers.ts` — nuevo: lógica pura de respuestas.
- [ ] `apps/mobile/src/features/herdr/HerdrAgentRoute.tsx` — selector Chat/Terminal.
- [ ] `apps/mobile/src/features/herdr/chat/HerdrChatScreen.tsx` — conectar editor y tarjeta.
- [ ] Prueba: `apps/mobile/src/features/herdr/chat/approvalAnswers.test.ts`.

## Implementation Steps

### Step 1: contrato

Del API Contract (binding): **`POST /v1/agents/:address/chat/send`** `{ "text": "…" }`. **`/interrupt`** `{}`. **`/approve`** `{ "requestId": "…", "decision": "allow" | "deny", "answers": { … } | null }` → `409 {"error":{"kind":"not_pending"}}` si ya se resolvió o venció.

Ítem de aprobación: `{ "kind": "approval", "id": requestId, "tool": string, "input": object, "suggestions": [...], "questions": [...] | null, "state": "pending" | "resolved" | "expired" }`. Para `AskUserQuestion`, `questions` es el arreglo original de Claude Code: `[{ question: string, header?: string, options: [{ label: string, description?: string }], multiSelect?: boolean }]`; `answers` tiene como clave el texto de cada `question` y como valor la etiqueta elegida (varias separadas por `", "`), como espera el gateway (historia 12).

Validation & Error Handling (binding): aprobación respondida en la terminal antes que en el teléfono: la tarjeta pasa a `resolved` y el botón ya no envía nada (`409`).

### Step 2: `HerdrChatComposer.tsx`

- `HerdrChatComposer({ turnActive, onSend(text): Promise<void>, onInterrupt(): Promise<void>, draftKey })` con `ComposerEditor` (`draftKey = "herdr:" + hostId + ":" + address`), sin adjuntos, menciones ni comandos de T3.
- Botón "Enviar" (deshabilitado con texto vacío o mientras se envía) → `send(text)` y limpiar el borrador al confirmar; error → mensaje bajo el editor y el texto se conserva. Lo enviado aparece en la terminal del escritorio (el gateway usa `terminal.send`) y vuelve al chat por el hook `UserPromptSubmit`; no agregar el ítem localmente.
- Mientras `turnActive`, botón "Detener" → `interrupt()`.

### Step 3: `HerdrApprovalCard.tsx` y `approvalAnswers.ts`

- `approvalAnswers.ts` (puro): `buildAnswers(questions, selected: Record<number, string[]>): Record<string, string>` y `canSubmitAnswers(questions, selected): boolean` (cada pregunta con al menos una opción; `multiSelect` permite varias). La forma de `answers` (clave = texto de la pregunta, valor = etiquetas unidas con `", "`) es la única que acepta Claude Code, sin ids estables: si dos preguntas tienen el mismo texto, o una etiqueta elegida contiene `", "`, la respuesta sería ambigua → `canSubmitAnswers` devuelve `false` y la tarjeta dice "Responde esta pregunta en la terminal" (solo queda "Denegar").
- Tarjeta de **permiso** (`questions === null`): título con `tool`, resumen del `input` (por ejemplo `command` de `Bash`, `file_path` de `Edit`/`Write`, o JSON compacto), botones **"Permitir"** (`approve(id, "allow", null)`) y **"Denegar"** (`approve(id, "deny", null)`). Si `suggestions` no está vacío, se listan como texto informativo ("Claude sugiere: …") y nunca como botones (decisión 19).
- Tarjeta de **preguntas** (`questions` no nulo): por pregunta, `header` y `question`, opciones como `Chip` seleccionables (una o varias según `multiSelect`); botón "Responder" (habilitado con `canSubmitAnswers`) → `approve(id, "allow", buildAnswers(...))`; botón "Denegar" → `approve(id, "deny", null)`.
- Estados: `pending` → botones activos; al tocar uno, deshabilitar mientras responde; respuesta `"not_pending"` (409) o `state` `resolved` → tarjeta en solo lectura "Respondido" y ningún botón envía nada; `expired` → "Venció: sigue en la terminal".
- Recordatorio en la tarjeta: la espera tiene un máximo (120 s, o 30 s si la historia 12 aplicó el respaldo de la decisión 17; leer el tracker); pasado ese tiempo Claude sigue su flujo normal en la terminal.

### Step 4: `HerdrChatScreen.tsx`

Pasar `footer={<HerdrChatComposer … />}` y `renderApproval={(item) => <HerdrApprovalCard item={item} onApprove={approve} />}`. Las aprobaciones `pending` se muestran fijas sobre el editor (no se pierden al desplazar); las ya resueltas pueden quedar en la lista o desaparecer según el camino de la historia 14.

### Step 5: selector en `HerdrAgentRoute.tsx`

- Si la terminal del panel visible tiene `app === "claude-code"` o `app === "codex"`: `SegmentedControl` con **Chat** y **Terminal** en el encabezado, **Chat por defecto**; Chat → `HerdrChatScreen`, Terminal → `HerdrTerminalPane` (de la historia 9). El modo elegido se recuerda por terminal mientras la app está abierta (estado local o átomo, no persistido).
- Cualquier otro `app` (o `null`): solo terminal, sin selector.
- Al cambiar a Terminal, el chat se desmonta (cierra su WebSocket, y con eso deja de contar como visor: el hook no espera al teléfono, decisiones 17 y 20); al volver a Chat se reabre con `snapshot`. Una aprobación levantada mientras se veía la Terminal, o antes de abrir el chat, se queda en la terminal y no aparece como tarjeta pendiente (decisión 20); es el comportamiento buscado.
- Si el `snapshot` del chat trae `sessionUncertain: true` (Codex, historia 16), mostrar una línea "Sesión de Codex sin confirmar" bajo el encabezado; para eso, extender `HerdrChatState` de la historia 14 con ese campo opcional.
- No cambiar la paginación de paneles ni el `HerdrFilesInspector` que ya monta la pantalla.

## Out of Scope

- El modelo de chat del gateway y sus rutas: historia 13; Codex en el gateway: historia 16.
- Adjuntos, menciones, comandos `/` o selector de modelo del editor de T3.
- Guardar una `permission_suggestion` como regla permanente: el contrato no lo permite.
- Chat para agentes que no sean Claude Code o Codex: se ven como terminal (fuera del alcance).
- Notificaciones push: historias 17 y 18.
- Editar `ThreadComposer.tsx`, `ComposerEditor.tsx`, `PendingApprovalCard.tsx` u otros archivos de T3.

## Acceptance Criteria

- [ ] Dentro de `apps/mobile`: `vp test run src/features/herdr/chat/approvalAnswers.test.ts` pasa (una opción, varias con `multiSelect`, pregunta sin respuesta bloquea el envío, dos preguntas con el mismo texto o una etiqueta con `", "` bloquean el envío).
- [ ] Dentro de `apps/mobile`: `vp run typecheck` pasa.
- [ ] Ningún archivo fuera de `apps/mobile/src/features/herdr/` cambia; `node scripts/herdr/check-patch-surface.ts` sale con código 0.
- [ ] Verificación en Android en `mylinux` con un Claude Code real: abrir el agente abre en Chat; escribir un mensaje lo hace aparecer en la terminal del escritorio y la respuesta llega palabra por palabra; cuando Claude pide permiso, tocar "Permitir" hace que la terminal siga sin intervención; responder primero en la terminal deja la tarjeta en "Respondido"; "Detener" interrumpe el turno; cambiar a Terminal muestra la terminal ajustada; **decisión 20:** con el agente en Terminal, provocar un permiso y volver a Chat: no aparece tarjeta pendiente y la petición se responde en la terminal; el historial reciente se ve al abrir y las herramientas aparecen plegadas (verificación nativa que la historia 14 dejó para aquí). Un agente que no es Claude Code ni Codex abre directo en Terminal sin selector. Anotar el resultado en el tracker.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
