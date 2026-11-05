---
status: not_started
feature: "herdr-core-mobile"
story: "08"
layers:
  - "frontend"
dependencies:
  - "story-07"
---

# Story 08: App, lista de agentes e iPad

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code cuya app móvil abre en Herdr (historia 7) y mantiene un modelo en vivo por servidor. Esta historia convierte el inicio en la vista real: arriba los agentes que necesitan atención (`blocked`, `done`) y los que están trabajando, ordenados por prioridad; debajo, cada servidor con sus carpetas plegables, espacios y pestañas. En iPad, la misma lista ocupa la barra lateral del layout adaptativo de T3. Además, con la app abierta, avisa cuando un agente pasa a `blocked` o `done`.

## Dependencies

- Depends on: Story 07 — App, conexión e inicio (`useHerdrSnapshot()`, `useHerdrMachine(hostId)`, `HerdrMachineView`, `useHerdrHosts()`; rutas `HerdrAgent` `{ hostId, address }`, `HerdrFiles` `{ hostId, path? }`, `HerdrNewTab` `{ hostId, spaceId }`, `HerdrAddHost`; `HerdrHomeRoute.tsx` simple).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Qué se ve al abrir?** → **Decision:** Herdr es la pantalla principal (ruta `Home`). Las pestañas de T3 no aparecen.
4. **Q: ¿Alcance de la v1?** → **Decision:** espacios, pestañas, terminal en vivo, agentes por prioridad, archivos (navegar, ver, descargar, subir) y avisos con la app abierta. Las notificaciones push van al final y bloqueadas (decisión 13). El chat va justo después de la base (decisión 8).
5. **Q: ¿Un gateway central o uno por servidor?** → **Decision:** uno por servidor. Cada servidor corre su gateway, su `herdr-tap` y sus hooks. La app guarda una lista de gateways y se conecta a cada uno. Ejemplo: si `gx10` se apaga, `mylinux` sigue visible y `gx10` aparece como desconectado.
6. **Q: ¿Notificaciones push sin cuentas de Apple ni Firebase?** → **Decision:** historias 17 (gateway) y 18 (app), bloqueadas (`🚧`) hasta que el usuario tenga Apple Developer de pago y un proyecto de Firebase. Mientras tanto, la v1 avisa solo con la app abierta.

## Affected Files

### Frontend

- [ ] Pattern to follow: `apps/mobile/src/features/workspace/WorkspaceThreadSidebar.tsx` y `apps/mobile/src/features/threads/ThreadNavigationSidebar.tsx` (lista de la barra lateral, secciones, filas). Layout: `AdaptiveWorkspaceLayout` (`apps/mobile/src/features/layout/AdaptiveWorkspaceLayout.tsx`, envuelto por `RootStackLayout` en `Stack.tsx`) y su contexto `useAdaptiveWorkspaceLayout`; rol del panel con `usePaneRole()` de `apps/mobile/src/components/ui/Pane.tsx` (`"sidebar"` | `"content"` | …). Avisos: `Toast` de `apps/mobile/src/components/ui/Toast.tsx` (`message`, `description`, `tone`, `action`, `dismissMode: "swipe"`) dentro de `OverlayPortal` (`components/ui/OverlayPortal.tsx`; su host ya está montado en `App.tsx`).
- [ ] Conventions to follow: estado con átomos de `@effect/atom-react` o hooks de React con estado local donde baste; `PascalCase.tsx` para componentes y `camelCase.ts` para lógica; estilos con clases Uniwind (`className`) y `useThemeColor`; símbolos con `AppSymbol`; texto con `AppText`; `ListSection`, `ListRow`, `StatusPill`, `RelativeTime` (`components/RelativeTime.tsx`) y demás de `components/ui`; colores de estado de agente `working`, `blocked`, `done`, `idle` como en el resto de la app; pruebas `*.test.ts` junto al archivo.
- [ ] `apps/mobile/src/features/herdr/HerdrHomeRoute.tsx` — reemplazar la lista simple.
- [ ] `apps/mobile/src/features/herdr/HerdrAgentList.tsx` — nuevo.
- [ ] `apps/mobile/src/features/herdr/HerdrSpaceTree.tsx` — nuevo.
- [ ] `apps/mobile/src/features/herdr/HerdrSidebar.tsx` — nuevo.
- [ ] `apps/mobile/src/features/herdr/useAgentAlerts.ts` — nuevo (incluye el componente `HerdrAgentAlerts` o se exporta desde `HerdrHomeRoute.tsx`).
- [ ] `apps/mobile/src/features/herdr/agentPriority.ts` — nuevo: lógica pura de orden y transiciones (para probarla).
- [ ] `apps/mobile/src/features/layout/AdaptiveWorkspaceLayout.tsx` (archivo de T3) — cambio mínimo de la barra lateral, solo si hace falta (ver Step 4).
- [ ] `docs/herdr/patch-surface.txt` — agregar el archivo del layout con su motivo, si se toca.
- [ ] Prueba: `apps/mobile/src/features/herdr/agentPriority.test.ts`.

## Implementation Steps

### Step 1: `agentPriority.ts` (lógica pura)

- Un agente es una terminal cuyo `status` no es nulo. `interface HerdrAgentRow { hostId; hostName; address; spaceId; spaceName; tabId; tabName; app; state; kind; msg; changedAtMs }` construido desde `HerdrMachineView[]` (nombres de espacio y pestaña desde `snapshot.spaces`).
- `const STATE_RANK = { blocked: 0, error: 0, done: 1, working: 2, idle: 3 }`; `error` (estado de herdr-core) va con `blocked`: necesita atención y avisa. Cualquier otro estado desconocido = 4.
- `sortAgents(rows)`: por rango y luego por `changedAtMs` descendente.
- `groupAgents(rows)`: `{ attention: blocked + error + done, working, other: idle + resto }`.
- `detectAlertTransitions(prev: Map<string, string | null>, rows): HerdrAgentRow[]`: devuelve los agentes cuyo estado **cambió** a `blocked`, `error` o `done` respecto del mapa anterior (clave `hostId/address`).
- **Línea base por host:** el primer snapshot de un host después de cada (re)conexión —es decir, el primero que llega tras pasar su `state` de no conectado a `connected`— solo fija la línea base de ese host y no genera avisos. Desde ahí, se avisa de cada transición, incluidas las que traiga un `snapshot` completo por `missed` mientras la conexión sigue (no es reconexión) y un agente nuevo que aparece ya en `blocked`. Quitar un host borra su línea base; volver a agregarlo empieza de nuevo. Exportar `createAlertTracker()` con `update(machines: HerdrMachineView[]): HerdrAgentRow[]` que encapsula esta regla para probarla.

### Step 2: `HerdrAgentList.tsx`

Secciones "Necesitan atención" (blocked y done) y "Trabajando"; los demás agentes en una sección "Otros agentes" plegada por defecto. Cada fila: color/píldora del estado, nombre del espacio y de la pestaña, `app` (`claude-code`, `codex`, …), `msg` del estado (una línea), servidor (si hay más de uno) y hace cuánto (`RelativeTime` con `changedAtMs`). Tocar → `navigation.navigate("HerdrAgent", { hostId, address })`. Agentes de un servidor "Sin conexión" se muestran atenuados.

### Step 3: `HerdrSpaceTree.tsx`

Árbol servidor → carpeta → espacio → pestaña:

- Encabezado del servidor con su estado (`Conectando…`, `Sin conexión` + hora del último dato) y el menú "Quitar servidor" (de la historia 7).
- Carpetas (`snapshot.folders`, en su orden, con `color` si viene) plegables; estado inicial = `folder.folded`; el plegado es local a la app (no se escribe al servidor). Los espacios sin carpeta van después de las carpetas.
- Espacio: nombre y, al desplegarlo, sus pestañas (nombre; punto de color del estado si alguna terminal de la pestaña es agente). Acciones del espacio: "+" → `navigation.navigate("HerdrNewTab", { hostId, spaceId })` (la hoja la implementa la historia 9) y "Archivos" → `navigation.navigate("HerdrFiles", { hostId, path: space.cwd })` (la pantalla la implementa la historia 10).
- Tocar una pestaña → `HerdrAgent` con la terminal del primer panel (`tab.panes[0].terminal`).

### Step 4: `HerdrSidebar.tsx`, `HerdrHomeRoute.tsx` y el layout de iPad

- `HerdrSidebar`: `HerdrAgentList` + `HerdrSpaceTree` en un solo scroll compacto, con el botón "Agregar servidor".
- `HerdrHomeRoute`: en teléfono, la misma composición a pantalla completa (agentes arriba, árbol debajo); monta `HerdrAgentAlerts` (Step 5).
- **iPad, hallazgo de la planificación (verificar):** en el layout de T3 la ruta `Home` se dibuja como barra lateral junto a las páginas (`features/layout/createWorkspaceStackNavigator.android.tsx`: "Home is the sidebar beside the pages"; en iOS, `createWorkspaceStackNavigator.ios.tsx` usa `props.primary && layout.usesSplitView`). Como la historia 7 ya puso `HerdrHomeRoute` en `Home`, la barra lateral en iPadOS 26 y Android ancho ya es Herdr: en `HerdrHomeRoute`, si `usePaneRole() === "sidebar"`, renderizar `HerdrSidebar`. Solo cuando no hay ruta de barra lateral (iPadOS anterior a 26), `AdaptiveWorkspaceLayout.tsx` dibuja su respaldo en la línea ~1088: `{sidebar ?? <WorkspaceThreadSidebar />}`. El cambio más pequeño posible es reemplazar ese respaldo por `<HerdrSidebar />` (un import y una línea; el import de `WorkspaceThreadSidebar` se quita solo si queda sin uso; el archivo `WorkspaceThreadSidebar.tsx` no se toca). Hacerlo solo si se comprueba que ese respaldo se usa en algún dispositivo soportado; si no, no tocar el archivo y anotarlo en el tracker.
- Si se toca `AdaptiveWorkspaceLayout.tsx` (o, si resultara más pequeño, `WorkspaceThreadSidebar.tsx`), agregar esa ruta a `docs/herdr/patch-surface.txt` en el mismo commit con el comentario `# Barra lateral de iPad muestra HerdrSidebar (historia 8)`.
- Contenido e inspector de iPad: el contenido es la página abierta (`HerdrAgent`, historia 9) y el inspector de archivos lo registra la historia 10; aquí no se hace nada más.
- Las secciones y filas de T3 que no usamos (Chats, PRs, Usage, proyectos) no se muestran ni se borran.

### Step 5: `useAgentAlerts.ts`

- `useAgentAlerts(): { alert: HerdrAgentRow | null; dismiss(): void }` sobre `useHerdrSnapshot()`, guardando un `createAlertTracker()` en un `useRef` y llamando `update` en cada cambio del modelo (efecto con `// oxlint-disable-next-line t3code/no-unreasoned-state-sync -- compara con el estado anterior del modelo en vivo`). Si llegan varios, mostrar el más reciente.
- `HerdrAgentAlerts` (componente): `OverlayPortal` + `Toast` con `message` = `"<espacio>: necesita atención"` (blocked), `"<espacio>: error"` (error) o `"<espacio>: terminó"` (done), `description` = `msg` del estado, `tone` `danger` (blocked, error) o `success` (done), `dismissMode: "swipe"` y `action: { label: "Abrir", onPress: () => navigation.navigate("HerdrAgent", { hostId, address }) }`. Se cierra solo a los 6 s.
- No avisar del agente que el usuario ya tiene abierto en pantalla (ruta actual `HerdrAgent` con el mismo `hostId`/`address`).
- Solo con la app abierta (decisión 13): nada de notificaciones del sistema ni `expo-notifications`.

### Step 6: pruebas (`agentPriority.test.ts`)

Orden por rango y por `changedAtMs`; agrupación; terminal con `status: null` no es agente; `detectAlertTransitions`: `working → blocked` avisa, `blocked → blocked` no, `idle → done` avisa, `working → error` avisa y `error` se ordena con `blocked`; con `createAlertTracker`: primer snapshot tras conectar no avisa, el primero tras una reconexión (`offline → connected`) tampoco, un `snapshot` por `missed` sin reconexión sí avisa de lo que cambió, agente nuevo que aparece ya en `blocked` después de la línea base sí avisa, quitar y volver a agregar un host reinicia la línea base.

## Out of Scope

- Terminal en vivo y la hoja de pestaña nueva (`HerdrAgentRoute`, `HerdrNewTabSheet`): historia 9. Aquí solo se navega a ellas.
- Pantalla de archivos e inspector de archivos de iPad: historia 10.
- Notificaciones push con la app cerrada: historias 17 y 18 (bloqueadas).
- Crear, renombrar o eliminar espacios; renombrar o cerrar pestañas; crear o editar carpetas de space-folders: fuera del alcance.
- Live Activities, widgets y la pantalla de uso de T3.
- Editar `Stack.tsx` o `herdrRoutes.tsx`: las rutas ya existen (historia 7).

## Acceptance Criteria

- [ ] Dentro de `apps/mobile`: `vp test run src/features/herdr/agentPriority.test.ts` pasa.
- [ ] Dentro de `apps/mobile`: `vp run typecheck` pasa.
- [ ] Ningún archivo de T3 cambia salvo, si hizo falta, el archivo del layout de la barra lateral, y en ese caso está en `docs/herdr/patch-surface.txt` con su motivo; `node scripts/herdr/check-patch-surface.ts` sale con código 0.
- [ ] Con un gateway real: un agente de Claude Code que pide permiso en el escritorio aparece en "Necesitan atención" y, con la app abierta en otra pantalla, sale el aviso que lleva a ese agente. (Verificación en Android en `mylinux`; anotar en el tracker. En iPad se verifica a mano en la Mac del usuario.)
- [ ] Las carpetas de `~/.config/herdr-space-folders.json` del servidor aparecen en el árbol, plegables.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
