---
status: not_started
feature: "herdr-core-mobile"
story: "18"
layers:
  - "frontend"
dependencies:
  - "story-08"
  - "story-17"
---

# Story 18: Push en la app (🚧 bloqueada hasta tener cuentas)

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

> **🚧 Bloqueada.** No empezar hasta que el usuario tenga una cuenta de Apple Developer de pago (Team ID, capacidad de push) y un proyecto de Firebase (`google-services.json`), y la historia 17 esté hecha.

## Objective

Este repo es un fork de T3 Code cuya app móvil (`apps/mobile`) abre en Herdr y se conecta a un gateway por servidor. El gateway ya sabe registrar dispositivos y mandar avisos por APNs y FCM (historia 17). Esta historia hace la mitad de la app: pedir permiso de notificaciones, obtener el token nativo del dispositivo, registrarlo en cada gateway guardado (y borrarlo al quitar un servidor), activar las capacidades de push en la configuración de Expo, y abrir el agente al tocar el aviso.

## Dependencies

- Depends on: Story 08 — App, lista de agentes e iPad (`HerdrHomeRoute.tsx`, donde se monta el registro; ruta `HerdrAgent` `{ hostId, address }`; `useHerdrHosts()` de la historia 7, con `id` = `coreId` del gateway; `createGatewayClient(baseUrl)` con `postJson` y `GatewayHttpError`).
- Depends on: Story 17 — Push en el gateway (`POST /v1/devices`, `DELETE /v1/devices/:token`; datos de la notificación `{ hostCoreId, address }`; tipo `DeviceRegistration` en `@herdr/core-client/gateway-api`).
- Bloqueo externo: Apple Developer de pago + proyecto de Firebase.

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Un gateway central o uno por servidor?** → **Decision:** uno por servidor. Cada servidor corre su gateway, su `herdr-tap` y sus hooks. La app guarda una lista de gateways y se conecta a cada uno. Ejemplo: si `gx10` se apaga, `mylinux` sigue visible y `gx10` aparece como desconectado.
4. **Q: ¿Notificaciones push sin cuentas de Apple ni Firebase?** → **Decision:** historias 17 (gateway) y 18 (app), bloqueadas (`🚧`) hasta que el usuario tenga Apple Developer de pago y un proyecto de Firebase. Mientras tanto, la v1 avisa solo con la app abierta.

`apps/mobile/app.config.ts` y `apps/mobile/app.config.test.ts` están en `docs/herdr/patch-surface.txt`.

## Affected Files

### Frontend

- [ ] Pattern to follow: `apps/mobile/src/features/agent-awareness/remoteRegistration.ts` (cómo T3 pide permisos y obtiene el token con `expo-notifications`; solo referencia, no se modifica). Configuración: la estructura existente de `app.config.ts` (ya incluye el plugin `expo-notifications` y `googleServicesFile` condicionado a `T3CODE_ANDROID_GOOGLE_SERVICES_FILE`).
- [ ] Conventions to follow: hooks de React con estado local o átomos de `@effect/atom-react`; `camelCase.ts` para lógica; efectos solo para sincronizar con el sistema de notificaciones, con su `oxlint-disable` razonado; pruebas `*.test.ts` junto al archivo; diff mínimo en `app.config.ts`.
- [ ] `apps/mobile/src/features/herdr/state/usePushRegistration.ts` — nuevo.
- [ ] `apps/mobile/src/features/herdr/state/pushRegistrationPlan.ts` — nuevo: lógica pura (qué hosts registrar o borrar).
- [ ] `apps/mobile/src/features/herdr/HerdrHomeRoute.tsx` — montar `usePushRegistration()`.
- [ ] `apps/mobile/app.config.ts` y `apps/mobile/app.config.test.ts` — capacidades de push condicionadas.
- [ ] `docs/herdr/mobile-build.md` — sección de push de la app.
- [ ] Prueba: `apps/mobile/src/features/herdr/state/pushRegistrationPlan.test.ts`.

## Implementation Steps

### Step 1: contrato

Del API Contract (binding): **`POST /v1/devices`** `{ "platform": "ios" | "android", "token": "…", "environment": "sandbox" | "production" }` → `204`. **`DELETE /v1/devices/:token`** (`token` codificado en la URL, máximo 4096 caracteres) → `204`; `404 {"error":{"kind":"not_found"}}` si no está registrado.

### Step 2: `app.config.ts`

Solo si `repoEnv.HERDR_PUSH_ENABLED === "1"`:

- iOS: `ios.entitlements["aps-environment"] = repoEnv.HERDR_APNS_ENVIRONMENT ?? "development"` (requiere `HERDR_APPLE_TEAM_ID` de una cuenta de pago: Personal Team no firma push).
- Android: `android.googleServicesFile = repoEnv.HERDR_ANDROID_GOOGLE_SERVICES_FILE` (ruta al `google-services.json` del proyecto de Firebase, fuera del repo; nunca se versiona).
  Sin la variable, la configuración queda igual que tras la historia 2. Ajustar `app.config.test.ts` con un caso sin `HERDR_PUSH_ENABLED` (sin `aps-environment` ni `googleServicesFile`) y uno con `HERDR_PUSH_ENABLED=1` (ambos presentes), ampliando el mock de `loadRepoEnv` solo con esas claves.

### Step 3: `pushRegistrationPlan.ts` (puro)

`planPushRegistrations(input: { hosts: HerdrHost[]; registered: Record<hostId, token>; token: string | null }): { register: HerdrHost[]; unregister: Array<{ hostId; url; token }> }`: registrar en cada host guardado donde ese token no está registrado (o cambió el token); borrar en los hosts que ya no están en la lista. `pushRegistrationPlan.test.ts`: host nuevo → registrar; token cambiado → registrar en todos; host quitado → borrar; sin token → nada.

### Step 4: `usePushRegistration.ts`

- Pedir permiso con `Notifications.requestPermissionsAsync()` una sola vez (si el usuario lo negó, no volver a pedir; recordar en `SecureStore` con la clave `herdr.push.v1`, junto con el mapa `{ hostId: token }` ya registrado).
- Token nativo: `Notifications.getDevicePushTokenAsync()` (APNs en iOS, FCM en Android). `environment` = `"sandbox"` en builds de desarrollo (`__DEV__` o `APP_VARIANT=development`), `"production"` en las demás.
- Con `planPushRegistrations` sobre `useHerdrHosts()`: `POST /v1/devices` en cada host a registrar (en cada arranque de la app se vuelve a registrar, así el gateway renueva `registeredAtMs`); `DELETE /v1/devices/<encodeURIComponent(token)>` en los hosts quitados (un `404` cuenta como hecho). Un host sin conexión se reintenta la próxima vez que se conecta; si un host quitado nunca vuelve, su registro vence solo a los 90 días en el gateway (historia 17).
- **Tocar la notificación:** `Notifications.addNotificationResponseReceivedListener` y `Notifications.getLastNotificationResponseAsync()` (arranque en frío) → buscar el host con `id === data.hostCoreId` y `navigation.navigate("HerdrAgent", { hostId, address: data.address })`; si el host ya no está guardado, abrir el inicio.
- **App en primer plano:** `Notifications.setNotificationHandler` con `shouldShowBanner: false`, `shouldShowList: false`, `shouldPlaySound: false`, `shouldSetBadge: false`: ya avisa `useAgentAlerts` (historia 8), así no hay doble aviso.

### Step 5: montar y documentar

- `HerdrHomeRoute.tsx`: llamar `usePushRegistration()` (una línea; el inicio está montado mientras la app vive).
- `docs/herdr/mobile-build.md`: sección "Push en la app": variables `HERDR_PUSH_ENABLED`, `HERDR_APNS_ENVIRONMENT`, `HERDR_ANDROID_GOOGLE_SERVICES_FILE`, `HERDR_APPLE_TEAM_ID`; que al activarlas hay que regenerar los proyectos nativos (`vp run mobile:install <ios|android>` compila uno nuevo porque cambia el fingerprint); y que iOS necesita la capacidad Push Notifications en el App ID de la cuenta de pago.

## Out of Scope

- El envío por APNs/FCM, el almacén de dispositivos y el notificador: historia 17.
- Live Activities, widgets y la pantalla de uso de T3: fuera del alcance.
- Modificar `remoteRegistration.ts` u otros archivos de T3 que no estén en la lista permitida.
- Publicar en App Store o Google Play.

## Acceptance Criteria

- [ ] Dentro de `apps/mobile`: `vp test run src/features/herdr/state/pushRegistrationPlan.test.ts app.config.test.ts` pasa.
- [ ] Dentro de `apps/mobile`: `vp run typecheck` pasa.
- [ ] `node scripts/herdr/check-patch-surface.ts` sale con código 0 (solo cambian archivos de `src/features/herdr/`, `app.config.ts`, `app.config.test.ts` y `docs/herdr/`).
- [ ] Con las cuentas configuradas y `HERDR_PUSH_ENABLED=1`: con la app cerrada, un agente que pasa a `blocked` en el escritorio produce una notificación en iOS y en Android con el nombre del espacio como título; tocarla abre ese agente; con la app abierta no aparece aviso del sistema (solo el de `useAgentAlerts`); quitar un servidor borra el token en ese gateway. Anotar el resultado en el tracker.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
