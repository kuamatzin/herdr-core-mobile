---
status: not_started
feature: "herdr-core-mobile"
story: "09"
layers:
  - "frontend"
dependencies:
  - "story-05"
  - "story-07"
---

# Story 09: App, terminal

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code cuya app móvil abre en Herdr y se conecta a un gateway por servidor. El gateway ya expone el stream WebSocket de cualquier terminal (historia 5). Esta historia hace la terminal en vivo en la app: reemplaza la pantalla temporal `HerdrAgentRoute` por la vista de los paneles de la pestaña (uno a la vez, deslizando), monta la terminal nativa de T3 (`TerminalSurface`) sobre el stream con barra de teclas, ajuste al tamaño del teléfono y reconexión automática, y reemplaza la hoja temporal `HerdrNewTabSheet` para abrir una pestaña con Shell, Claude o Codex.

## Dependencies

- Depends on: Story 05 — Gateway, terminal (`WS /v1/terminals/:address/stream`, `POST /v1/tabs`, `TERMINAL_STREAM_CLOSE_CODES` en `@herdr/core-client/gateway-api`).
- Depends on: Story 07 — App, conexión e inicio (`createGatewayClient(baseUrl)` con `wsUrl`/`postJson`; `useHerdrMachine(hostId)`, `useHerdrTerminal(hostId, address)`, `useHerdrHosts()`; rutas `HerdrAgent` `{ hostId, address }`, `HerdrNewTab` `{ hostId, spaceId }`, `HerdrFiles` `{ hostId, path? }`; archivos temporales `HerdrAgentRoute.tsx`, `HerdrNewTabSheet.tsx` y el componente `HerdrFilesInspector` que devuelve `null` hasta la historia 10).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Alcance de la v1?** → **Decision:** espacios, pestañas, terminal en vivo, agentes por prioridad, archivos (navegar, ver, descargar, subir) y avisos con la app abierta. Las notificaciones push van al final y bloqueadas (decisión 13). El chat va justo después de la base (decisión 8).
4. **Q: ¿Tamaño de la terminal?** → **Decision:** se ajusta **siempre** al teléfono. La app manda su tamaño en celdas al abrir y al rotar. En herdr-core, el último visor activo define el tamaño, así que el escritorio se reacomoda mientras el teléfono la usa y lo recupera cuando el escritorio vuelve a enfocarla.

## Affected Files

### Frontend

- [ ] Pattern to follow: `apps/mobile/src/features/terminal/ThreadTerminalRouteScreen.tsx` (cómo monta `TerminalSurface`, su barra de teclas sobre el teclado —`terminalToolbarActions`, línea ~540, y su render, línea ~972— y la altura `TERMINAL_ACCESSORY_HEIGHT = 52`). El componente `TerminalSurface` de `apps/mobile/src/features/terminal/NativeTerminalSurface.tsx`, props: `terminalKey`, `buffer` (`string | null`), `output` (`TerminalOutputState | null`, de `@t3tools/client-runtime/state/terminal`), `fontSize`, `grid`, `isRunning`, `readOnly`, `autoFocus`, `keyboardFocusRequest`, `captureRequest`, `onCapture`, `theme`, `onInput(data)`, `onResize({ cols, rows })`. Ayudas exportadas reutilizables: `resolveModifiedTerminalInput`, `chunkTerminalWrite`, `encodeTerminalPaste` (`features/terminal/terminalInput.ts`), `getMobileTerminalTheme` (`features/terminal/terminalTheme.ts`). Formato de salida: `packages/client-runtime/src/state/terminal.ts` y `terminalOutput.ts`.
- [ ] Conventions to follow: estado con átomos de `@effect/atom-react` o hooks de React con estado local donde baste; `PascalCase.tsx` para componentes, `camelCase.ts` para lógica; estilos con clases Uniwind y `useThemeColor`; símbolos con `AppSymbol`; texto con `AppText`; componentes de `components/ui`; pruebas `*.test.ts` junto al archivo; efectos solo para sincronizar con el WebSocket, con su `oxlint-disable` razonado.
- [ ] `apps/mobile/src/features/herdr/HerdrAgentRoute.tsx` — reemplazar la pantalla temporal.
- [ ] `apps/mobile/src/features/herdr/HerdrTerminalPane.tsx` — nuevo.
- [ ] `apps/mobile/src/features/herdr/state/useTerminalStream.ts` — nuevo.
- [ ] `apps/mobile/src/features/herdr/HerdrNewTabSheet.tsx` — reemplazar la hoja temporal.
- [ ] Prueba: `apps/mobile/src/features/herdr/state/useTerminalStream.test.ts`.

## Implementation Steps

### Step 1: contrato

Del API Contract (binding):

**`WS /v1/terminals/:address/stream?cols=80&rows=24&history=2000`**: servidor → `{ "type": "output", "data": "…" }`, `{ "type": "exit", "reason": "exited", "exitCode": 0 }`; cliente → `{ "type": "input", "data": "…" }`, `{ "type": "resize", "cols": 60, "rows": 30 }`.

**`POST /v1/tabs`** `{ "spaceId": "sp…", "argv": ["claude"] }` (`argv` opcional; sin él, shell) → `{ "address": "t9", "tabId": "tb…" }`.

Códigos de cierre del WebSocket (historia 5): `4404` la terminal no existe, `4503` el core no está disponible, `4400` parámetros inválidos.

### Step 2: `state/useTerminalStream.ts`

- `appendStreamOutput(state: TerminalOutputState, data: string, maxBytes = 2 * 1024 * 1024): TerminalOutputState` (función pura): `@t3tools/client-runtime/state/terminal` exporta el tipo `TerminalOutputState`, `INITIAL_TERMINAL_OUTPUT_CURSOR`, `readTerminalOutputUpdate` y `terminalOutputText`, pero **no** `appendOutput`, `resetOutput` ni `EMPTY_TERMINAL_OUTPUT_STATE` (solo es interno del paquete, verificado), y `packages/client-runtime` no está en la lista permitida. Implementar aquí el mismo modelo: nuevo chunk `{ startOffset: state.nextOffset, data, byteLength: <bytes UTF-8 de data> }`, `nextOffset += data.length` (offsets en unidades de `string`, como el original), `retainedBytes` sumado, y descartar chunks del principio mientras `retainedBytes > maxBytes`. `resetStreamOutput(state)`: `chunks: []`, `retainedBytes: 0`, `nextOffset: 0`, `resetVersion + 1`. Estado inicial: `{ generation: 0, chunks: [], retainedBytes: 0, resetVersion: 0, nextOffset: 0 }`.
- `useTerminalStream({ hostId, address, grid: { cols, rows } | null, enabled })` → `{ output: TerminalOutputState; phase: "connecting" | "live" | "reconnecting" | "exited" | "gone"; exit: { reason: string; exitCode: number | null } | null; send(data: string): void; resize(cols, rows): void }`.
  - **Tamaño inicial:** `grid` llega primero **estimado** desde el `onLayout` de la vista que contiene la terminal, con la misma fórmula que `estimateGridSize` en `NativeTerminalSurface.tsx` (función local, no exportada: copiarla en `useTerminalStream.ts` como `estimateTerminalGrid({ width, height, fontSize })`: celda de `fontSize * 0.62` × `fontSize * 1.35`, `cols` entre 20 y 400, `rows` entre 5 y 200). El WebSocket se abre en cuanto hay `grid` estimado y `enabled`; cada `onResize` de `TerminalSurface` corrige el tamaño con un mensaje `resize`. Así el stream nunca espera a un `onResize` que podría no llegar. URL: `client.wsUrl("/v1/terminals/" + encodeURIComponent(address) + "/stream", { cols, rows, history: 2000 })`.
  - `output` → `appendStreamOutput`. Agrupar las actualizaciones de estado por cuadro (`requestAnimationFrame` o un lote de 16 ms) para no renderizar por mensaje.
  - `exit` → `phase: "exited"` con `{ reason, exitCode }`; no reconectar.
  - Cierre `4404` → `phase: "gone"`; no reconectar.
  - Cualquier otro cierre o error → `phase: "reconnecting"` y reconectar con espera creciente 1, 2, 4, 8, 16, 30 s; al reconectar, `resetStreamOutput` (el gateway repinta la pantalla y el historial).
  - `send(data)` → `{ type: "input", data }` partiendo con `chunkTerminalWrite`. `resize(cols, rows)` → `{ type: "resize", cols, rows }`, con un retardo de 150 ms que se queda con el último tamaño, y guardándolo para la próxima reconexión.
  - Al salir de la pantalla (desmontar) se cierra el WebSocket (el gateway manda `CLOSE` al core).
- `useTerminalStream.test.ts`: `estimateTerminalGrid` (límites de 20/400 y 5/200), `appendStreamOutput` (offsets, `byteLength` con caracteres multibyte, recorte al superar `maxBytes`) y `resetStreamOutput`; que `readTerminalOutputUpdate` (exportada por `@t3tools/client-runtime/state/terminal`) devuelve `append` con solo lo nuevo tras un `appendStreamOutput`, y `reset` tras un `resetStreamOutput`.

### Step 3: `HerdrTerminalPane.tsx`

- `TerminalSurface` con `terminalKey = "herdr:" + hostId + ":" + address`, `buffer = null`, `output` del hook, `isRunning = phase === "live"`, `readOnly = phase !== "live"`, `theme = getMobileTerminalTheme(...)` (como `ThreadTerminalRouteScreen`), `onInput = send`, `onLayout` del contenedor → `setGrid(estimateTerminalGrid(...))` si aún no hay `grid`; `onResize = ({ cols, rows }) => { setGrid(...); resize(cols, rows) }` (decisión 10: el tamaño del teléfono se manda al abrir y al rotar; no existe modo espejo).
- **Barra de teclas** sobre el teclado (como `ThreadTerminalRouteScreen.tsx`; el tipo `TerminalToolbarAction` y su lista son locales a ese archivo, así que se recrea aquí la lista, no se importa): `esc` (`\u001b`), `tab` (`\t`), `ctrl` (modificador pendiente; la siguiente tecla se resuelve con `resolveModifiedTerminalInput`), `↑` `↓` `←` `→` (`\u001b[A`, `\u001b[B`, `\u001b[D`, `\u001b[C`) y `^C` (`\u0003`). Altura 52.
- Estados: `connecting`/`reconnecting` → indicador discreto ("Reconectando…") sin tapar la última pantalla; `exited` → mensaje exacto **"La terminal terminó (código N)"** (N = `exitCode`, o la `reason` si es nulo) y botón "Volver" (`navigation.goBack()`); `gone` → "La terminal ya no existe" y "Volver".

### Step 4: `HerdrAgentRoute.tsx`

Mantener `export function HerdrAgentRoute(props: StaticScreenProps<{ hostId: string; address: string }>)`.

- Con `useHerdrMachine(hostId)` encontrar la pestaña que contiene `address` (en `snapshot.spaces[].tabs[].panes[]`); sus paneles son las páginas. Si la terminal no está en ninguna pestaña, una sola página.
- Paginación horizontal (`FlatList` o `ScrollView` horizontal con `pagingEnabled`), una página por panel, página inicial = la de `address`, indicador de puntos si hay más de uno. **Solo el panel visible tiene stream abierto** (`enabled` del hook); los demás muestran su nombre hasta que se ven. No se muestran varios paneles a la vez.
- Encabezado: nombre de la pestaña y del espacio; botón "Archivos" → `navigation.navigate("HerdrFiles", { hostId, path: space.cwd })` en teléfono.
- Montar `<HerdrFilesInspector hostId={hostId} path={space?.cwd ?? terminal?.cwd ?? null} />` (de `./HerdrFilesInspector`; hoy devuelve `null`, la historia 10 lo convierte en el inspector de iPad).
- Si el servidor está "Sin conexión", mostrarlo sobre la terminal y seguir reintentando.
- Dejar el contenido de la página en un componente (`HerdrTerminalPane`) para que la historia 15 agregue el selector Chat/Terminal alrededor sin reescribir esta pantalla.

### Step 5: `HerdrNewTabSheet.tsx`

Mantener `export function HerdrNewTabSheet(props: StaticScreenProps<{ hostId: string; spaceId: string }>)`. Tres opciones: **Shell** (sin `argv`), **Claude** (`["claude"]`), **Codex** (`["codex", "--no-daemon"]`: herdr pide que Codex corra con `--no-daemon` dentro de `hc`; la opción existe en codex 0.166, verificado). Al elegir: `postJson("/v1/tabs", { spaceId, argv? })` → cerrar la hoja y `navigation.navigate("HerdrAgent", { hostId, address })`. Error → `ErrorBanner` con el mensaje y sin cerrar.

## Out of Scope

- Chat y el selector Chat/Terminal: historias 14 y 15 (aquí siempre se muestra la terminal).
- Pantalla e inspector de archivos: historia 10 (aquí solo se monta `HerdrFilesInspector` y se navega a `HerdrFiles`).
- Ver varios paneles de una pestaña al mismo tiempo: fuera del alcance.
- Escribir en la terminal en modo espejo sin ajustar el tamaño: fuera del alcance (decisión 10).
- Crear, renombrar o eliminar espacios; renombrar o cerrar pestañas.
- Usar `ssh` desde el teléfono.
- Editar archivos de T3 (`ThreadTerminalRouteScreen.tsx`, `NativeTerminalSurface.tsx`, `packages/client-runtime`): se reutilizan sin cambios.

## Acceptance Criteria

- [ ] Dentro de `apps/mobile`: `vp test run src/features/herdr/state/useTerminalStream.test.ts` pasa.
- [ ] Dentro de `apps/mobile`: `vp run typecheck` pasa.
- [ ] Ningún archivo fuera de `apps/mobile/src/features/herdr/` cambia; `node scripts/herdr/check-patch-surface.ts` sale con código 0.
- [ ] Con un gateway real (Android en `mylinux`): abrir un shell, la terminal se ve ajustada al ancho del teléfono, `ls` muestra la salida, `^C` interrumpe un `sleep 100`, rotar el teléfono reajusta; matar la conexión (apagar el Wi‑Fi unos segundos) reconecta sola; `exit` en el shell muestra "La terminal terminó (código 0)". "+" → Shell en un espacio abre una pestaña nueva. Anotar el resultado en el tracker.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
