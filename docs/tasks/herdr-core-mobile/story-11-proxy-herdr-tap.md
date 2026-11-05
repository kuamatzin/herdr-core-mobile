---
status: not_started
feature: "herdr-core-mobile"
story: "11"
layers:
  - "backend"
  - "other"
dependencies:
  - "story-04"
---

# Story 11: Proxy `herdr-tap`

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code que agrega una app móvil para herdr-core, con un chat nativo para los agentes de Claude Code que corren en terminales de herdr. Para ver la respuesta palabra por palabra sin tocar el Claude Code real, cada servidor corre `herdr-tap`: un proxy mínimo y aparte que reenvía todo a `https://api.anthropic.com` sin alterar los cuerpos y, en paralelo, emite los eventos del stream (texto, herramientas, pensamiento) a un socket local que leerá el gateway (historia 13). Esta historia crea el proxy, su parser SSE, la función `claude` (cargada desde `~/.zshenv`) que lo usa solo cuando es seguro y sin cambiar nada más del entorno, su servicio systemd, y **comprueba con Claude real** que funciona. Es una historia de riesgo: si el proxy falla, el resto del plan sigue.

## Dependencies

- Depends on: Story 04 — Gateway base (`apps/herdr-gateway/` con `package.json`, `vite.config.ts` y `scripts/install.sh` organizado en funciones con el bloque `# --- herdr-tap y hooks (historias 11 y 12) ---`).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Cómo se alimenta el chat de Claude Code con los mismos agentes del escritorio?** → **Decision:** dos fuentes oficiales, con el Claude Code real intacto en su terminal:
   - **Proxy `herdr-tap`** vía `ANTHROPIC_BASE_URL`: texto y herramientas palabra por palabra, correlacionados con el header `x-claude-code-session-id`.
   - **Hooks de Claude Code**: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PermissionRequest`, `Notification`, `Stop`, `SessionEnd`.
   - **Transcripción** (`~/.claude/projects/<proyecto>/<session-id>.jsonl`): solo para el historial al abrir el chat, porque su formato es interno y puede cambiar.
   - **Envío desde el teléfono**: `terminal.send` de herdr-core, sobre la misma terminal.
4. **Q: ¿A qué sesiones se aplica el proxy?** → **Decision:** a todo Claude Code que se lance desde zsh en el servidor, sin cambiar nada más del entorno. `~/.zshenv` carga `herdr-tap-env.sh`, que solo define una función de zsh `claude`; esa función corre `command claude` con `ANTHROPIC_BASE_URL=http://127.0.0.1:7790` puesta solo para ese proceso, y solo si en ese momento se cumplen tres condiciones: (a) `ANTHROPIC_BASE_URL` no está definida, (b) el archivo PID de `herdr-tap` existe y el proceso responde a `kill -0`, y (c) `/proc/<pid>/cmdline` de ese PID contiene `herdr-tap`. Si alguna falla, corre `command claude` sin tocar nada y Claude va directo a Anthropic. Ninguna otra variable ni programa cambia. Ejemplo: `claude-glm` y `flashgate`, que exportan su propia `ANTHROPIC_BASE_URL`, siguen su ruta y no tienen chat palabra por palabra.
5. **Q: ¿Cuándo va el chat?** → **Decision:** justo después de la base (olas 3 a 6 del breakdown). Las historias 11 y 12 comprueban primero, con Claude real, que el proxy y los hooks funcionan.
6. **(plan) Gateway sin Effect.** El gateway y `packages/herdr` usan TypeScript plano con Promises y APIs de Node, no Effect, para no acoplarse a la evolución de las librerías del servidor T3. `packages/herdr` no importa nada de `@t3tools/*`.
7. **(plan) El proxy es un proceso aparte.** `herdr-tap` corre como su propio servicio systemd y reenvía a `https://api.anthropic.com` sin depender del gateway. Si el gateway se reinicia o cae, los agentes no se cortan. `herdr-tap` nunca guarda credenciales ni cuerpos de petición o respuesta; solo emite eventos por un socket Unix local, sin esperar respuesta.

## Affected Files

### Backend

- [ ] Pattern to follow: la especificación de gateways de Claude Code (`https://code.claude.com/docs/en/llm-gateway-protocol.md`). Reglas documentadas que el proxy debe cumplir: reenviar `anthropic-version` y `anthropic-beta` tal cual; pasar el SSE evento por evento sin buffer; conservar `ping`; reenviar los cuerpos de error; pasar `x-should-retry`, `anthropic-ratelimit-unified-*` y `retry-after`; el header de sesión es `x-claude-code-session-id`; funciona con una suscripción de claude.ai cuando solo se define `ANTHROPIC_BASE_URL`.
- [ ] Conventions to follow: TypeScript plano con `node:http`, `node:https`, `node:net`, `node:fs` (decisión 14), sin dependencias de npm en `tap/` (el proceso debe ser mínimo); sin `enum`/`namespace`; pruebas `*.test.ts` junto al archivo, sin sleeps ni relojes reales.
- [ ] `apps/herdr-gateway/src/tap/main.ts` — nuevo: el proxy.
- [ ] `apps/herdr-gateway/src/tap/sse.ts` — nuevo: parser SSE y armado de eventos.
- [ ] `apps/herdr-gateway/scripts/herdr-tap-env.sh` — nuevo.
- [ ] `apps/herdr-gateway/scripts/install.sh` — agregar el servicio del tap, la línea de `.zshenv` y la llamada a la instalación de hooks.
- [ ] `apps/herdr-gateway/vite.config.ts` — segunda entrada del bundle (`src/tap/main.ts`).
- [ ] `docs/herdr/tap.md` — nuevo: resultado de la comprobación.
- [ ] Pruebas: `apps/herdr-gateway/src/tap/sse.test.ts`, `apps/herdr-gateway/src/tap/main.test.ts`.

## Implementation Steps

### Step 1: contrato del socket

Del API Contract: **Interno, socket Unix `$XDG_RUNTIME_DIR/herdr-gateway/tap.sock`:** datagramas `{ "sessionId", "agentId", "requestId", "event", "data" }`.

**Desviación anotada:** Node no soporta sockets Unix de datagramas (`dgram` solo tiene `udp4`/`udp6`). Se usa un socket Unix de stream con **un JSON por línea** (`\n`), cada línea es un "datagrama". Se conserva la semántica: sin respuesta, nunca bloquea, se descarta si falla. Quien escucha (el gateway, historia 13) crea el directorio `$XDG_RUNTIME_DIR/herdr-gateway` con modo 0700 y el socket con 0600; el tap solo se conecta. Access Control (binding): los procesos locales del mismo usuario son de confianza (ya tienen acceso de shell), así que no hay autenticación sobre el socket. Anotar esta desviación en `docs/herdr/tap.md`.

Forma de cada línea: `{ "sessionId": string, "agentId": string | null, "requestId": string, "event": "message_start" | "content_block_start" | "content_block_delta" | "content_block_stop" | "message_delta" | "message_stop", "data": {...} }`, con `data`:

- `message_start` → `{ messageId, model, anthropicRequestId, clientVersion }` (`anthropicRequestId` = header de respuesta `request-id` o `null`; `clientVersion` = la versión de `claude-cli/<versión>` del `user-agent` de la petición, o `null`; la historia 13 la guarda como `claudeVersion` del mapa de sesiones)
- `content_block_start` → `{ index, type: "text" | "tool_use" | "thinking" | …, id?: string, name?: string }` (de `content_block`)
- `content_block_delta` → `{ index, text }` (`text_delta`), `{ index, partialJson }` (`input_json_delta`) o `{ index, thinking }` (`thinking_delta`); otros tipos de delta se omiten
- `content_block_stop` → `{ index }`
- `message_delta` → `{ stopReason: delta.stop_reason ?? null }`
- `message_stop` → `{}`

`sessionId` = header de la petición `x-claude-code-session-id` (si no viene, no se emite nada para esa petición). `agentId` = header `x-claude-code-agent-id` si existe, si no `null` (en la comprobación, anotar qué nombres de headers `x-claude-code-*` llegan, **solo los nombres**). `requestId` = un id local estable asignado al **recibir** la petición, antes de la respuesta (`tap-<pid>-<n>`, contador por proceso), igual para todos los eventos de esa petición; el `request-id` de Anthropic va aparte en `message_start.data.anthropicRequestId`.

### Step 2: `tap/sse.ts`

- `export class SseParser { constructor(onEvent: (event: string, data: unknown) => void); push(chunk: Uint8Array): void; end(): void }`: incremental, acepta `\n` y `\r\n`, eventos separados por línea vacía, varias líneas `data:` unidas con `\n`, `JSON.parse` del data (si falla, se ignora ese evento). Usa `TextDecoder` en modo `stream: true`.
- `export function toTapEvent(event: string, data: unknown): { event: TapEventName; data: object } | null` con el mapeo del Step 1 (`ping`, `error` y desconocidos → `null`).

### Step 3: `tap/main.ts`

- Escucha en `127.0.0.1:${HERDR_TAP_PORT ?? 7790}` (nunca otra interfaz). Destino `HERDR_TAP_UPSTREAM ?? "https://api.anthropic.com"` (la variable existe para las pruebas).
- Cada petición se reenvía con `https.request` (o `http.request` si el destino es `http:`), mismo método, ruta y query, **todos** los headers del cliente salvo los de salto (`connection`, `keep-alive`, `proxy-*`, `te`, `trailer`, `upgrade`) y `host` (que pasa a ser el del destino); `authorization`, `x-api-key`, `anthropic-version`, `anthropic-beta` y los `x-claude-code-*` pasan tal cual. El cuerpo de la petición se pasa en stream con contrapresión (`pipeline(req, upstreamReq)`), sin leerlo ni guardarlo.
- La respuesta: `res.writeHead(status, headers)` con los headers de extremo a extremo del destino (incluyendo `anthropic-ratelimit-unified-*`, `retry-after`, `x-should-retry`, `request-id`, `content-type`, `content-encoding`) y **sin** los de salto (`connection`, `keep-alive`, `transfer-encoding`, `proxy-*`, `te`, `trailer`, `upgrade`): Node arma su propio framing. `content-length` se conserva si vino (cuerpo de tamaño conocido); si no vino, Node usa chunked. `res.flushHeaders()`, `socket.setNoDelay(true)`. Cada chunk se escribe en cuanto llega, sin buffer y sin quitar `ping`, con contrapresión: si `res.write(c)` devuelve `false`, `upstreamRes.pause()` hasta el `drain` de `res`. Cuerpos y códigos de error se reenvían igual; los bytes del cuerpo nunca se modifican.
- **En paralelo, sin retrasar el reenvío:** si la petición es `POST` a una ruta que empieza con `/v1/messages`, el status es 200, `content-type` empieza con `text/event-stream` y **no** hay `content-encoding`, los mismos chunks se pasan también a un `SseParser` (después de `res.write`) cuyo resultado va a `emitTapEvent(...)`. Con `content-encoding` no se interpreta (se reenvía igual) y se anota en `tap.md`.
- `emitTapEvent`: una sola conexión `net.connect({ path: $XDG_RUNTIME_DIR/herdr-gateway/tap.sock })` reutilizada; si no está conectada, intentar reconectar como máximo una vez cada 5 s y descartar el evento; si `socket.writableLength > 1 MiB`, descartar; errores del socket → descartar en silencio. Nunca `await`.
- Cliente que corta (petición o respuesta cerrada antes de terminar) → `upstreamReq.destroy()`; destino que corta a media respuesta → `res.destroy()` (el cliente ve el corte, no una respuesta truncada como completa). Destino inalcanzable → `502` con cuerpo `{"type":"error","error":{"type":"api_error","message":"herdr-tap: upstream unreachable"}}` y header `x-should-retry: true`.
- Sin tiempos de espera propios (las respuestas pueden durar minutos).
- **Nunca** escribe a disco ni registra headers o cuerpos; los logs (stderr → journal) solo dicen "escuchando en …" y errores de arranque. El único archivo es el PID: al arrancar, `mkdir -p $XDG_RUNTIME_DIR/herdr-tap` y escribir `process.pid` en `$XDG_RUNTIME_DIR/herdr-tap/pid`; borrarlo en `SIGTERM`/`SIGINT` antes de salir. Si `XDG_RUNTIME_DIR` no está definido, usar `/run/user/<uid>`.
- No importa nada del gateway (`../main.ts`, `../snapshot.ts`, …): es un proceso aparte (decisión 15).

### Step 4: bundle

En `apps/herdr-gateway/vite.config.ts`, agregar `src/tap/main.ts` a `pack.entry` para producir un segundo archivo (`herdr-tap`); verificar el nombre del archivo resultante.

### Step 5: `scripts/herdr-tap-env.sh` (se carga desde `~/.zshenv`; solo define una función, decisión 6)

```zsh
# herdr-tap: define solo la función `claude` (decisión 6). No exporta nada.
claude() {
  local pidfile="${XDG_RUNTIME_DIR:-/run/user/$UID}/herdr-tap/pid" pid
  if [[ -z "${ANTHROPIC_BASE_URL-}" && -r "$pidfile" ]] && read -r pid < "$pidfile" \
     && kill -0 "$pid" 2>/dev/null && [[ "$(tr '\0' ' ' < /proc/$pid/cmdline 2>/dev/null)" == *herdr-tap* ]]; then
    ANTHROPIC_BASE_URL=http://127.0.0.1:7790 command claude "$@"
  else
    command claude "$@"
  fi
}
```

- Es sintaxis de zsh (el archivo solo lo carga zsh). No exporta variables ni toca nada más del entorno: `ANTHROPIC_BASE_URL` se pone solo para ese proceso `claude`.
- La condición (c) evita usar un PID reciclado: el proceso debe tener `herdr-tap` en su `cmdline` (el bundle se llama `herdr-tap.mjs`).
- Si ya existe una función o alias `claude` del usuario, no se pisa: el archivo empieza con `(( $+functions[claude] || $+aliases[claude] )) && return 0` y `tap.md` lo documenta.

### Step 6: `scripts/install.sh` (en el bloque `# --- herdr-tap y hooks (historias 11 y 12) ---`)

Instalar (idempotente):

1. Copiar el bundle del tap a `~/.local/share/herdr-gateway/herdr-tap.mjs` y `herdr-tap-env.sh` a `~/.local/share/herdr-gateway/herdr-tap-env.sh`, cada uno a un temporal en la misma carpeta y luego `mv` (atómico), **antes** de reiniciar servicios. `install.sh` siempre corre desde el checkout del repo (compila desde ahí), así que los scripts de la historia 12 están disponibles en la misma ruta.
2. `~/.config/systemd/user/herdr-tap.service` (mismo formato que `herdr-gateway.service`: `ExecStart=<node absoluto> %h/.local/share/herdr-gateway/herdr-tap.mjs`, `Restart=always`, `RestartSec=2`, `WantedBy=default.target`); `daemon-reload`, `enable`, `restart`.
3. Agregar a `~/.zshenv`, solo si no existe ya una línea con el marcador `# herdr-tap`: `[ -r "$HOME/.local/share/herdr-gateway/herdr-tap-env.sh" ] && . "$HOME/.local/share/herdr-gateway/herdr-tap-env.sh" # herdr-tap`.
4. **Hooks (historia 12, que corre en paralelo):** si existen `apps/herdr-gateway/scripts/herdr-hook.sh` y `apps/herdr-gateway/scripts/install-hooks.ts`, copiar `herdr-hook.sh` a `~/.local/share/herdr-gateway/herdr-hook.sh` (`chmod +x`) y correr `node <repo>/apps/herdr-gateway/scripts/install-hooks.ts install --hook-command "$HOME/.local/share/herdr-gateway/herdr-hook.sh"` (contrato fijado con la historia 12; requiere Node ≥ 22.18 para ejecutar `.ts`; el script lo verifica). Si no existen, saltar en silencio.
   Desinstalar (`--uninstall`): `systemctl --user disable --now herdr-tap`, borrar su unit, borrar de `~/.zshenv` la línea con `# herdr-tap` (con respaldo `~/.zshenv.bak-<fecha>`), borrar los archivos copiados, y si existe `install-hooks.ts`, correr `node .../install-hooks.ts uninstall`.

### Step 7: pruebas

- `sse.test.ts`: eventos partidos en cualquier byte, `\r\n`, varias líneas `data:`, `ping` ignorado por `toTapEvent`, mapeo de `text_delta`, `input_json_delta`, `thinking_delta`, `content_block_start` de `tool_use` con `id` y `name`.
- `main.test.ts`: un servidor HTTP local que imita a Anthropic (`HERDR_TAP_UPSTREAM=http://127.0.0.1:<puerto>`) y un servidor `net` en un socket Unix temporal: (a) un stream SSE llega al cliente evento por evento —el cliente recibe el primer evento antes de que el servidor falso mande el segundo, sincronizado con promesas, no con tiempos—, con `ping` incluido; (b) headers `anthropic-version`, `anthropic-beta`, `authorization` llegan al destino y `x-should-retry`, `retry-after`, `anthropic-ratelimit-unified-status` vuelven al cliente; (c) un `429` con cuerpo JSON se reenvía igual; (d) el socket Unix recibe líneas con el `sessionId` del header; (e) sin socket Unix, el proxy sigue respondiendo igual.

### Step 8: comprobación con Claude real (criterio de aceptación)

En `mylinux`, con `install.sh` corrido: abrir un shell nuevo de zsh en una terminal de herdr (para que `.zshenv` defina la función), verificar `whence -w claude` (debe decir `function`) y que `echo $ANTHROPIC_BASE_URL` sigue vacío en el shell, abrir `claude` interactivo (con la cuenta de suscripción de claude.ai del usuario, modelo barato) y mandar un prompt corto que use una herramienta. Con un oyente temporal en `$XDG_RUNTIME_DIR/herdr-gateway/tap.sock` (un `node -e` con `net.createServer` que imprima solo `event` y `sessionId`, nunca el texto) comprobar que llegan `content_block_delta` con el `sessionId` de esa sesión (comparar con el `session_id` que muestra `/status` o el nombre del `.jsonl` en `~/.claude/projects/`). Documentar en `docs/herdr/tap.md`: resultado (funciona / no funciona y por qué), nombres de headers `x-claude-code-*` observados, si hubo `content-encoding`, que `claude-glm`/`flashgate` siguen su propia ruta (decisión 6), la desviación del socket de stream, que `ANTHROPIC_BASE_URL` desactiva Remote Control de Anthropic en esas sesiones, y que si el tap cae a media sesión esa sesión falla hasta reiniciarla.

## Out of Scope

- Escuchar el socket `tap.sock` en el gateway y unir los eventos al chat: historia 13.
- Hooks de Claude Code y su receptor: historia 12 (aquí solo se llama a su instalador si ya existe).
- Guardar credenciales, cuerpos o headers, o cualquier registro de contenido: prohibido (decisión 15).
- Afectar sesiones con su propia `ANTHROPIC_BASE_URL` (`claude-glm`, `flashgate`).
- Chat para agentes que no sean Claude Code: fuera del alcance.

## Acceptance Criteria

- [ ] Dentro de `apps/herdr-gateway`: `vp test run src/tap/sse.test.ts src/tap/main.test.ts` pasa.
- [ ] Dentro de `apps/herdr-gateway`: `vp run typecheck` pasa.
- [ ] `grep -rn "writeFile\|appendFile\|console\.\(log\|info\|debug\)" apps/herdr-gateway/src/tap` solo encuentra la escritura del archivo PID y el log de arranque.
- [ ] `grep -rn "from \"\.\./" apps/herdr-gateway/src/tap` no encuentra imports del gateway.
- [ ] `main.test.ts` cubre además: header `transfer-encoding`/`connection` del destino no se copian; corte del cliente a media respuesta destruye la petición al destino; `requestId` igual en todos los eventos de una petición.
- [ ] En `mylinux`: `systemctl --user is-active herdr-tap` responde `active`; correr `install.sh` dos veces deja una sola línea `# herdr-tap` en `~/.zshenv`; `--uninstall` la quita.
- [ ] En un zsh nuevo, `env | grep ANTHROPIC_BASE_URL` no muestra nada (el shell no exporta nada); `claude` es una función. Con un `claude` falso en el `PATH` que imprima `$ANTHROPIC_BASE_URL`: con el tap vivo imprime `http://127.0.0.1:7790`; con `ANTHROPIC_BASE_URL=http://otra` definida imprime `http://otra`; con el tap detenido, o con un archivo PID que apunta a un proceso que no es `herdr-tap`, imprime vacío.
- [ ] **Comprobación con Claude real** (Step 8) hecha y documentada en `docs/herdr/tap.md`: una sesión interactiva de Claude Code en una terminal de herdr responde normal con el tap activo, y el socket recibe deltas de texto con el `sessionId` de esa sesión. Si falla, `tap.md` explica por qué y la historia se marca en el tracker con el resultado.
- [ ] `node scripts/herdr/check-patch-surface.ts` sale con código 0.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
