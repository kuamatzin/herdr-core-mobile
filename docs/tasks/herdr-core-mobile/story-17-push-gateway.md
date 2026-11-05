---
status: not_started
feature: "herdr-core-mobile"
story: "17"
layers:
  - "backend"
dependencies:
  - "story-04"
---

# Story 17: Push en el gateway (🚧 bloqueada hasta tener cuentas)

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

> **🚧 Bloqueada.** No empezar hasta que el usuario tenga una cuenta de Apple Developer de pago (llave APNs `.p8`, Team ID) y un proyecto de Firebase (cuenta de servicio para FCM HTTP v1). Mientras tanto, la app avisa solo con la app abierta (historia 8).

## Objective

Este repo es un fork de T3 Code cuya app móvil abre en Herdr y se conecta a un gateway por servidor (`apps/herdr-gateway`). Con la app abierta ya hay avisos cuando un agente pasa a `blocked` o `done`. Esta historia hace la mitad del gateway de los avisos **con la app cerrada**: el registro y borrado de dispositivos, el notificador que detecta las transiciones a `blocked` o `done`, y el envío por APNs (iOS) y FCM (Android), con las credenciales guardadas de forma segura. La mitad de la app (token del dispositivo, registro en cada gateway, abrir el agente al tocar) es la historia 18.

## Dependencies

- Depends on: Story 04 — Gateway base (stubs `apps/herdr-gateway/src/routes/devices.ts` con `registerDeviceRoutes(router, ctx)` y `apps/herdr-gateway/src/push/notifier.ts` con `startPushNotifier(ctx): { close(): void }`, ya registrados por `main.ts`; `Router` con `post` y `delete` en `http.ts`; `snapshot.onMessage` con los mensajes `snapshot`, `terminal`, `terminalRemoved`, `layout`; tipo `DeviceRegistration` en `packages/herdr/src/gateway-api.ts`; `EnvironmentFile=-%h/.config/herdr-gateway/push.env` ya en el unit de systemd).
- Bloqueo externo: Apple Developer de pago + proyecto de Firebase.
- Esta historia **no** edita `main.ts`, `http.ts`, `install.sh` ni `gateway-api.ts` (en la ola 3 los editan las historias 5, 11 y 12).

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Un gateway central o uno por servidor?** → **Decision:** uno por servidor. Cada servidor corre su gateway, su `herdr-tap` y sus hooks. La app guarda una lista de gateways y se conecta a cada uno. Ejemplo: si `gx10` se apaga, `mylinux` sigue visible y `gx10` aparece como desconectado.
4. **Q: ¿Seguridad del gateway?** → **Decision:** solo Tailscale. El gateway escucha únicamente en la IP de Tailscale del servidor (`tailscale ip -4`), nunca en `0.0.0.0`. Además, en cada conexión HTTP y WebSocket consulta `tailscale whois --json <ip>:<puerto>` del cliente y acepta solo si `UserProfile.LoginName` coincide con el dueño del propio nodo. Si no coincide, responde `403`. Ejemplo verificado: la MacBook del usuario resuelve a `kuamatzin@gmail.com`, igual que `mylinux`. Un dispositivo de las socias de Protact sería rechazado aunque vea el puerto.
5. **Q: ¿Notificaciones push sin cuentas de Apple ni Firebase?** → **Decision:** historias 17 (gateway) y 18 (app), bloqueadas (`🚧`) hasta que el usuario tenga Apple Developer de pago y un proyecto de Firebase. Mientras tanto, la v1 avisa solo con la app abierta.
6. **(plan) Gateway sin Effect.** El gateway y `packages/herdr` usan TypeScript plano con Promises y APIs de Node, no Effect, para no acoplarse a la evolución de las librerías del servidor T3. `packages/herdr` no importa nada de `@t3tools/*`.

Access Control (binding): credenciales de push en `~/.config/herdr-gateway/push.env` (modo 600), cargadas con `EnvironmentFile=`; tokens y llaves nunca se registran.

## Affected Files

### Backend

- [ ] Pattern to follow: `infra/relay/src/agentActivity/ApnsClient.ts` (APNs con llave `.p8`, JWT ES256 y HTTP/2; solo como referencia, no importar: es Effect) y `apps/server/src/relay/`. El router y el contexto de `apps/herdr-gateway/src/http.ts`.
- [ ] Conventions to follow: TypeScript plano con `node:http2`, `node:crypto`, `node:fs` (decisión 14), sin SDKs de Firebase (FCM HTTP v1 con un JWT RS256 de la cuenta de servicio firmado con `node:crypto` e intercambiado por un token OAuth); errores `{ "error": { "kind", "message" } }`; sin `enum`/`namespace`; pruebas `*.test.ts` con clientes HTTP falsos, temporizadores y reloj inyectados.
- [ ] `apps/herdr-gateway/src/push/devices.ts` — nuevo: almacén de dispositivos.
- [ ] `apps/herdr-gateway/src/push/apns.ts` — nuevo.
- [ ] `apps/herdr-gateway/src/push/fcm.ts` — nuevo.
- [ ] `apps/herdr-gateway/src/push/notifier.ts` — reemplazar el stub.
- [ ] `apps/herdr-gateway/src/routes/devices.ts` — reemplazar el stub.
- [ ] Pruebas: `apps/herdr-gateway/src/push/devices.test.ts`, `apps/herdr-gateway/src/push/notifier.test.ts`, `apps/herdr-gateway/src/push/apns.test.ts`, `apps/herdr-gateway/src/push/fcm.test.ts`.

## Implementation Steps

### Step 1: contrato

Del API Contract (binding):

**`POST /v1/devices`** (historia 17) `{ "platform": "ios" | "android", "token": "…", "environment": "sandbox" | "production" }` → `204`.
**`DELETE /v1/devices/:token`** (historia 17; `token` codificado en la URL, máximo 4096 caracteres) → `204`; `404 {"error":{"kind":"not_found"}}` si no está registrado.

Ambas pasan por la autorización de Tailscale de la decisión 11 que ya aplica `createGatewayServer`. Database changes (binding): el estado del gateway vive en memoria, salvo los dispositivos push en `~/.config/herdr-gateway/devices.json`.

### Step 2: `push/devices.ts` (almacén)

- Archivo `~/.config/herdr-gateway/devices.json` (carpeta 0700, archivo 0600): `[{ platform, token, environment, registeredAtMs }]`. Escritura atómica (temporal 0600 + `rename`), serializada en una cola de promesas dentro del proceso (un solo gateway escribe el archivo; no hace falta bloqueo entre procesos).
- `upsert(device)`: **sin duplicar por `token`** (el mismo token reemplaza su registro y actualiza `registeredAtMs`; la app lo vuelve a registrar en cada arranque, historia 18). `remove(token): boolean`. `list()`.
- Retención: al cargar y una vez al día, se borran los dispositivos con `registeredAtMs` de hace más de 90 días (la app no volvió a registrarlos).
- Validación: `platform` y `environment` de los valores del contrato; `token` de 1–4096 caracteres imprimibles; si no → `400 bad_request`.

### Step 3: `push/apns.ts` y `push/fcm.ts`

- **Credenciales:** variables de `~/.config/herdr-gateway/push.env` (modo 600; el servicio ya lo carga con `EnvironmentFile=` desde la historia 4): `HERDR_APNS_KEY_PATH`, `HERDR_APNS_KEY_ID`, `HERDR_APPLE_TEAM_ID`, `HERDR_APNS_TOPIC` (bundle ID) y `HERDR_FCM_SERVICE_ACCOUNT` (ruta al JSON). Al arrancar, si `push.env`, la llave `.p8` o el JSON de la cuenta de servicio tienen permisos más abiertos que 600, registrar un aviso (sin el contenido). Si faltan las variables de una plataforma o la credencial es inválida, esa plataforma se desactiva con **un** aviso en el log y el gateway sigue.
- **Nunca** se registran tokens de dispositivo, llaves, JWT ni tokens OAuth: en los logs, un token se muestra como sus últimos 6 caracteres.
- **APNs:** JWT ES256 renovado cada 50 min; `POST /3/device/<token>` por HTTP/2 a `api.sandbox.push.apple.com` (`environment: "sandbox"`) o `api.push.apple.com` (`"production"`); headers `apns-topic`, `apns-push-type: alert`, `apns-priority: 10`, `apns-collapse-id` = `<address>:<turn>:<state>`. Respuesta `410` o `400 BadDeviceToken` → `devices.remove(token)`.
- **FCM HTTP v1:** token OAuth con JWT RS256 (`scope: https://www.googleapis.com/auth/firebase.messaging`) cacheado hasta 1 min antes de vencer; `POST https://fcm.googleapis.com/v1/projects/<project_id>/messages:send`. `404` con `UNREGISTERED` → `devices.remove(token)`.
- Errores transitorios (`429`, `5xx`, red): un reintento con espera de 2 s; si vuelve a fallar, se descarta ese aviso (no hay cola persistente).

### Step 4: `push/notifier.ts`

- Mantener `export function startPushNotifier(ctx: GatewayContext): { close(): void }`. Se suscribe a `ctx.snapshot.onMessage` y mantiene el último estado por terminal.
- **Un aviso por (terminal, turno, estado):** al llegar un mensaje `terminal` cuyo `status.state` pasa a `blocked` o `done`, la clave `<address>:<turn>:<state>` se avisa una sola vez (se recuerdan las últimas 1 000 claves). Un mensaje `snapshot` (conexión con el core o `missed`) solo actualiza el estado conocido y nunca avisa. Cada gateway avisa solo de sus propias terminales, así que no hay duplicados entre servidores.
- Notificación: **título = nombre del espacio** de la terminal (de `snapshot.spaces`), **cuerpo = `msg` del estado** (o "Necesita atención" / "Terminó" si no hay `msg`), datos `{ hostCoreId: machine.coreId, address }`, a todos los dispositivos registrados.
- `close()` suelta la suscripción y los temporizadores.

### Step 5: `routes/devices.ts`

Mantener `export function registerDeviceRoutes(router: Router, ctx: GatewayContext): void`: `post /v1/devices` → `devices.upsert` → `204`; `delete /v1/devices/:token` (decodificado; más de 4096 caracteres → `400`) → `204` o `404 not_found`. El almacén es un único objeto compartido con el notificador (módulo `push/devices.ts` con una instancia creada al primer uso).

### Step 6: pruebas

- `devices.test.ts` (carpeta temporal): `upsert` dos veces con el mismo token deja uno; `remove`; archivo con modo 0600; vencidos a los 90 días se borran (reloj falso).
- `notifier.test.ts`: `working → blocked` avisa una vez; el mismo `(address, turn, state)` repetido no vuelve a avisar; un `snapshot` no avisa; título y cuerpo correctos.
- `apns.test.ts` / `fcm.test.ts` con clientes falsos: `410`/`BadDeviceToken` y `UNREGISTERED` borran el dispositivo; plataforma sin credenciales queda desactivada; ningún token completo aparece en lo que se registra (logger falso).

### Step 7: documentación

Agregar a `docs/herdr/mobile-build.md` (historia 2) una sección "Push en el gateway": cómo crear `~/.config/herdr-gateway/push.env` con modo 600, qué variables lleva y que hay que reiniciar el servicio.

## Out of Scope

- Token del dispositivo, registro desde la app, capacidades de push en `app.config.ts` y abrir el agente al tocar: historia 18.
- Editar `main.ts`, `http.ts`, `install.sh` o `gateway-api.ts`: ya los dejó listos la historia 4.
- Live Activities y widgets de T3: fuera del alcance.
- Un servicio central de push o un relay: cada gateway manda sus avisos (decisión 9).
- Avisos de estados distintos de `blocked` y `done`.
- Publicar en App Store o Google Play.

## Acceptance Criteria

- [ ] Dentro de `apps/herdr-gateway`: `vp test run src/push/devices.test.ts src/push/notifier.test.ts src/push/apns.test.ts src/push/fcm.test.ts` pasa.
- [ ] Dentro de `apps/herdr-gateway`: `vp run typecheck` pasa.
- [ ] `git diff --name-only` de la historia solo toca `apps/herdr-gateway/src/push/`, `apps/herdr-gateway/src/routes/devices.ts`, sus pruebas y `docs/herdr/mobile-build.md`.
- [ ] `POST /v1/devices` responde `204` y escribe `~/.config/herdr-gateway/devices.json` con modo `0600`; `DELETE` de un token desconocido responde `404`; sin autorización de Tailscale, `403`.
- [ ] Con las cuentas configuradas: un agente que pasa a `blocked` en el escritorio produce una notificación en un dispositivo registrado a mano (token de prueba) por APNs y por FCM, con el nombre del espacio como título. Anotar el resultado en el tracker.
- [ ] `node scripts/herdr/check-patch-surface.ts` sale con código 0.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
