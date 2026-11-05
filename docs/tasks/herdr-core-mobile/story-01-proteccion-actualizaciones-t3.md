---
status: not_started
feature: "herdr-core-mobile"
story: "01"
layers:
  - "other"
  - "cli"
dependencies:
  - "none"
---

# Story 01: Protección de actualizaciones de T3

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code (`pingdotgg/t3code`) que agrega una app móvil para herdr-core. T3 recibe cientos de commits al mes, y el fork debe poder fusionarlos cada semana sin conflictos. Esta historia crea las reglas del fork y la maquinaria que las hace cumplir: una lista de archivos de T3 que se permite modificar, un script que falla si se modifica otro, un workflow de CI que corre ese script, y un workflow semanal que abre un PR con los cambios de T3. Todas las historias siguientes dependen de esta red de seguridad.

## Dependencies

- Depends on: none

## Resolved Decisions

1. **Q: ¿Qué base?** → **Decision:** T3 Code directo (`pingdotgg/t3code`), no `benkraus/herdr-mobile`. Motivo: en `benkraus` solo ~8,000 de 60,500 líneas se ejecutan, y el chat de T3 es código muerto de hace 7,209 commits. Con T3 hay historial común y se puede hacer `git merge`.
2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **Q: ¿Actualizaciones de T3?** → **Decision:** semanales y revisadas. Un workflow abre cada semana un PR de `pingdotgg/t3code@main` hacia el fork, corre el chequeo de archivos permitidos y las pruebas de `packages/herdr`, `apps/herdr-gateway` y `apps/mobile/src/features/herdr`. Si hay conflictos, el PR se abre como borrador listando los archivos en conflicto. Nunca se fusiona solo.
4. **(plan) Planes dentro del fork.** El `AGENTS.md` de T3 pide no subir planes. En este fork, `docs/tasks/` es una excepción documentada en `docs/herdr/AGENTS.md`, porque son carpetas nuevas y no chocan con T3.

## Affected Files

### Other (CI, scripts y documentación del fork)

- [ ] Pattern to follow: scripts existentes en `scripts/` (Node con TypeScript, se ejecutan con `node scripts/<x>.ts`, pruebas `*.test.ts` al lado, por ejemplo `scripts/check-changed.ts` y `scripts/check-changed.test.ts`); workflows existentes en `.github/workflows/ci.yml` (usa la acción compuesta `./.github/actions/setup-ci-deps` para instalar Vite+ y dependencias).
- [ ] Conventions to follow: TypeScript plano con APIs de Node (`node:child_process`, `node:fs`), sin Effect; funciones puras exportadas para poder probarlas; sin `enum` ni `namespace`.
- [ ] `docs/herdr/AGENTS.md` — nuevo: reglas del fork.
- [ ] `docs/herdr/patch-surface.txt` — nuevo: lista permitida de archivos de T3.
- [ ] `scripts/herdr/check-patch-surface.ts` — nuevo: chequeo de la lista permitida.
- [ ] `scripts/herdr/check-patch-surface.test.ts` — nuevo: pruebas del chequeo.
- [ ] `.github/workflows/herdr-guard.yml` — nuevo: corre el chequeo en CI.
- [ ] `.github/workflows/herdr-upstream-sync.yml` — nuevo: PR semanal desde T3.
- [ ] `CLAUDE.md` — agregar una línea al final.

## Implementation Steps

### Step 1: `docs/herdr/AGENTS.md`

Escribir en español, corto y operativo (es leído por agentes de código):

- **Directorios propios del fork:** `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/`, `.github/workflows/herdr-*.yml` y `docs/tasks/`. Todo código nuevo va ahí.
- **Archivos de T3:** solo se modifican los listados en `docs/herdr/patch-surface.txt`. Para agregar uno, se agrega primero a esa lista con un comentario `#` que diga por qué, en el mismo commit. Nunca se borra código de T3: lo que no se usa se oculta (ejemplo: la ruta `Home` de `apps/mobile/src/Stack.tsx` apunta a la pantalla de Herdr, pero `WorkspaceRootScreen` sigue existiendo y compilando).
- **Excepción de planes (decisión 18):** el `AGENTS.md` de T3 pide no subir planes; en este fork `docs/tasks/` sí se versiona, porque es una carpeta nueva que no choca con T3.
- **Cómo verificar lo propio:** `node scripts/herdr/check-patch-surface.ts`; `vp test run <archivos>` y `vp run typecheck` dentro de `packages/herdr`, `apps/herdr-gateway` y `apps/mobile` (pruebas de `src/features/herdr`). Nunca chequeos de todo el repo (`vp check`, `vp run -r test`, `vp run -r typecheck`).
- **Actualizaciones de T3:** el workflow `herdr-upstream-sync` abre un PR cada lunes; nunca se fusiona solo; si trae conflictos, el PR es borrador y lista los archivos en conflicto. Un cambio de props en componentes de T3 que reutilizamos (`ThreadFeed`, `TerminalSurface`, layout) se arregla en ese PR.
- **Workflows de T3 en el fork:** se deshabilitan con `gh workflow disable <nombre>` (acción manual del usuario, no edición de archivos). Listar los nombres actuales de `.github/workflows/` que no empiezan con `herdr-` (hoy: `agent-context-budget.yml`, `ci.yml`, `deploy-relay.yml`, `desktop-macos-preview-publish.yml`, `desktop-macos-preview.yml`, `issue-labels.yml`, `mobile-android-apk.yml`, `mobile-eas-preview-fork.yml`, `mobile-eas-preview.yml`, `mobile-eas-production.yml`, `mobile-fingerprint-check.yml`, `mobile-fingerprint-label.yml`, `mobile-showcase-screenshots.yml`, `pr-size.yml`, `pr-vouch.yml`, `publish-aur.yml`, `relay-preview-cleanup.yml`, `relay-preview.yml`, `release-desktop.yml`, `release.yml`, `stale-needs-more-info.yml`, `thread-transfer-report.yml`, `web-preview.yml`, `windows-tests.yml`) con un bucle de ejemplo: `for w in $(ls .github/workflows | grep -v '^herdr-'); do gh workflow disable "$w"; done`.
- **Remoto `upstream`:** `git remote add upstream https://github.com/pingdotgg/t3code.git && git fetch upstream main` (el chequeo lo necesita).
- **Token del sync:** explicar que un PR abierto con el `GITHUB_TOKEN` por defecto no dispara otros workflows (`herdr-guard` no correría en ese PR); por eso el workflow de sync corre el chequeo y las pruebas él mismo, y si existe el secreto `HERDR_SYNC_TOKEN` (un PAT con `contents:write` y `pull-requests:write`) lo usa para que `herdr-guard` también corra en el PR.

### Step 2: `docs/herdr/patch-surface.txt`

Una ruta por línea, relativa a la raíz, con un comentario `#` al final o en la línea anterior que diga por qué. Líneas vacías y comentarios se ignoran. Contenido inicial exacto (rutas):

```
# Carga docs/herdr/AGENTS.md para los agentes de código
CLAUDE.md
# Ruta Home apunta a la pantalla de Herdr y registra las rutas herdr (historia 7)
apps/mobile/src/Stack.tsx
# Identidad propia de la app, sin OTA ni dominios de T3 (historia 2) y push (historia 18)
apps/mobile/app.config.ts
apps/mobile/app.config.test.ts
# Dependencia @herdr/core-client (historia 7)
apps/mobile/package.json
# Cambia con cada dependencia nueva de los paquetes del fork
pnpm-lock.yaml
```

Sin comodines: coincidencia exacta de ruta.

### Step 3: `scripts/herdr/check-patch-surface.ts`

Se ejecuta con `node scripts/herdr/check-patch-surface.ts` desde la raíz (solo módulos de Node; nada de dependencias del workspace, para que corra en CI sin `vp i`).

1. Exportar funciones puras:
   - `parsePatchSurface(text: string): Set<string>` — quita comentarios (`#` hasta fin de línea), espacios y líneas vacías.
   - `findPatchSurfaceViolations(input: { changed: ReadonlyArray<string>; existsUpstream: (path: string) => boolean; allowed: ReadonlySet<string> }): string[]` — devuelve, ordenadas, las rutas cambiadas que existen en upstream y no están permitidas.
2. `main()` (solo cuando el archivo es el punto de entrada, `import.meta.url === pathToFileURL(process.argv[1]).href`):
   - Si `git rev-parse --verify --quiet upstream/main` falla: imprimir `Falta upstream/main. Corre: git remote add upstream https://github.com/pingdotgg/t3code.git && git fetch upstream main` y salir con código 2.
   - `base = git merge-base HEAD upstream/main`.
   - Archivos cambiados: `git diff --name-only --no-renames -z <base>` (compara contra el árbol de trabajo, así también detecta cambios sin commit; `--no-renames` hace que un renombre cuente como borrado más alta). Usar `execFileSync("git", [...])`, nunca un shell.
   - `existsUpstream(path)`: `git cat-file -e upstream/main:<path>` con `execFileSync` en `try/catch` (código distinto de 0 → no existe). Los archivos nuevos que no existen en upstream no cuentan.
   - Leer `docs/herdr/patch-surface.txt`. Si hay violaciones: imprimir `Archivos de T3 modificados fuera de docs/herdr/patch-surface.txt:` seguido de una ruta por línea, y salir con código 1. Si no: imprimir `patch-surface OK (<n> archivos de T3 permitidos modificados)` y salir con 0.
3. `scripts/herdr/check-patch-surface.test.ts` (con `vite-plus/test`, como las pruebas vecinas en `scripts/`): casos de `parsePatchSurface` (comentarios, líneas vacías, espacios) y de `findPatchSurfaceViolations`: archivo de T3 permitido → sin violación; archivo de T3 no permitido → violación; archivo nuevo (no existe upstream) → sin violación; borrado de un archivo de T3 no permitido → violación.

### Step 4: `.github/workflows/herdr-guard.yml`

```yaml
name: herdr-guard
on:
  push:
  pull_request:
permissions:
  contents: read
jobs:
  patch-surface:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - uses: ./.github/actions/setup-ci-deps
        with:
          install: "false"
      - run: node --version
      - run: git remote add upstream https://github.com/pingdotgg/t3code.git
      - run: git fetch --no-tags upstream main
      - run: node scripts/herdr/check-patch-surface.ts
```

El Node lo fija Vite+ (`engines.node` de `package.json`, hoy `26.12.1`) a través de la acción `setup-ci-deps` de T3 (se reutiliza sin modificarla; `install: "false"` instala solo Vite+ y su Node). El paso `node --version` deja constancia en el log; si el `node` del `PATH` no es el de Vite+, ajustar el paso para usar el de `~/.vite-plus` y anotarlo. El script solo usa sintaxis de TypeScript que Node ejecuta sin banderas (sin `enum`, `namespace` ni parámetros de propiedad). No usar los runners `blacksmith-*` de T3 (son de pago de T3).

### Step 5: `.github/workflows/herdr-upstream-sync.yml`

- Disparadores: `schedule: - cron: "0 14 * * 1"` (lunes 14:00 UTC) y `workflow_dispatch`.
- `permissions: contents: write, pull-requests: write`.
- Job `sync` (bash; `set -uo pipefail`, **sin** `-e` en los pasos que capturan resultados):
  1. `actions/checkout@v6` con `fetch-depth: 0` y `token: ${{ secrets.HERDR_SYNC_TOKEN || github.token }}`.
  2. `git config user.name "herdr-upstream-sync"` y `user.email "herdr-upstream-sync@users.noreply.github.com"`.
  3. `git remote add upstream https://github.com/pingdotgg/t3code.git && git fetch --no-tags upstream main`.
  4. Si `git merge-base --is-ancestor upstream/main HEAD`: imprimir "Sin cambios de T3" y terminar con éxito (no abrir PR).
  5. `BRANCH=upstream-sync/$(date -u +%F)`; si la rama ya existe en el remoto (`git ls-remote --exit-code --heads origin "$BRANCH"`), terminar con éxito sin abrir otro PR; si no, `git switch -c "$BRANCH"`.
  6. El merge va dentro de un `if` explícito para que un conflicto no aborte el script:
     ```bash
     if git merge --no-edit upstream/main; then CONFLICTS=""; else CONFLICTS=$(git diff --name-only --diff-filter=U); git add -A; git commit --no-edit -m "chore(upstream): merge pingdotgg/t3code@main con conflictos"; fi
     ```
     Con conflictos: `git push -u origin "$BRANCH"` y `gh pr create --draft --base main --head "$BRANCH" --title "chore(upstream): T3 $(date -u +%F) (conflictos)" --body "<lista de CONFLICTS, una por línea con viñeta>"`. Terminar.
  7. Sin conflictos, **siempre** (sin condicionar a qué directorios existen):
     - `./.github/actions/setup-ci-deps` (instala Vite+, su Node fijado y las dependencias; equivale a `vp i`).
     - `node scripts/herdr/check-patch-surface.ts`.
     - `vp run typecheck` dentro de `packages/herdr`, `apps/herdr-gateway` y `apps/mobile`.
     - `vp test run` dentro de `packages/herdr` y `apps/herdr-gateway`, y `vp test run src/features/herdr` dentro de `apps/mobile`.
       Cada paso guarda su resultado (pasa/falla) sin abortar el job. Un directorio herdr que todavía no existe (antes de las historias 3, 4 o 7) cuenta como "no aplica", no como pasa.
  8. `git push -u origin "$BRANCH"` y `gh pr create --base main --head "$BRANCH" --title "chore(upstream): T3 $(date -u +%F)" --body "<tabla con el resultado de cada paso>"`, agregando `--draft` si **cualquier** paso falló; en ese caso el job termina en fallo para que se vea en Actions.
  - `GH_TOKEN: ${{ secrets.HERDR_SYNC_TOKEN || github.token }}` en el entorno de los pasos con `gh`.
  - Nunca `gh pr merge`.
- Job aparte `android-build`, solo `workflow_dispatch` (manual) y **no requerido**: `setup-ci-deps` y la compilación de Android de desarrollo de `apps/mobile` (el comando de `docs/herdr/mobile-build.md` de la historia 2; si ese archivo aún no existe, dejar el job con un `echo` que lo diga).

### Step 6: `CLAUDE.md`

Agregar al final, como última línea, exactamente: `@docs/herdr/AGENTS.md`. No tocar la línea `@AGENTS.md` existente.

## Out of Scope

- Deshabilitar los workflows de T3: es una acción manual del usuario con `gh workflow disable`; solo se documenta. No se editan ni borran workflows de T3.
- Crear el fork `kuamatzin/herdr-core-mobile` o el remoto `origin`: lo hace el usuario (ver Blockers del tracker).
- Cualquier código de app, gateway o paquete (historias 2 a 17).
- Agregar a `patch-surface.txt` el archivo del layout de iPad: lo hace la historia 8 cuando sepa cuál es.

## Acceptance Criteria

- [ ] `vp test run scripts/herdr/check-patch-surface.test.ts` (desde la raíz, o `vp test run herdr/check-patch-surface.test.ts` dentro de `scripts/`) pasa con los cuatro casos de `findPatchSurfaceViolations` del Step 3.
- [ ] Con el remoto `upstream` presente y el árbol actual (solo archivos nuevos más `CLAUDE.md`), `node scripts/herdr/check-patch-surface.ts` sale con código 0.
- [ ] Modificar temporalmente un archivo de T3 que no está en la lista (por ejemplo, agregar un espacio a `apps/mobile/src/App.tsx` o cualquier `.ts` de T3) hace que el script salga con código 1 y lo imprima; revertir el cambio.
- [ ] Sin `upstream/main`, el script sale con código 2 y el mensaje con el comando para agregarlo.
- [ ] En `herdr-upstream-sync.yml`, el `git merge` está dentro de un `if`, los pasos del Step 5.7 corren siempre que no hay conflictos, y un fallo de cualquiera marca el PR como borrador.
- [ ] `herdr-guard.yml` y `herdr-upstream-sync.yml` son YAML válido (`python3 -c 'import sys,yaml; [yaml.safe_load(open(f)) for f in sys.argv[1:]]' .github/workflows/herdr-*.yml` sin error, o `actionlint` si está instalado) y el sync nunca llama a `gh pr merge`.
- [ ] `CLAUDE.md` termina con la línea `@docs/herdr/AGENTS.md`.
- [ ] `docs/herdr/AGENTS.md` cubre: directorios propios, lista permitida y cómo agregar a ella, no borrar código de T3, excepción de `docs/tasks/`, cómo correr las pruebas propias, `gh workflow disable` y el secreto `HERDR_SYNC_TOKEN`.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
