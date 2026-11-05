# HERDR-MOBILE-1 App móvil de T3 Code sobre herdr-core — Progress Tracker

## Story Status

| Story    | Title                                                 | Wave | Status         | Completed At |
| -------- | ----------------------------------------------------- | ---- | -------------- | ------------ |
| story-01 | Protección de actualizaciones de T3                   | 1    | ❌ Not Started | —            |
| story-02 | Identidad propia de la app                            | 1    | ❌ Not Started | —            |
| story-03 | Cliente de `hc`                                       | 1    | ❌ Not Started | —            |
| story-04 | Gateway base                                          | 2    | ❌ Not Started | —            |
| story-05 | Gateway, terminal                                     | 3    | ❌ Not Started | —            |
| story-06 | Gateway, archivos                                     | 3    | ❌ Not Started | —            |
| story-07 | App, conexión e inicio                                | 3    | ❌ Not Started | —            |
| story-08 | App, lista de agentes e iPad                          | 4    | ❌ Not Started | —            |
| story-09 | App, terminal                                         | 4    | ❌ Not Started | —            |
| story-10 | App, archivos                                         | 4    | ❌ Not Started | —            |
| story-11 | Proxy `herdr-tap`                                     | 3    | ❌ Not Started | —            |
| story-12 | Hooks de Claude Code                                  | 3    | ❌ Not Started | —            |
| story-13 | Gateway, sesión de chat                               | 4    | ❌ Not Started | —            |
| story-14 | App, pantalla de chat                                 | 5    | ❌ Not Started | —            |
| story-15 | App, escribir y aprobar                               | 6    | ❌ Not Started | —            |
| story-16 | Chat de Codex                                         | 5    | ❌ Not Started | —            |
| story-17 | Push en el gateway (🚧 bloqueada hasta tener cuentas) | 3    | 🚧 Blocked     | —            |
| story-18 | Push en la app (🚧 bloqueada hasta tener cuentas)     | 5    | 🚧 Blocked     | —            |

_(Status values: ❌ Not Started · 🚧 Blocked · 🔄 In Progress · ✅ Completed)_

## Technical Decisions

| Decision                                                                                                                                                                                                                                                                                                                                             | Rationale                                                                                         | Story                          |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | ------------------------------ |
| 1. Base: T3 Code directo (`pingdotgg/t3code`), no `benkraus/herdr-mobile`                                                                                                                                                                                                                                                                            | Historial común permite `git merge`; en `benkraus` casi todo era código muerto                    | 1                              |
| 2. Superficie herdr aislada: código nuevo solo en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/`, `.github/workflows/herdr-*.yml`; archivos de T3 solo si están en `docs/herdr/patch-surface.txt`; nada de T3 se borra                                                                 | Recibir las actualizaciones semanales de T3 sin conflictos                                        | 1–18                           |
| 3. Herdr es la pantalla principal (ruta `Home`); las pestañas de T3 no aparecen                                                                                                                                                                                                                                                                      | El usuario abre la app para ver sus agentes y terminales                                          | 7, 8                           |
| 4. Alcance v1: espacios, pestañas, terminal en vivo, agentes por prioridad, archivos, avisos con la app abierta; chat después; push al final                                                                                                                                                                                                         | Base usable primero                                                                               | 8, 9, 10                       |
| 5. Chat de Claude Code: proxy `herdr-tap` (palabra por palabra), hooks (prompts, herramientas, permisos, fin de turno), transcripción solo para historial, envío con `terminal.send`                                                                                                                                                                 | Fuentes oficiales con el Claude Code real intacto en su terminal                                  | 11, 12, 13, 14, 15             |
| 6. (revisada) `~/.zshenv` solo define una función zsh `claude` que pone `ANTHROPIC_BASE_URL` para ese proceso si no está definida y el tap está vivo (PID + `kill -0` + `/proc/<pid>/cmdline` con `herdr-tap`); nada más del entorno cambia                                                                                                          | Exportar la variable afectaba a cualquier SDK o script; un PID reciclado no prueba que sea el tap | 11                             |
| 7. Codex: comprobación primero; base garantizada leyendo `rollout-*.jsonl` (por bloques); palabra por palabra solo si un `app-server` compartido no rompe `--no-daemon`                                                                                                                                                                              | El protocolo en vivo de Codex puede ser inviable dentro de `hc`                                   | 16                             |
| 8. El chat va justo después de la base (olas 3 a 6); 11 y 12 comprueban con Claude real                                                                                                                                                                                                                                                              | Validar los riesgos antes de construir encima                                                     | 11, 12, 13, 14                 |
| 9. Un gateway por servidor; la app se conecta a todos; uno caído no afecta a los demás                                                                                                                                                                                                                                                               | Servidores independientes (`mylinux`, `gx10`)                                                     | 4, 7, 8, 17, 18                |
| 10. La terminal se ajusta siempre al teléfono (tamaño al abrir y al rotar); el último visor activo manda                                                                                                                                                                                                                                             | Legibilidad en el teléfono; sin modo espejo                                                       | 5, 9                           |
| 11. Solo Tailscale: escucha en `tailscale ip -4`, nunca `0.0.0.0`; `tailscale whois` por conexión y solo el `LoginName` dueño del nodo; si no, `403`                                                                                                                                                                                                 | Acceso equivalente a shell, sin cuentas propias                                                   | 4, 5, 6, 17                    |
| 12. Actualizaciones de T3 semanales y revisadas: PR automático desde `pingdotgg/t3code@main`, chequeo de lista permitida y pruebas propias; borrador si hay conflictos; nunca se fusiona solo                                                                                                                                                        | Absorber el churn de T3 con revisión humana                                                       | 1                              |
| 13. Push en las historias 17 (gateway) y 18 (app), bloqueadas hasta tener Apple Developer de pago y Firebase; mientras, avisos solo con la app abierta                                                                                                                                                                                               | Sin cuentas no hay APNs/FCM                                                                       | 2, 8, 17, 18                   |
| 14. Gateway y `packages/herdr` en TypeScript plano (Promises + Node), sin Effect; `packages/herdr` no importa `@t3tools/*`                                                                                                                                                                                                                           | No acoplarse a la evolución de las librerías del servidor T3                                      | 3, 4, 5, 6, 11, 12, 13, 16, 17 |
| 15. `herdr-tap` es un proceso/servicio aparte; nunca guarda credenciales ni cuerpos; emite eventos a un socket local sin esperar respuesta                                                                                                                                                                                                           | Si el gateway cae, los agentes no se cortan                                                       | 11, 13                         |
| 16. Archivos con `node:fs` (no por `hc`), limitados a `$HOME` después de `realpath`                                                                                                                                                                                                                                                                  | El gateway corre en el servidor; evitar salidas por `..` o enlaces                                | 6, 10                          |
| 17. (revisada) El hook solo espera al teléfono si hay una app con el chat de ese agente abierto; máximo 120 s; se libera con `PostToolUse`, `PostToolUseFailure`, `PermissionDenied`, `Stop`, `UserPromptSubmit` o `SessionEnd` de la misma sesión (no con `Notification`); respaldo aprobado: 30 s si responder en la terminal no termina la espera | No bloquear al escritorio aunque no haya señal de cancelación                                     | 12, 13, 15                     |
| 18. `docs/tasks/` se versiona en el fork (excepción documentada en `docs/herdr/AGENTS.md`)                                                                                                                                                                                                                                                           | Carpeta nueva, no choca con T3                                                                    | 1                              |
| 19. `permission_suggestions` solo como información; el teléfono responde solo `allow`/`deny`                                                                                                                                                                                                                                                         | Límite del contrato de `/approve`                                                                 | 12, 13, 15                     |
| 20. Aprobaciones al teléfono solo con el chat de ese agente abierto; las levantadas antes o en modo Terminal se quedan en la terminal (buscado y probado)                                                                                                                                                                                            | Evitar tomar decisiones que el escritorio ya puede estar respondiendo                             | 12, 13, 15                     |
| (review) Correlación sesión ↔ terminal: hook manda `$HC_TERMINAL_ID`; mapa `{sessionId → terminal, transcriptPath, claudeVersion}` persistido en `$XDG_RUNTIME_DIR/herdr-gateway/sessions.json`; eventos del tap sin mapear se guardan 30 s / 500 y se reproducen; transcripción solo por `transcript_path` del hook                                 | Correlación estable ante carreras y reinicios del gateway                                         | 12, 13                         |
| (review) Modelo de confianza: procesos del mismo usuario son de confianza; TOCTOU fuera de alcance pero `O_NOFOLLOW` + revalidación del descriptor; socket del tap 0700/0600; ruta de depuración solo con `HERDR_GATEWAY_DEBUG=1`, nunca en el unit                                                                                                  | Acceso equivalente a shell; mitigación barata                                                     | 6, 11, 12, 13                  |
| (review) Compatibilidad fail-closed: `hello.generation === 1` o `503 core_incompatible`; documento `herdr-app` validado o `503 layout_unsupported`; `install.sh` avisa si `hc schema` difiere del fixture                                                                                                                                            | Un `hc` nuevo no debe corromper el modelo en silencio                                             | 4                              |
| (review) Sync semanal: tras el merge siempre `vp i`, patch-surface, typecheck de los tres paquetes y pruebas herdr; borrador si algo falla; merge dentro de un `if`; Node fijado por Vite+; build de Android manual y no requerido                                                                                                                   | Que el PR semanal pruebe la integración                                                           | 1                              |
| (review) Pestaña nueva visible en el escritorio (binding): etiquetas `herdr.space`/`herdr.tab` con id nuevo `tb` + 32 hex                                                                                                                                                                                                                            | Documentado en herdr-app: id desconocido = pestaña nueva                                          | 5                              |
| (review) `error` cuenta como `blocked` (atención y aviso); línea base de avisos = primer snapshot tras cada (re)conexión por host                                                                                                                                                                                                                    | El estado `error` existe en herdr-core; evitar avisos falsos al reconectar                        | 8                              |
| (review) `ThreadFeed` solo si una comprobación acotada pasa (fixture con providers ya montados, sin tocar T3); si no, `HerdrChatFeed` (camino planeado); herramientas truncadas a 4 KB por el gateway                                                                                                                                                | Decisión acotada de antemano                                                                      | 13, 14                         |
| (review) Codex: sesión por proceso (`foreground.pid` → `codex` → `/proc/<pid>/fd` con `rollout-*.jsonl`); por cwd+tiempo solo con `sessionUncertain: true`; "viable" = comandos bajo hc y deltas sin cambiar cómo se lanza Codex                                                                                                                     | Varias sesiones pueden compartir cwd                                                              | 16                             |
| (review) Push partido en gateway (17, ola 3) y app (18, ola 5); stubs `routes/devices.ts` y `push/notifier.ts` y `EnvironmentFile=` en la historia 4; secretos en `push.env` 600; un aviso por (terminal, turno, estado); `DELETE /v1/devices/:token`                                                                                                | Tamaño y área; sin compartir archivos con la ola 3                                                | 4, 17, 18                      |
| (shard) Socket del tap: socket Unix de stream con un JSON por línea en lugar de datagramas                                                                                                                                                                                                                                                           | Node no soporta `AF_UNIX`/`SOCK_DGRAM`; misma semántica (sin respuesta, se descarta si falla)     | 11, 13                         |
| (shard) Historia 7 reserva también `HerdrNewTab` (`HerdrNewTabSheet.tsx`) y `HerdrFilesInspector.tsx` como temporales                                                                                                                                                                                                                                | Que las historias 8, 9 y 10 (ola 4) no compartan `herdrRoutes.tsx` ni `HerdrAgentRoute.tsx`       | 7, 8, 9, 10                    |
| (shard) `install.sh` lo edita solo la historia 11, que llama a `install-hooks.ts install/uninstall` de la historia 12 si existe                                                                                                                                                                                                                      | Historias 11 y 12 corren en paralelo (ola 3)                                                      | 11, 12                         |

## Implementation Notes

### story-01

[To be filled by the story agent]

### story-02

[To be filled by the story agent]

### story-03

[To be filled by the story agent]

### story-04

[To be filled by the story agent]

### story-05

[To be filled by the story agent]

### story-06

[To be filled by the story agent]

### story-07

[To be filled by the story agent]

### story-08

[To be filled by the story agent]

### story-09

[To be filled by the story agent]

### story-10

[To be filled by the story agent]

### story-11

[To be filled by the story agent]

### story-12

[To be filled by the story agent]

### story-13

[To be filled by the story agent]

### story-14

[To be filled by the story agent — anotar aquí qué camino se tomó (`ThreadFeed` de T3 o `HerdrChatFeed`) y por qué]

### story-15

[To be filled by the story agent]

### story-16

[To be filled by the story agent]

### story-17

[To be filled by the story agent]

### story-18

[To be filled by the story agent]

## Blockers

| Story        | Blocker                                                                                                                                                   | Resolution                                                                                                                                                                            |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| story-17     | Requiere cuenta de Apple Developer de pago (llave APNs `.p8`, Team ID) y un proyecto de Firebase (cuenta de servicio FCM HTTP v1)                         | El usuario crea las cuentas y `~/.config/herdr-gateway/push.env` (modo 600) con `HERDR_APNS_*`, `HERDR_APPLE_TEAM_ID`, `HERDR_FCM_SERVICE_ACCOUNT`; entonces se pasa a ❌ Not Started |
| story-18     | Mismas cuentas (capacidad Push en el App ID, `google-services.json`) y la historia 17 hecha                                                               | El usuario configura `HERDR_PUSH_ENABLED`, `HERDR_APPLE_TEAM_ID`, `HERDR_ANDROID_GOOGLE_SERVICES_FILE`; entonces se pasa a ❌ Not Started                                             |
| repo (todas) | El fork `kuamatzin/herdr-core-mobile` hoy viene de `benkraus`; el checkout local solo tiene el remoto `upstream` (`pingdotgg/t3code`) y no tiene `origin` | El usuario borra ese fork, hace fork de `pingdotgg/t3code` con el nombre `kuamatzin/herdr-core-mobile` y agrega `origin` antes de subir la historia 1                                 |

## Files Created / Modified

| File | Change | Story |
| ---- | ------ | ----- |
| —    | —      | —     |

## Current State & Next Steps

Ready to implement. Run: `/implement-feature docs/tasks/herdr-core-mobile/`

Antes de empezar: resolver el blocker del repo (fork recreado desde `pingdotgg/t3code` y remoto `origin`). Las historias 17 (ola 3) y 18 (ola 5) están 🚧 bloqueadas por cuentas externas: `/implement-feature` reintenta las historias bloqueadas al arrancar, así que hay que excluirlas de la corrida hasta que existan las cuentas. Nada más depende de ellas.
