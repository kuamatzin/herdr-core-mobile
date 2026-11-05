---
status: not_started
feature: "herdr-core-mobile"
story: "02"
layers:
  - "frontend"
dependencies:
  - "none"
---

# Story 02: Identidad propia de la app

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code cuya app móvil (`apps/mobile`, Expo / React Native) se va a conectar a herdr-core en lugar de a un servidor T3. Para poder instalarla junto a la app oficial de T3 y compilarla sin las cuentas de T3, la app necesita nombre, bundle ID y esquemas propios, y no debe apuntar a las actualizaciones OTA, dominios asociados, dueño de Expo ni equipo de Apple de T3. El cambio es de valores en `app.config.ts`, con el diff mínimo, para no chocar con las actualizaciones semanales de T3.

## Dependencies

- Depends on: none

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Notificaciones push sin cuentas de Apple ni Firebase?** → **Decision:** historias 17 (gateway) y 18 (app), bloqueadas (`🚧`) hasta que el usuario tenga Apple Developer de pago y un proyecto de Firebase. Mientras tanto, la v1 avisa solo con la app abierta.

`apps/mobile/app.config.ts` y `apps/mobile/app.config.test.ts` están en la lista permitida de `docs/herdr/patch-surface.txt` (historia 1, que corre en paralelo).

## Affected Files

### Frontend

- [ ] Pattern to follow: la estructura existente de `VARIANT_CONFIG` (línea ~80) y `repoEnv` (`loadRepoEnv()`, línea 8) en `apps/mobile/app.config.ts`. Mantener el diff mínimo: cambiar valores, no reestructurar ni mover bloques.
- [ ] Conventions to follow: variables de entorno leídas de `repoEnv` como las vecinas (`repoEnv.T3CODE_IOS_PERSONAL_TEAM`, etc.); pruebas en `app.config.test.ts` con `vite-plus/test`, `vi.mock("../../scripts/lib/public-config.ts", ...)` y `loadConfig(variant)` como ya están.
- [ ] `apps/mobile/app.config.ts` — valores de identidad y desactivación de lo que apunta a T3.
- [ ] `apps/mobile/app.config.test.ts` — expectativas nuevas.
- [ ] `docs/herdr/mobile-build.md` — nuevo: cómo compilar.

## Implementation Steps

### Step 1: `VARIANT_CONFIG` (`apps/mobile/app.config.ts`, línea ~80)

Cambiar solo estos valores (dejar `relyingParty` y `assets` como están):

| Variante      | `appName`       | `scheme`        | `iosBundleIdentifier` / `androidPackage` |
| ------------- | --------------- | --------------- | ---------------------------------------- |
| `development` | `Herdr Dev`     | `herdr-dev`     | `com.kuamatzin.herdrmobile.dev`          |
| `preview`     | `Herdr Preview` | `herdr-preview` | `com.kuamatzin.herdrmobile.preview`      |
| `production`  | `Herdr`         | `herdr`         | `com.kuamatzin.herdrmobile`              |

### Step 2: desactivar lo que apunta a T3

En el objeto `config`:

- `updates.enabled`: `repoEnv.HERDR_MOBILE_UPDATES_ENABLED === "1"` (hoy es `repoEnv.T3CODE_MOBILE_UPDATES_ENABLED !== "0"`, línea ~244). `updates.url` (`https://u.expo.dev/d763fcb8-…`): solo si `HERDR_MOBILE_UPDATES_URL` está definida; si no, omitir la clave `url` (spread condicional).
- `ios.appleTeamId` (hoy `"ARK85ZXQ4Z"`, línea ~255): `repoEnv.HERDR_APPLE_TEAM_ID?.trim() || undefined` (vacío por defecto). Conservar el comentario existente solo si sigue siendo cierto; si no, reemplazarlo por uno de una línea.
- `ios.associatedDomains` (applinks/webcredentials de `clerk.t3.codes` y `app.t3.codes`): omitir por defecto; incluir el arreglo actual solo si `repoEnv.HERDR_T3_LINKS === "1"`.
- `android.intentFilters` con `host: HOSTED_APP_HOST` (`app.t3.codes`, `pathPrefix: "/threads/"`): omitir por defecto, igual condicionado a `HERDR_T3_LINKS === "1"`. No borrar la constante `HOSTED_APP_HOST`.
- `owner: "pingdotgg"`: `repoEnv.HERDR_EXPO_OWNER?.trim() || undefined`.
- `extra.eas.projectId` (`d763fcb8-…`): `repoEnv.HERDR_EAS_PROJECT_ID?.trim() || undefined`. Conservar `extra.eas.observe`.
- `ios.entitlements["keychain-access-groups"]` usa `variant.iosBundleIdentifier`; queda bien con el bundle nuevo, no tocar.
- `NSAllowsArbitraryLoads: true` se queda (el gateway es `http://` sobre Tailscale).
- No agregar capacidades de push ni tocar `expo-notifications`: eso es de la historia 18.
- No tocar las variables `T3CODE_IOS_PERSONAL_TEAM` / `T3CODE_IOS_PERSONAL_TEAM_BUNDLE_ID`: siguen sirviendo para firmar con Personal Team.

### Step 3: `apps/mobile/app.config.test.ts`

- La tabla `it.each` de identificadores pasa a `[undefined, "com.kuamatzin.herdrmobile"]`, `["development", "com.kuamatzin.herdrmobile.dev"]`, `["preview", "com.kuamatzin.herdrmobile.preview"]`, `["production", "com.kuamatzin.herdrmobile"]`, verificando `ios.bundleIdentifier` y `android.package`.
- Quitar de ese caso las expectativas que atan la app a `app.t3.codes` (`associatedDomains` contiene `applinks:<host>`, `apple-app-site-association` contiene `<teamId>.<identifier>`, `intentFilters[0].data`, `assetlinks` contiene el package): ya no aplican por defecto. Quitar los imports que queden sin uso (`DEFAULT_HOSTED_APP_URL`, `appleAppSiteAssociation`, `assetLinks`).
- Agregar un caso: sin variables `HERDR_*`, `config.updates?.enabled === false`, `config.ios?.appleTeamId === undefined`, `config.ios?.associatedDomains === undefined`, `config.android?.intentFilters === undefined`, `config.owner === undefined`, `config.extra?.eas?.projectId === undefined`, y `config.scheme === "herdr"`.
- El mock de `loadRepoEnv` hoy devuelve solo `APP_VARIANT`; ampliarlo para que devuelva **solo** las claves que la prueba controla (`APP_VARIANT` y `HERDR_MOBILE_UPDATES_ENABLED`, leídas de `process.env`), así ninguna otra variable del entorno de quien corre la prueba entra a la configuración y agregar un caso con `vi.stubEnv("HERDR_MOBILE_UPDATES_ENABLED", "1")` → `updates.enabled === true`.
- Mantener el caso existente que rechaza `APP_VARIANT` inválido.

### Step 4: `docs/herdr/mobile-build.md`

Guía corta en español:

- **Android en `mylinux`:** SDK en `~/Android/Sdk` (`ANDROID_HOME`), `vp i` en la raíz, y `APP_VARIANT=development vp run mobile:install android [device-id]` desde la raíz (comando documentado en el `AGENTS.md` de T3: instala el cliente de desarrollo del fingerprint actual y compila solo si hace falta; su salida es una línea JSON y el log del build va al archivo que nombra). Anotar la versión de JDK que pidió el build y dónde quedó el APK; cómo instalarlo en el teléfono por `adb`.
- **iOS en la Mac con Personal Team:** `T3CODE_IOS_PERSONAL_TEAM=1` y `T3CODE_IOS_PERSONAL_TEAM_BUNDLE_ID=com.kuamatzin.herdrmobile.dev` (o el que corresponda), la app caduca a los ~7 días sin cuenta de pago; `HERDR_APPLE_TEAM_ID` solo cuando haya cuenta de pago.
- **Variables:** tabla con `HERDR_MOBILE_UPDATES_ENABLED`, `HERDR_MOBILE_UPDATES_URL`, `HERDR_APPLE_TEAM_ID`, `HERDR_T3_LINKS`, `HERDR_EXPO_OWNER`, `HERDR_EAS_PROJECT_ID` y su valor por defecto.
- No usar el `.env.example` de T3: decir qué variables mínimas hacen falta (ninguna para un build de desarrollo Android).

## Out of Scope

- Capacidades y entitlements de push (APNs/FCM): historia 18.
- Cambiar la pantalla de inicio o rutas (`Stack.tsx`): historia 7.
- Íconos, splash, assets de marca y nombres de plugins de Expo: se quedan los de T3 (ningún asset nuevo). Los proyectos nativos generados (`ios/`, `android/`) se regeneran con el build; no se versionan cambios en ellos.
- Live Activities, widgets y la pantalla de uso de T3: fuera del alcance del plan; no se borran ni se configuran.
- Publicar en App Store o Google Play.
- Reestructurar `app.config.ts` o borrar constantes de T3.

## Acceptance Criteria

- [ ] Dentro de `apps/mobile`: `vp test run app.config.test.ts` pasa, incluyendo los casos nuevos de identificadores, valores por defecto sin `HERDR_*` y `HERDR_MOBILE_UPDATES_ENABLED=1`.
- [ ] Dentro de `apps/mobile`: `vp run typecheck` pasa.
- [ ] El diff de `app.config.ts` solo cambia valores y agrega condicionales; no mueve ni borra bloques ni constantes (`HOSTED_APP_HOST` sigue existiendo).
- [ ] Si `scripts/herdr/check-patch-surface.ts` ya existe (historia 1 corre en paralelo), `node scripts/herdr/check-patch-surface.ts` sale con código 0.
- [ ] Dentro de `apps/mobile`, `APP_VARIANT=development vp exec expo config --type public` muestra `name: "Herdr Dev"`, `scheme: "herdr-dev"` y `android.package: "com.kuamatzin.herdrmobile.dev"`. La compilación Android en `mylinux` se valida a mano siguiendo `docs/herdr/mobile-build.md` (anotar el resultado en las notas de la historia del tracker).
- [ ] `docs/herdr/mobile-build.md` existe con las secciones Android en `mylinux`, iOS con Personal Team y la tabla de variables.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

S
