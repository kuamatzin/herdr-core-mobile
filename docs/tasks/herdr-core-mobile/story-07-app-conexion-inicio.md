---
status: not_started
feature: "herdr-core-mobile"
story: "07"
layers:
  - "frontend"
dependencies:
  - "story-04"
---

# Story 07: App, conexión e inicio

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code cuya app móvil (`apps/mobile`, Expo / React Native) se conecta a herdr-core a través de un gateway por servidor (historia 4). Esta historia hace que la app abra en Herdr: guarda la lista de servidores (URL del gateway en Tailscale), se conecta a todos a la vez, mantiene un modelo en vivo por servidor (snapshot + eventos), muestra una lista simple de espacios y pestañas por servidor, permite agregar un servidor, y reserva las rutas `HerdrAgent`, `HerdrFiles` y `HerdrNewTab` (y el componente `HerdrFilesInspector`) con pantallas temporales que las historias 9 y 10 reemplazan editando solo sus propios archivos.

## Dependencies

- Depends on: Story 04 — Gateway base (`GET /v1/health`, `GET /v1/snapshot`, `WS /v1/events`; tipos en `@herdr/core-client/gateway-api`: `GatewayHealth`, `GatewaySnapshot`, `GatewayEventMessage`, `GatewayError`).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Qué se ve al abrir?** → **Decision:** Herdr es la pantalla principal (ruta `Home`). Las pestañas de T3 no aparecen.
4. **Q: ¿Un gateway central o uno por servidor?** → **Decision:** uno por servidor. Cada servidor corre su gateway, su `herdr-tap` y sus hooks. La app guarda una lista de gateways y se conecta a cada uno. Ejemplo: si `gx10` se apaga, `mylinux` sigue visible y `gx10` aparece como desconectado.

`apps/mobile/src/Stack.tsx`, `apps/mobile/package.json` y `pnpm-lock.yaml` están en `docs/herdr/patch-surface.txt` (historia 1).

## Affected Files

### Frontend

- [ ] Pattern to follow: `apps/mobile/src/features/connection/ConnectionsNewRouteScreen.tsx` para la hoja de agregar (formulario, validación, errores); `createNativeStackScreen` y las rutas de hoja como `ConnectionsNew` en `apps/mobile/src/Stack.tsx` (línea ~974: `presentation` de form sheet, `sheetAllowedDetents`, `sheetGrabberVisible`); parámetros de ruta con `StaticScreenProps<...>` y `useNavigation()` de `@react-navigation/native` como en `apps/mobile/src/features/terminal/ThreadTerminalRouteScreen.tsx`.
- [ ] Conventions to follow: estado con átomos de `@effect/atom-react` como el resto de `apps/mobile/src/state`, o hooks de React con estado local donde baste; nombres de archivo como los vecinos (`PascalCase.tsx` para componentes, `camelCase.ts` para lógica); estilos con clases Uniwind (`className`) y `useThemeColor`; símbolos con `AppSymbol`; texto con `AppText`; componentes de `apps/mobile/src/components/ui` (`ListSection`, `ListRow`, `Button`, `ErrorBanner`, `EmptyState`, `StatusPill`); pruebas `*.test.ts` junto al archivo. Los efectos de React llevan `// oxlint-disable-next-line t3code/no-unreasoned-state-sync -- <por qué>` solo cuando sincronizan con algo externo (WebSocket, SecureStore).
- [ ] `apps/mobile/src/features/herdr/state/hosts.ts` — nuevo.
- [ ] `apps/mobile/src/features/herdr/state/gatewayClient.ts` — nuevo.
- [ ] `apps/mobile/src/features/herdr/state/useHerdrSnapshot.ts` — nuevo.
- [ ] `apps/mobile/src/features/herdr/HerdrHomeRoute.tsx` — nuevo (la historia 8 lo completa).
- [ ] `apps/mobile/src/features/herdr/HerdrHostSheet.tsx` — nuevo.
- [ ] `apps/mobile/src/features/herdr/herdrRoutes.tsx` — nuevo.
- [ ] `apps/mobile/src/features/herdr/HerdrAgentRoute.tsx` — nuevo, pantalla temporal (la historia 9 la reemplaza).
- [ ] `apps/mobile/src/features/herdr/HerdrFilesRoute.tsx` — nuevo, pantalla temporal (la historia 10 la reemplaza).
- [ ] `apps/mobile/src/features/herdr/HerdrNewTabSheet.tsx` — nuevo, hoja temporal (la historia 9 la reemplaza).
- [ ] `apps/mobile/src/features/herdr/HerdrFilesInspector.tsx` — nuevo, componente temporal que devuelve `null` (la historia 10 lo reemplaza; la historia 9 lo monta en `HerdrAgentRoute`).
- [ ] `apps/mobile/src/Stack.tsx` — `Home` usa `HerdrHomeRoute` y un spread de `herdrScreens`.
- [ ] `apps/mobile/package.json` — dependencia `@herdr/core-client: workspace:*`.
- [ ] Pruebas: `apps/mobile/src/features/herdr/state/useHerdrSnapshot.test.ts`, `apps/mobile/src/features/herdr/state/hosts.test.ts`.

## Implementation Steps

### Step 1: dependencia

`apps/mobile/package.json`: agregar `"@herdr/core-client": "workspace:*"` a `dependencies`; `vp i` en la raíz. La app solo importa `@herdr/core-client/gateway-api` (tipos). Nunca `@herdr/core-client/hc`.

### Step 2: `state/hosts.ts`

- `interface HerdrHost { id: string; url: string; name: string }`. `id` = `machine.coreId` de `/v1/health` (estable aunque cambie la URL; evita duplicar un servidor).
- Guardado en `expo-secure-store` (ya es dependencia) con la clave `herdr.hosts.v1` y valor JSON `[{id,url,name}]`. JSON inválido → lista vacía.
- `normalizeGatewayUrl(input: string): string | null` (función pura): recorta espacios, agrega `http://` si no tiene esquema, acepta solo `http:`/`https:` y **solo** `scheme://host[:port]`: una `/` final se quita, pero cualquier otra ruta, query o fragmento (`http://x:7788/algo`) → `null` (se rechaza, no se recorta); `null` si no es URL.
- Las escrituras a SecureStore se serializan en **una sola cola de promesas** (cada `upsert`/`remove` espera a la anterior, lee el valor vigente en memoria, escribe y recién entonces publica el nuevo estado); si una escritura falla, el estado en memoria vuelve al último valor guardado y el error se devuelve a quien llamó. El store no publica hosts hasta terminar la primera lectura (`ready`).
- Si un servidor se reinstala y cambia su `coreId`, al agregarlo de nuevo queda como host nuevo; el anterior aparece "Sin conexión" hasta que el usuario lo quita.
- API: `loadHosts()`, `upsertHost(host)` (reemplaza por `id`), `removeHost(id)`, y un hook `useHerdrHosts(): { hosts: HerdrHost[]; ready: boolean; upsert; remove }` sobre un store de módulo (`useSyncExternalStore`) o un átomo, para que Home, la hoja y la historia 8 vean el mismo valor.
- `hosts.test.ts`: `normalizeGatewayUrl` (`100.1.2.3:7788` → `http://100.1.2.3:7788`; `http://x:7788/` → `http://x:7788`; `http://x:7788/v1/` → `null`; `http://x:7788?a=1` → `null`; `ftp://x` → `null`; vacío → `null`); y la cola: dos `upsert` lanzados a la vez terminan con ambos hosts guardados (SecureStore falso en memoria).

### Step 3: `state/gatewayClient.ts`

- `createGatewayClient(baseUrl: string)` con `getJson<T>(path)`, `postJson<T>(path, body)`, `wsUrl(path, query?)` (cambia `http`→`ws`, `https`→`wss`) — las historias 9, 10 y 14 los reutilizan. Respuesta no‑2xx → lanza `GatewayHttpError { status, kind }` (leyendo `{ error: { kind } }` si existe).
- `health(): Promise<GatewayHealth>`; valida la forma (`gatewayVersion` string, `machine.coreId` string, `hc.version` string); si no cumple → `GatewayHttpError(0, "not_a_gateway")`.
- `connectEvents({ onMessage(m: GatewayEventMessage), onState(s: { state: "connecting" | "connected" | "offline"; lastDataAtMs: number | null }) }): () => void`: abre `WebSocket(wsUrl("/v1/events"))`; reconecta con espera creciente 1, 2, 4, 8, 16, 30, 30… s (vuelve a 1 s al recibir el primer mensaje); `connected` al recibir el primer `snapshot`; `offline` al cerrarse o si no llega ningún mensaje en 45 s (tres heartbeats) — en ese caso cerrar y reconectar. `lastDataAtMs` = hora del último mensaje recibido. La función devuelta cierra y detiene los reintentos.

### Step 4: `state/useHerdrSnapshot.ts`

- Reducer puro exportado `applyGatewayEvent(current: GatewaySnapshot | null, m: GatewayEventMessage): GatewaySnapshot | null`: `snapshot` → reemplaza todo (así un `missed` del core rehace el modelo sin perder la navegación: las rutas usan `hostId` y `address`, no índices); `terminal` → alta o reemplazo por `address`; `terminalRemoved` → quita; `layout` → reemplaza `spaces` y `folders`; `heartbeat` → solo `seq`. Sin snapshot previo, los mensajes que no son `snapshot` se ignoran.
- Store de módulo `herdrConnections` (con `useSyncExternalStore`) que por cada host guardado mantiene `{ host, state, lastDataAtMs, snapshot }`. Abre las conexiones al montarse el primer suscriptor y las cierra 30 s después de que se va el último (si llega un suscriptor antes, ese temporizador se cancela; un host quitado se desconecta de inmediato y uno agregado se conecta de inmediato); al pasar la app a segundo plano (`AppState` distinto de `active`) cierra los WebSockets y al volver los reabre. Un host que falla no afecta a los demás (decisión 9).
- `useHerdrSnapshot(): ReadonlyArray<HerdrMachineView>` donde `HerdrMachineView = { host: HerdrHost; state: "connecting" | "connected" | "offline"; lastDataAtMs: number | null; snapshot: GatewaySnapshot | null }`, en el orden de la lista de hosts. Exportar también `useHerdrMachine(hostId)` y `useHerdrTerminal(hostId, address)` (las historias 8–10 las usan).
- `useHerdrSnapshot.test.ts`: `applyGatewayEvent` para cada tipo de mensaje, incluida la secuencia snapshot → terminal → terminalRemoved → layout → snapshot (missed) y el ignorar mensajes sin snapshot previo.

### Step 5: `HerdrHostSheet.tsx`

Hoja (ruta `HerdrAddHost`, form sheet) con un campo para pegar la URL (`http://100.x.y.z:7788`), botón "Agregar". Al enviar: `normalizeGatewayUrl`; `createGatewayClient(url).health()`; si falla la red, la forma o la URL → mostrar exactamente **"No es un gateway de Herdr"** y **no** guardar; si responde, mostrar el nombre del servidor (`machine.name`) y guardar `{ id: machine.coreId, url, name: machine.name }` con `upsertHost`, y cerrar. Mientras valida, botón deshabilitado con indicador.

### Step 6: `HerdrHomeRoute.tsx` (versión simple)

- Por cada `HerdrMachineView`: encabezado con el nombre del servidor y su estado: `Conectando…`, conectado (sin texto), o **`Sin conexión`** con la hora del último dato (`lastDataAtMs`, formato local `HH:mm`). Debajo, sus espacios (`snapshot.spaces`) y, por espacio, sus pestañas (`tabs`, nombre). Si un servidor está sin conexión se siguen mostrando sus últimos datos atenuados.
- Tocar una pestaña → `navigation.navigate("HerdrAgent", { hostId, address: tab.panes[0].terminal })`.
- Sin hosts: `EmptyState` con el botón "Agregar servidor". Con hosts: botón "Agregar servidor" en el encabezado o al final de la lista → `navigation.navigate("HerdrAddHost")`.
- Quitar un servidor: acción "Quitar servidor" en un menú del encabezado de cada servidor (`Menu` de `components/ui`) → `remove(id)` (camino de vuelta de "agregar").
- La historia 8 reemplaza este contenido por la lista de agentes y el árbol con carpetas; mantener los componentes pequeños.

### Step 7: `herdrRoutes.tsx` y pantallas temporales

```ts
export const herdrScreens = {
  HerdrAgent: createNativeStackScreen({
    screen: HerdrAgentRoute,
    linking: "herdr/:hostId/:address",
    options: { title: "" },
  }),
  HerdrFiles: createNativeStackScreen({
    screen: HerdrFilesRoute,
    linking: "herdr/:hostId/files",
    options: { title: "Archivos" },
  }),
  HerdrAddHost: createNativeStackScreen({
    screen: HerdrHostSheet,
    linking: "herdr/add-host",
    options: {
      title: "Agregar servidor",
      presentation: "formSheet",
      sheetAllowedDetents: [0.5],
      sheetGrabberVisible: true,
    },
  }),
  HerdrNewTab: createNativeStackScreen({
    screen: HerdrNewTabSheet,
    linking: "herdr/:hostId/spaces/:spaceId/new-tab",
    options: {
      title: "Pestaña nueva",
      presentation: "formSheet",
      sheetAllowedDetents: [0.4],
      sheetGrabberVisible: true,
    },
  }),
};
```

Estas cuatro rutas son todo el registro de rutas herdr del plan: las historias 8, 9 y 10 (ola 4, en paralelo) no editan `herdrRoutes.tsx` ni `Stack.tsx`; solo editan los archivos de pantalla que les tocan.

- `HerdrAgentRoute.tsx` (temporal): `export function HerdrAgentRoute(props: StaticScreenProps<{ hostId: string; address: string }>)` que muestra el nombre de la terminal y "Disponible en la siguiente versión". La historia 9 reemplaza el contenido de este archivo sin cambiar su export ni sus parámetros.
- `HerdrFilesRoute.tsx` (temporal): `export function HerdrFilesRoute(props: StaticScreenProps<{ hostId: string; path?: string }>)` con el mismo texto (`path` no está en el patrón de la ruta: React Navigation lo lleva como query `?path=`, codificado). La historia 10 lo reemplaza sin cambiar su export ni sus parámetros.
- `HerdrNewTabSheet.tsx` (temporal): `export function HerdrNewTabSheet(props: StaticScreenProps<{ hostId: string; spaceId: string }>)` con el mismo texto. La historia 9 lo reemplaza sin cambiar su export ni sus parámetros.
- `HerdrFilesInspector.tsx` (temporal): `export function HerdrFilesInspector(props: { hostId: string; path: string | null }): ReactNode` que por ahora devuelve `null`. La historia 10 lo reemplaza (registra el visor de archivos en el inspector de iPad con `Inspector` de `features/layout/AdaptiveWorkspaceLayout.tsx`) sin cambiar su export ni sus props.
- Las constantes de opciones de hoja de `Stack.tsx` (`HORIZONTAL_FORM_SHEET_OPTIONS`, `SHEET_GLASS_HEADER_OPTIONS`) son locales a ese archivo: no exportarlas; definir las opciones aquí.

### Step 8: `Stack.tsx` (único cambio a este archivo en todo el plan, salvo lo que la historia 8 justifique)

- En `screens` de la configuración raíz (línea ~582): `Home: createNativeStackScreen({ screen: HerdrHomeRoute, ... })` — cambiar **solo** `screen: WorkspaceRootScreen` por `screen: HerdrHomeRoute`; conservar `linking: ""`, `listeners` y `options`. `WorkspaceRootScreen` solo se usaba en esa línea (verificado), así que su import en `Stack.tsx` queda sin uso: quitar solo esa línea de import. El archivo `features/workspace/WorkspaceRootScreen.tsx` no se toca, sigue existiendo y compilando (lo importan sus propias pruebas).
- Agregar `...herdrScreens,` en el mismo objeto `screens`, inmediatamente después de `Home`.
- Imports: `HerdrHomeRoute` y `herdrScreens` desde `./features/herdr/...`. Nada más en este archivo.

## Out of Scope

- Lista de agentes por prioridad, árbol con carpetas, barra lateral de iPad y avisos con la app abierta: historia 8.
- Terminal en vivo y pestaña nueva: historia 9. Archivos: historia 10. Chat: historias 14 y 15. Push: historias 17 y 18.
- Conectar la app a servidores T3 normales: fuera del alcance; el código de T3 sigue ahí, sin probarse ni mantenerse.
- Usar `ssh` desde el teléfono: la app solo habla con el gateway.
- Mostrar el árbol de máquinas de `hc machine`.
- Borrar pantallas o código de T3 (Chats, PRs, Usage, proyectos): se ocultan al no ser el inicio.

## Acceptance Criteria

- [ ] Dentro de `apps/mobile`: `vp test run src/features/herdr/state/useHerdrSnapshot.test.ts src/features/herdr/state/hosts.test.ts` pasa.
- [ ] Dentro de `apps/mobile`: `vp run typecheck` pasa.
- [ ] `git diff apps/mobile/src/Stack.tsx` muestra solo: imports nuevos, la línea de import de `WorkspaceRootScreen` quitada, `screen: HerdrHomeRoute` en `Home` y la línea `...herdrScreens,`; ninguna ruta de T3 borrada.
- [ ] `grep -rn "@herdr/core-client/hc" apps/mobile/src` no encuentra nada.
- [ ] Con un gateway real en `mylinux`: agregar `http://<ip-tailscale>:7788` muestra `mylinux` con sus espacios y pestañas; una URL que no es gateway (o que trae una ruta) muestra "No es un gateway de Herdr" y no se guarda; al detener el gateway (`systemctl --user stop herdr-gateway`) el servidor pasa a "Sin conexión" con la hora del último dato y al arrancarlo vuelve solo. (Verificación en Android en `mylinux`; anotar el resultado en el tracker.)
- [ ] `node scripts/herdr/check-patch-surface.ts` sale con código 0.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
