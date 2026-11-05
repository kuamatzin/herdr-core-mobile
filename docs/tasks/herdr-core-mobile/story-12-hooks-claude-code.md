---
status: not_started
feature: "herdr-core-mobile"
story: "12"
layers:
  - "backend"
  - "other"
dependencies:
  - "story-04"
---

# Story 12: Hooks de Claude Code

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code que agrega una app móvil para herdr-core, con un chat nativo para los agentes de Claude Code que corren en terminales de herdr. Además del proxy `herdr-tap` (historia 11, en paralelo), el chat se alimenta de los **hooks oficiales de Claude Code**: prompts, herramientas, permisos, preguntas y fin de turno. Esta historia crea el script `herdr-hook`, su instalador que agrega entradas a `~/.claude/settings.json` sin pisar las existentes, y el receptor local en el gateway (`127.0.0.1:7789`) que mapea sesión ↔ terminal y retiene las decisiones de permiso y de preguntas para que la app pueda responderlas. También **comprueba con Claude real** cómo conviven el diálogo de permiso de la terminal y la espera del hook. Es una historia de riesgo: si falla, la base sigue.

## Dependencies

- Depends on: Story 04 — Gateway base (`apps/herdr-gateway/src/main.ts`, `apps/herdr-gateway/src/http.ts` con `GatewayContext`, `sendJson`, `sendError`).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Cómo se alimenta el chat de Claude Code con los mismos agentes del escritorio?** → **Decision:** dos fuentes oficiales, con el Claude Code real intacto en su terminal:
   - **Proxy `herdr-tap`** vía `ANTHROPIC_BASE_URL`: texto y herramientas palabra por palabra, correlacionados con el header `x-claude-code-session-id`.
   - **Hooks de Claude Code**: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`, `Notification`, `Stop`, `SessionEnd`.
   - **Transcripción** (`~/.claude/projects/<proyecto>/<session-id>.jsonl`): solo para el historial al abrir el chat, porque su formato es interno y puede cambiar.
   - **Envío desde el teléfono**: `terminal.send` de herdr-core, sobre la misma terminal.
4. **Q: ¿Cuándo va el chat?** → **Decision:** justo después de la base (olas 3 a 6 del breakdown). Las historias 11 y 12 comprueban primero, con Claude real, que el proxy y los hooks funcionan.
5. **(plan) Gateway sin Effect.** El gateway y `packages/herdr` usan TypeScript plano con Promises y APIs de Node, no Effect, para no acoplarse a la evolución de las librerías del servidor T3. `packages/herdr` no importa nada de `@t3tools/*`.
6. **(plan) Aprobaciones sin bloquear al escritorio.** El hook solo espera la decisión del teléfono si hay al menos una app con el chat de ese agente abierto en ese momento. Si no, responde de inmediato sin decidir y Claude sigue su flujo normal en la terminal. La espera máxima es 120 s; al vencer, también sigue el flujo normal. Una espera también se libera sin decidir al llegar, para la misma sesión, cualquiera de estos eventos: `PostToolUse`, `PostToolUseFailure`, `PermissionDenied`, `Stop`, `UserPromptSubmit` o `SessionEnd` (nunca `Notification`). Respaldo aprobado: si la comprobación con Claude real muestra que responder en la terminal no termina la espera del hook, la espera máxima baja a 30 s y se anota en `docs/herdr/hooks.md` y en el tracker.
7. **(plan) Sugerencias de permiso solo informativas.** Las `permission_suggestions` de Claude Code se muestran en la tarjeta solo como información; desde el teléfono solo se responde `allow` o `deny`, porque el contrato de `/approve` no tiene cómo elegir una sugerencia.
8. **(plan) Aprobaciones solo con el chat abierto.** Una aprobación o pregunta llega al teléfono solo mientras el chat de ese agente está abierto en el teléfono. Una petición que se levantó antes de abrirlo, o mientras se ve en modo Terminal, se queda en la terminal. Es el comportamiento buscado y se prueba.

Access Control (binding): el receptor de hooks escucha solo en `127.0.0.1`, nunca en la IP de Tailscale.

## Affected Files

### Backend

- [ ] Pattern to follow: el hook que ya instala herdr en `~/.claude/hooks/herdr-agent-state.sh`, como ejemplo de cómo leer stdin en `sh` y salir siempre con 0 (**no modificar ese archivo**). Hooks documentados de Claude Code: campos comunes `session_id`, `transcript_path`, `cwd`, `hook_event_name`; `UserPromptSubmit` trae `prompt`; `Stop` trae `last_assistant_message`; `PreToolUse`/`PostToolUse` traen `tool_name`, `tool_input` (y `tool_response` en `PostToolUse`); `PermissionRequest` trae `tool_name`, `tool_input` y `permission_suggestions`; la decisión de `PermissionRequest` va en `hookSpecificOutput.decision.behavior` = `allow` | `deny` (con `updatedInput` / `message`); `AskUserQuestion` se responde desde `PreToolUse` con `permissionDecision: "allow"` más `updatedInput` con las `questions` originales y un objeto `answers`; el timeout por defecto de un hook `command` es 600 s.
- [ ] Conventions to follow: TypeScript plano con `node:http`, `node:fs`, `node:crypto` (decisión 14); `install-hooks.ts` solo con módulos de Node (se ejecuta con `node` directo, sin build); errores `{ "error": { "kind", "message" } }`; sin `enum`/`namespace`; pruebas `*.test.ts` junto al archivo, con reloj/temporizadores inyectados (sin esperas reales de 120 s).
- [ ] `apps/herdr-gateway/scripts/herdr-hook.sh` — nuevo.
- [ ] `apps/herdr-gateway/scripts/install-hooks.ts` — nuevo.
- [ ] `apps/herdr-gateway/src/hooks.ts` — nuevo: receptor y `HookHub`.
- [ ] `apps/herdr-gateway/src/main.ts` — arrancar el receptor y ponerlo en el contexto.
- [ ] `apps/herdr-gateway/src/http.ts` — agregar `hooks: HookHub` a `GatewayContext` (ninguna otra historia de la ola 3 toca `main.ts` ni `http.ts`).
- [ ] `apps/herdr-gateway/tsconfig.json` — agregar `"scripts"` a `include` para que `vp run typecheck` cubra `install-hooks.ts`.
- [ ] `docs/herdr/hooks.md` — nuevo: resultado de la comprobación.
- [ ] Pruebas: `apps/herdr-gateway/src/hooks.test.ts`, `apps/herdr-gateway/scripts/install-hooks.test.ts`.

## Implementation Steps

### Step 1: contrato

Del API Contract: **Interno, solo `127.0.0.1:7789`:** `POST /v1/hooks` (headers `x-herdr-terminal`, `x-herdr-event`; cuerpo: el JSON del hook) → `200` con la salida JSON del hook, o `204` para no decidir.

Validation & Error Handling (binding): hook sin gateway o sin app conectada: no decide; Claude sigue su flujo normal (decisión 17). Aprobación respondida en la terminal antes que en el teléfono: la tarjeta pasa a `resolved` y el botón ya no envía nada (`409`, lo expone la historia 13).

### Step 2: `scripts/herdr-hook.sh`

- Se invoca como `<ruta>/herdr-hook.sh <EventName>` (el nombre del evento va como argumento para no tener que parsear JSON en `sh`).
- Lee el JSON del hook por stdin y lo manda con `curl -sS --connect-timeout 1 -m 125 -X POST --data-binary @- -H 'content-type: application/json' -H "x-herdr-terminal: ${HC_TERMINAL_ID:-}" -H "x-herdr-event: $1" -w '\n%{http_code}' http://127.0.0.1:7789/v1/hooks`. `HC_TERMINAL_ID` está en el entorno de toda terminal de hc (verificado: `HC_TERMINAL_ID=t1`, junto a `HC_TERMINAL_SOCKET` y `HC_SCOPE`); Claude Code la hereda y la pasa a sus hooks.
- Si el código es `200` y hay cuerpo, imprime el cuerpo tal cual a stdout (es la salida JSON del hook). Con `204`, otro código, `curl` ausente, gateway caído o cualquier error: sale con código `0` sin imprimir nada.
- Un `curl -m 125` es el techo; el gateway responde antes (como mucho a los 120 s, o 30 s si se aplica el respaldo de la decisión 17).
- `set -u` pero nunca `set -e` que pueda dejar un código distinto de 0; `trap` para borrar temporales.
- La espera larga solo ocurre cuando el gateway la decide (`PermissionRequest`, y `PreToolUse` con `tool_name == "AskUserQuestion"`); para todo lo demás el gateway responde `204` al instante.

### Step 3: `scripts/install-hooks.ts`

CLI (contrato fijado con la historia 11, que lo llama desde `install.sh`): `node apps/herdr-gateway/scripts/install-hooks.ts install --hook-command <ruta absoluta de herdr-hook.sh>` y `node apps/herdr-gateway/scripts/install-hooks.ts uninstall`. Verifica Node ≥ 22.18 (ejecuta `.ts` sin banderas) y si no, sale con un mensaje claro. Solo sintaxis de TypeScript que Node ejecuta sin banderas.

- Funciones puras exportadas (para la prueba): `applyHerdrHooks(settings: Record<string, unknown>, hookCommand: string): Record<string, unknown>` y `removeHerdrHooks(settings): Record<string, unknown>`.
- **Marca única:** cada `command` nuestro termina exactamente con el comentario de shell ` # herdr-gateway-hook` (por ejemplo `/home/u/.local/share/herdr-gateway/herdr-hook.sh PreToolUse # herdr-gateway-hook`). Una entrada es nuestra **solo** si su `command` termina con ese sufijo; nunca se identifica por contener `herdr-hook.sh` (un hook ajeno o un envoltorio del usuario no se toca). `applyHerdrHooks` primero quita las nuestras y luego agrega una entrada `command` por evento, **sin tocar las existentes** (hoy el usuario tiene otras en casi todos los eventos, por ejemplo `moshi-hook` y `herdr-agent-state.sh`):
  ```json
  {
    "matcher": "*",
    "hooks": [
      {
        "type": "command",
        "command": "<hookCommand> PreToolUse # herdr-gateway-hook",
        "timeout": 130
      }
    ]
  }
  ```
  Eventos (10): `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied`, `Notification`, `Stop`, `SessionEnd` (`PostToolUseFailure` y `PermissionDenied` sirven para liberar esperas, decisión 17). `matcher: "*"` solo en `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest` y `PermissionDenied`; los demás sin `matcher`. `timeout`: 130 s en `PermissionRequest` y `PreToolUse`; 10 s en los demás. Si un evento queda sin entradas al quitar las nuestras, se borra la clave; si `hooks` queda vacío, se borra.
- Lee `~/.claude/settings.json` (si no existe, `{}`; si es JSON inválido, aborta sin escribir). Si el resultado es igual al original, no escribe. Si cambia, primero copia a `~/.claude/settings.json.bak-<AAAAMMDD-HHMMSS>` y luego escribe con 2 espacios de sangría, a un temporal y `rename`.
- `install-hooks.test.ts`: con un `settings` que ya tiene hooks ajenos en `PreToolUse` y `Stop` —incluido uno ajeno cuyo `command` contiene `herdr-hook.sh` pero no termina con la marca—: `applyHerdrHooks` agrega las 10 entradas y conserva las ajenas en su orden; aplicarlo dos veces da el mismo resultado; `removeHerdrHooks` deja exactamente el original (el ajeno con `herdr-hook.sh` sigue ahí); `timeout` y `matcher` correctos.

### Step 4: `src/hooks.ts`

```ts
export type HookEventName =
  | "SessionStart"
  | "UserPromptSubmit"
  | "PreToolUse"
  | "PostToolUse"
  | "PostToolUseFailure"
  | "PermissionRequest"
  | "PermissionDenied"
  | "Notification"
  | "Stop"
  | "SessionEnd";
export interface HookEvent {
  address: string | null;
  sessionId: string;
  event: HookEventName;
  payload: Record<string, unknown>;
  atMs: number;
}
export interface SessionInfo {
  terminal: string;
  transcriptPath: string | null;
  claudeVersion: string | null;
}
export interface PendingDecision {
  requestId: string;
  address: string;
  sessionId: string;
  kind: "permission" | "question";
  toolName: string;
  toolInput: Record<string, unknown>;
  suggestions: unknown[];
  questions: unknown[] | null;
  createdAtMs: number;
}
export type DecisionState = "pending" | "resolved" | "expired";
export interface HookHub {
  onEvent(listener: (e: HookEvent) => void): () => void;
  onDecision(listener: (p: PendingDecision, state: DecisionState) => void): () => void;
  onSessionMapped(listener: (sessionId: string, info: SessionInfo) => void): () => void;
  session(sessionId: string): SessionInfo | null;
  terminalForSession(sessionId: string): string | null;
  sessionForTerminal(address: string): string | null;
  setClaudeVersion(sessionId: string, version: string): void;
  pending(address: string): PendingDecision[];
  resolve(
    requestId: string,
    decision: { decision: "allow" | "deny"; answers: Record<string, string> | null },
  ): "ok" | "not_pending";
  setViewerCheck(hasViewer: (address: string) => boolean): void;
  close(): Promise<void>;
}
export function createHookHub(opts?: {
  waitMs?: number;
  now?: () => number;
  setTimer?: (fn: () => void, ms: number) => () => void;
  sessionsFile?: string;
  readFile?;
  writeFileAtomic?;
}): HookHub & {
  handle(req: {
    terminal: string;
    event: string;
    body: unknown;
  }): Promise<{ status: 200; body: object } | { status: 204 }>;
};
export function startHookReceiver(
  hub,
  opts?: { port?: number },
): Promise<{ close(): Promise<void> }>;
```

- `startHookReceiver` escucha en `127.0.0.1:${HERDR_HOOKS_PORT ?? 7789}` (nunca otra IP; sin `whois` porque es local y los procesos del mismo usuario son de confianza), ruta `POST /v1/hooks`, cuerpo máximo 1 MiB; JSON inválido → `400`.
- **Mapa de sesiones (persistido):** `sessionId → { terminal, transcriptPath, claudeVersion }`. Si `x-herdr-terminal` no está vacío, `body.session_id` queda asociado a esa terminal; `transcriptPath = body.transcript_path` (todos los hooks lo traen); una terminal tiene una sola sesión vigente (la última). `claudeVersion` lo pone la historia 13 con `setClaudeVersion` (lo saca del `user-agent` que reporta el tap); `null` si no se sabe. Cada cambio se guarda de forma atómica (temporal + `rename`, modo 0600) en `$XDG_RUNTIME_DIR/herdr-gateway/sessions.json` (directorio 0700; si no hay `XDG_RUNTIME_DIR`, `/run/user/<uid>`), y `createHookHub` lo recarga al arrancar, así un reinicio del gateway no pierde la correlación. `onSessionMapped` avisa cada alta o cambio (la historia 13 reproduce ahí los eventos del tap guardados). `SessionEnd` emite el evento y luego borra la entrada.
- **Eventos:** cada petición válida emite `onEvent({ address, sessionId, event, payload: body, atMs })`.
- **Decisiones (decisiones 17 y 20):** para `PermissionRequest`, y para `PreToolUse` con `body.tool_name === "AskUserQuestion"`: si la terminal es conocida y `hasViewer(address)` es verdadero **en ese momento** (por defecto `() => false`; la historia 13 lo conecta a los chats abiertos), crear `PendingDecision` con `requestId = randomUUID()`, emitir `onDecision(p, "pending")` y esperar. Si no hay app con el chat abierto → `204` inmediato (la petición se queda en la terminal aunque el chat se abra después, decisión 20).
  - `resolve(requestId, …)` sobre una pendiente → responde el hook con `200` y estado `resolved`; sobre una inexistente, resuelta o vencida → `"not_pending"`.
  - **Liberación (decisión 17):** al llegar para la **misma sesión** cualquiera de `PostToolUse`, `PostToolUseFailure`, `PermissionDenied`, `Stop`, `UserPromptSubmit` o `SessionEnd`, toda decisión pendiente de esa sesión se libera con `204` (no decide) y estado `resolved`. `Notification`, `PreToolUse` y `SessionStart` **no** liberan.
  - **Espera máxima:** `waitMs` = 120 000 por defecto; al vencer → `204` y estado `expired`. Respaldo aprobado: si la comprobación del Step 6 muestra que responder en la terminal no termina la espera del hook, el valor por defecto pasa a 30 000 (constante `DEFAULT_DECISION_WAIT_MS`) y se anota en `docs/herdr/hooks.md` y en el tracker.
- **Salidas JSON del hook (`200`)** (formato documentado de Claude Code; anotar en `hooks.md` la versión con la que se comprobó, `claude --version`):
  - `PermissionRequest` permitir: `{ "hookSpecificOutput": { "hookEventName": "PermissionRequest", "decision": { "behavior": "allow" } } }`; denegar: `{ "hookSpecificOutput": { "hookEventName": "PermissionRequest", "decision": { "behavior": "deny", "message": "Denegado desde Herdr" } } }`.
  - `AskUserQuestion` permitir: `{ "hookSpecificOutput": { "hookEventName": "PreToolUse", "permissionDecision": "allow", "updatedInput": { "questions": <tool_input.questions original>, "answers": <answers> } } }` donde `answers` tiene como clave el texto de cada pregunta y como valor la etiqueta elegida (varias separadas por `", "`; es la forma que acepta Claude Code, sin ids estables); denegar: `{ "hookSpecificOutput": { "hookEventName": "PreToolUse", "permissionDecision": "deny", "permissionDecisionReason": "Denegado desde Herdr" } }`.
  - `suggestions` = `body.permission_suggestions ?? []` (solo informativas, decisión 19: el contrato de la app manda `allow`/`deny`); `questions` = `tool_input.questions` para `AskUserQuestion`, si no `null`.
- **Modo de depuración (Access Control):** solo si `HERDR_GATEWAY_DEBUG=1` en el entorno del proceso, el receptor expone además, solo en `127.0.0.1`, `GET /v1/hooks/debug/pending` y `POST /v1/hooks/debug/resolve` `{ requestId, decision, answers }`, y `hasViewer` es siempre verdadero. Sin esa variable, esas rutas responden `404`. `HERDR_GATEWAY_DEBUG` **nunca** se pone en el servicio systemd.

### Step 5: `main.ts` y `http.ts`

- `http.ts`: `GatewayContext` gana `hooks: HookHub`.
- `main.ts`: `const hooks = createHookHub()` (recarga `sessions.json`), `await startHookReceiver(hooks)` al arrancar (independiente del estado de `hc connect`), y `ctx.hooks = hooks`. El receptor arranca aunque Tailscale aún no responda. `hooks.close()` al salir.

### Step 6: comprobación con Claude real (criterio de aceptación)

En `mylinux`, con los hooks instalados (`node apps/herdr-gateway/scripts/install-hooks.ts install --hook-command ~/.local/share/herdr-gateway/herdr-hook.sh`, copiando antes el script) y el gateway corriendo **a mano** con `HERDR_GATEWAY_DEBUG=1` (detener el servicio con `systemctl --user stop herdr-gateway` y lanzar el bundle desde una terminal con esa variable; nunca ponerla en el unit; al terminar, matar ese proceso por el PID que se capturó y volver a arrancar el servicio):

1. Abrir `claude` en una terminal de herdr (modelo barato) y pedir algo que requiera permiso (`Bash(ls)` sin permiso previo). Mientras el hook espera, observar la terminal.
2. Documentar en `docs/herdr/hooks.md`: (a) si el diálogo de permiso sigue visible en la terminal mientras el hook espera; (b) qué pasa si el usuario responde en la terminal primero: ¿Claude sigue?, ¿el proceso del hook se mata?, ¿llega `PostToolUse`/`PermissionDenied`/otro evento de la sesión que libere la espera?; (c) que resolver con `curl -X POST http://127.0.0.1:7789/v1/hooks/debug/resolve` con `allow` hace que la terminal siga sin intervención; (d) lo mismo con una pregunta de `AskUserQuestion` respondida con `answers`; (e) que `HC_TERMINAL_ID` llega al hook; (f) cómo conviven con los otros hooks de `PermissionRequest` que el usuario ya tiene (`moshi-hook`); (g) la versión de Claude Code (`claude --version`); (h) que tras reiniciar el gateway el mapa de `sessions.json` se recarga y la sesión sigue asociada a su terminal.
3. **Respaldo aprobado (decisión 17):** si responder en la terminal **no** termina la espera del hook (ni lo mata ni llega un evento de liberación), cambiar `DEFAULT_DECISION_WAIT_MS` a 30 000 y anotarlo en `hooks.md` y en `## Implementation Notes` del tracker.

### Step 7: pruebas (`hooks.test.ts`)

Con `createHookHub` y temporizadores falsos, llamando `handle(...)` directamente (y una prueba HTTP mínima del receptor en `127.0.0.1` puerto `0`): mapa sesión ↔ terminal y borrado en `SessionEnd`; `PermissionRequest` sin visor → `204` inmediato; con visor → pendiente, `resolve("allow")` → `200` con la salida exacta; `deny`; `AskUserQuestion` con `answers` → `updatedInput` con `questions` y `answers`; vencimiento a los `DEFAULT_DECISION_WAIT_MS` → `204` y `expired`; `resolve` de una ya resuelta → `"not_pending"`; cada uno de `PostToolUse`, `PostToolUseFailure`, `PermissionDenied`, `Stop`, `UserPromptSubmit` y `SessionEnd` de la misma sesión libera la pendiente con `204` y `resolved`, y `Notification` **no** la libera; una petición sin visor en ese momento → `204` aunque después se abra un visor; el mapa de sesiones se guarda y se recarga (archivo falso) con `terminal` y `transcriptPath`; sin `HERDR_GATEWAY_DEBUG`, `/v1/hooks/debug/resolve` → `404`; `PreToolUse` de otra herramienta → `204` inmediato.

## Out of Scope

- Unir hooks, tap y transcripción en el chat, y las rutas `/v1/agents/:address/chat/*` (incluido `409 not_pending`): historia 13.
- Saber qué apps tienen el chat abierto: la historia 13 llama `setViewerCheck`.
- Editar `apps/herdr-gateway/scripts/install.sh`: lo hace la historia 11 (llama a `install-hooks.ts` con el contrato del Step 3).
- Modificar `~/.claude/hooks/herdr-agent-state.sh` o cualquier hook ajeno.
- Guardar el `.jsonl` de la transcripción o leerlo: historia 13.
- Chat para agentes que no sean Claude Code: fuera del alcance.

## Acceptance Criteria

- [ ] Dentro de `apps/herdr-gateway`: `vp test run src/hooks.test.ts scripts/install-hooks.test.ts` pasa.
- [ ] Dentro de `apps/herdr-gateway`: `vp run typecheck` pasa.
- [ ] `grep -n "listen" apps/herdr-gateway/src/hooks.ts` muestra que el receptor solo escucha en `127.0.0.1`.
- [ ] `sh -n apps/herdr-gateway/scripts/herdr-hook.sh` sin errores; con el gateway detenido, `echo '{}' | apps/herdr-gateway/scripts/herdr-hook.sh Stop; echo $?` imprime solo `0`.
- [ ] En `mylinux`: `install` dos veces deja una sola entrada nuestra por evento y las ajenas intactas (comparar con el `.bak`); `uninstall` deja `settings.json` igual que antes de instalar.
- [ ] **Comprobación con Claude real** (Step 6) hecha y documentada en `docs/herdr/hooks.md` con los puntos (a)–(f).
- [ ] `node scripts/herdr/check-patch-surface.ts` sale con código 0.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
