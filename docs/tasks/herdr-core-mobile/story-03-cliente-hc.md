---
status: not_started
feature: "herdr-core-mobile"
story: "03"
layers:
  - "backend"
dependencies:
  - "none"
---

# Story 03: Cliente de `hc`

> **Precedence.** Resolved Decisions, Out of Scope, and Acceptance Criteria are **binding**; Implementation Steps, Pattern to follow, and Conventions to follow are the **recommended path** — if a step conflicts with the binding items, the intent wins; deviate only with a noted reason.

## Objective

Este repo es un fork de T3 Code que agrega una app móvil para herdr-core (`hc`), el runtime de terminales detrás de la app de escritorio Herdr. Un gateway por servidor (historia 4) hablará con `hc connect` por stdin/stdout. Esta historia crea `packages/herdr` (`@herdr/core-client`): el cliente TypeScript del protocolo de `hc` — tramas binarias, JSON-RPC sobre el canal 0, eventos y streams de terminal con control de flujo por créditos — más un fixture del esquema oficial y una prueba de contrato que avisa si `hc` cambia.

## Dependencies

- Depends on: none

## Resolved Decisions

2. **Q: ¿Arquitectura?** → **Decision:** superficie herdr aislada. Todo el código nuevo vive en `packages/herdr/`, `apps/herdr-gateway/`, `apps/mobile/src/features/herdr/`, `docs/herdr/`, `scripts/herdr/` y `.github/workflows/herdr-*.yml`. Los archivos de T3 solo se modifican si están en `docs/herdr/patch-surface.txt`. Nada de T3 se borra: lo que no se use se oculta. Ejemplo: para que el inicio sea Herdr se cambia el `screen` de la ruta `Home` en `apps/mobile/src/Stack.tsx`; `WorkspaceRootScreen` sigue existiendo y compilando.
3. **(plan) Gateway sin Effect.** El gateway y `packages/herdr` usan TypeScript plano con Promises y APIs de Node, no Effect, para no acoplarse a la evolución de las librerías del servidor T3. `packages/herdr` no importa nada de `@t3tools/*`.

## Affected Files

### Backend

- [ ] Pattern to follow: **fuentes de verdad del protocolo, en el repo:** `docs/tasks/herdr-core-mobile/hc-protocol.md` (copia de la especificación de herdr-core, `schema/README.md`, commit `c6b316f`, Apache-2.0: modos plano y con tramas, streams de attach, créditos, eventos, documentos) y `docs/tasks/herdr-core-mobile/hc-schema-0.1.18.json` (OpenRPC de `hc schema`, herdr-core 0.1.18, generación 1: métodos, parámetros y formas de respuesta). Leer ambos antes de escribir código; no inferir el formato de la red. Referencia de diseño (Apache-2.0, solo lectura): `zzulanas/herdr-ios`, `Packages/HerdrKit/Sources/HerdrKit/HerdrClient.swift` y `Frame.swift`. **No copiar** código de `juanibiapina/herdr-mobile` (sin licencia). Para la forma del paquete: `packages/tailscale/package.json` y `packages/tailscale/tsconfig.json`.
- [ ] Conventions to follow: TypeScript plano con Promises (decisión 14), sin `enum` ni `namespace` (solo tipos que se borran), sin imports de `@t3tools/*`. El código de `src/` no usa APIs de Node (`Buffer`, `node:*`): solo `Uint8Array`, `DataView`, `TextEncoder`/`TextDecoder`, porque la app móvil importará `@herdr/core-client/gateway-api`. Pruebas `*.test.ts` junto al archivo con `vite-plus/test`.
- [ ] `packages/herdr/package.json` — nuevo.
- [ ] `packages/herdr/tsconfig.json` — nuevo.
- [ ] `packages/herdr/src/hc/frames.ts` — nuevo: codificador/decodificador de tramas.
- [ ] `packages/herdr/src/hc/connection.ts` — nuevo: `HcConnection`.
- [ ] `packages/herdr/src/hc/types.ts` — nuevo: tipos del esquema.
- [ ] `packages/herdr/src/hc/index.ts` — nuevo: reexporta los tres anteriores (entrada de `./hc`).
- [ ] `packages/herdr/src/gateway-api.ts` — nuevo, mínimo (`export {}` más un comentario de una línea); la historia 4 lo llena.
- [ ] `packages/herdr/fixtures/hc-schema.json` — nuevo: copia de `hc schema`.
- [ ] Pruebas: `packages/herdr/src/hc/frames.test.ts`, `packages/herdr/src/hc/connection.test.ts`, `packages/herdr/src/hc/contract.test.ts`.

## Implementation Steps

### Step 1: paquete

- `packages/herdr/package.json`:
  ```json
  {
    "name": "@herdr/core-client",
    "private": true,
    "type": "module",
    "exports": {
      "./hc": { "types": "./src/hc/index.ts", "import": "./src/hc/index.ts" },
      "./gateway-api": { "types": "./src/gateway-api.ts", "import": "./src/gateway-api.ts" }
    },
    "scripts": {
      "typecheck": "effect-tsgo tsrs --noEmit",
      "test": "vp test run --config ../../vite.config.ts --dir ."
    },
    "devDependencies": { "@types/node": "catalog:", "vite-plus": "catalog:" }
  }
  ```
  Sin `dependencies`. `pnpm-workspace.yaml` ya incluye `packages/*`. Correr `vp i` en la raíz (cambia `pnpm-lock.yaml`, que está en la lista permitida).
- `packages/herdr/tsconfig.json`: igual que `packages/tailscale/tsconfig.json` (`extends ../../tsconfig.base.json`, `types: ["node"]`, `include: ["src"]`). Los tipos de Node están disponibles para las pruebas, pero el código de `src/` que no es prueba no los usa (lo verifica el criterio con `grep`); así el paquete se puede importar desde la app.
- **Fixture:** copiar `docs/tasks/herdr-core-mobile/hc-schema-0.1.18.json` a `packages/herdr/fixtures/hc-schema.json` (es `hc schema` de herdr-core 0.1.18, generación 1, OpenRPC).

### Step 2: `frames.ts`

Contrato (de `docs/tasks/herdr-core-mobile/hc-protocol.md`): cada trama es `[channel u32 LE][kind u8][len u32 LE][payload]` (cabecera de 9 bytes), payload de máximo 16 MiB.

| kind | nombre          | payload                                                       |
| ---- | --------------- | ------------------------------------------------------------- |
| 0    | DATA            | canal 0: un mensaje JSON-RPC; otros canales: bytes del stream |
| 1    | CREDIT          | u32 LE: bytes que el receptor consumió                        |
| 2    | CLOSE           | el canal terminó; desde el core, un JSON con la razón         |
| 3    | RESIZE          | u16 LE cols, u16 LE rows                                      |
| 4    | FOCUS           | 1 ganó foco, 0 lo perdió                                      |
| 5    | FACTS           | JSON de `Display`                                             |
| 6    | AUTOMATIC_INPUT | bytes de respuesta generados por la terminal                  |

Exportar:

- `export const FrameKind = { DATA: 0, CREDIT: 1, CLOSE: 2, RESIZE: 3, FOCUS: 4, FACTS: 5, AUTOMATIC_INPUT: 6 } as const;` y `type FrameKind = (typeof FrameKind)[keyof typeof FrameKind]` (no `enum`).
- `export const MAX_FRAME_PAYLOAD = 16 * 1024 * 1024;`
- `interface Frame { channel: number; kind: FrameKind; payload: Uint8Array }`.
- `encodeFrame(frame): Uint8Array` y ayudas `encodeCredit(channel, bytes)`, `encodeResize(channel, cols, rows)`, `encodeClose(channel)`.
- `class FrameDecoder { push(chunk: Uint8Array): Frame[] }` incremental: acumula bytes, devuelve todas las tramas completas de un chunk, conserva el resto para el siguiente; lanza `HcProtocolError` si `len > MAX_FRAME_PAYLOAD`. Sin copiar más de lo necesario (un buffer creciente con offset de lectura).

### Step 3: `connection.ts`

`HcConnection` sobre un transporte inyectado:

```ts
export interface ByteTransport {
  write(bytes: Uint8Array): void;
  onData(cb: (chunk: Uint8Array) => void): void;
  onClose(cb: (error?: Error) => void): void;
}
```

- **Modo plano → con tramas:** `HcConnection.open(transport, { clientName?: string }): Promise<HcConnection>` escribe en modo plano la línea `{"jsonrpc":"2.0","id":0,"method":"hello","params":{"framed":true,"generation":1}}\n`, lee bytes hasta el primer `\n`, parsea la respuesta (`HelloResult`), y desde ese momento todo lo que llega se decodifica como tramas (los bytes que vinieron después del `\n` en el mismo chunk ya son tramas; no perderlos). Expone `hello: HelloResult`. Si `result.framed !== true` o `generation !== 1`, rechaza con `HcProtocolError`.
- **JSON-RPC sobre canal 0:** `call<T>(method: string, params: object): Promise<T>`: id incremental, cada mensaje es una trama `DATA` en canal 0 con el JSON en UTF-8; las llamadas corren en paralelo y se resuelven por `id`. Error JSON-RPC → rechaza con `HcCallError { code, message, kind: data.kind, hint: data.hint }` (los errores traen `{"code":-32000,"message":"no terminal t9","data":{"kind":"not_found","hint":"..."}}`).
- **Notificaciones:** los mensajes sin `id` en el canal 0 se entregan a los oyentes de notificaciones.
- **`subscribe(params?: { terminal?: string; labels?: Record<string,string> }): Promise<HcSubscription>`:** llama `events.subscribe`; en modo con tramas la respuesta (`SubscribeResult`) trae `channel`, y los eventos llegan por ese canal como líneas JSON (JSON-RPC sin id, separadas por `\n`, posiblemente partidas entre tramas) con créditos: devolver `CREDIT` por los bytes consumidos. `HcSubscription = { seq: number; terminals: Terminal[]; onEvent(cb: (n: HcNotification) => void): void; close(): void }` donde `HcNotification` es `{ method: "event", params: { seq, type, terminal?, data, machine? } } | { method: "heartbeat", params: { seq } } | { method: "missed", params: { seq, terminals } }`. Si los eventos llegan por el canal 0 en vez del canal propio, entregarlos igual (robustez).
- **`attach(terminal: string, opts: { cols: number; rows: number; history?: number }): Promise<HcAttachStream>`:** llama `terminal.attach` con `{ terminal, cols, rows, history }` y usa el `channel` de `AttachResult`. Devuelve:
  ```ts
  interface HcAttachStream {
    channel: number;
    result: AttachResult;
    onData(cb: (bytes: Uint8Array) => void): void;
    onClose(
      cb: (
        reason: { reason: string; exit_code: number | null; signal: number | null } | null,
      ) => void,
    ): void;
    write(bytes: Uint8Array): void; // DATA del visor (teclas)
    resize(cols: number, rows: number): void; // RESIZE
    consumed(bytes: number): void; // devuelve CREDIT por bytes ya entregados
    close(): void; // manda CLOSE; el core contesta con CLOSE vacío
  }
  ```
  **Créditos (según `hc-protocol.md`):** el core da a cada stream de attach 256 KiB de crédito al abrirlo; el visor manda `CREDIT` (u32 LE en el payload, en el canal del stream) por los bytes que ya consumió; el cliente **nunca** manda un `CREDIT` inicial. El cliente tampoco devuelve crédito automáticamente al recibir: lo hace quien consume llamando `consumed(n)` (el gateway lo llama cuando entregó los bytes al socket; historia 5). Un `CLOSE` del core trae un JSON `{"reason":"exited","exit_code":0,"signal":null}`; un `CLOSE` vacío (respuesta a nuestro `CLOSE`) llama `onClose(null)`. Un visor más de 4 MiB atrasado recibe un repintado en lugar del atraso (lo hace el core; no hay que implementarlo). Los eventos de `events.subscribe` también llevan crédito por su canal: devolver `CREDIT` por los bytes procesados.
- **Cierre del transporte:** rechaza todas las llamadas pendientes con `HcClosedError`, llama `onClose({ reason: "connection_lost", exit_code: null, signal: null })` en cada stream abierto (así el gateway distingue la caída del core de un fin del programa), cierra las suscripciones y emite `onClose` de la conexión.
- `close()` cierra todo del lado cliente.

### Step 4: `types.ts`

Tipos (solo `interface`/`type`) tomados de `fixtures/hc-schema.json` (`components.schemas`), con los campos que el gateway usa:

- `Terminal`: `address`, `argv`, `cwd`, `state`, `created_ms`, `cols`, `rows`, `labels: Record<string,string>`, `title`, `turn`, `status: Status | null`, `status_ms: number | null`, `exit_code`, `finished_ms`, `reason`, `foreground: { pid, name, argv, cwd } | null` (`pid` es el líder del grupo de procesos en primer plano; `null` cuando terminó), `name`, `keep` (opcionales los no requeridos por el esquema; requeridos: `address`, `labels`, `argv`, `cwd`, `state`, `created_ms`, `screen`, `cols`, `rows`).
- `Status`: `state` (`idle`, `working`, `done`, `blocked`, `error`, … como `string`), `app?`, `kind?: string | null` (`permission`, `question`, `auth`), `msg?`, `title?`, `progress?`.
- `HelloResult` (`generation`, `version`, `core: { id, name }`, `framed`, `methods`, `features`), `AttachResult` (`channel`, `terminal`, `resize_notifications`, `automatic_input`), `SubscribeResult` (`seq`, `terminals`, `channel: number | null`), `ReadResult` (`text`, `complete`, `how`, `why`), y también `SendResult` (`turn`, `started`, `status`), `RunResult` (`address`, …), `DocGetResult` (`value`, `version`), `ListResult`, `KeysResult`, `InterruptResult` (`outcome`, `key`).

### Step 5: pruebas

- `frames.test.ts`: una trama partida en cada posición posible (byte a byte), varias tramas en un solo chunk, payload vacío, `len` mayor a 16 MiB → error, ida y vuelta de `encodeResize`/`encodeCredit`.
- `connection.test.ts` con un transporte falso en memoria que simula el core: `hello` en modo plano y cambio a tramas (incluyendo tramas pegadas en el mismo chunk que la línea de `hello`), dos `call` en paralelo resueltas en orden inverso, error JSON-RPC con `data.kind`, suscripción con eventos JSON partidos entre tramas y `CREDIT` devuelto por bytes consumidos, `attach` con `consumed()` → trama `CREDIT` correcta, `CLOSE` del core con razón JSON → `onClose` con `{reason, exit_code, signal}`, cierre del transporte → llamadas pendientes rechazadas.
- `contract.test.ts`: lee `fixtures/hc-schema.json` y verifica que cada método que usa el cliente o el gateway existe y que cada parámetro que mandamos existe y que los requeridos por el esquema se mandan: `hello` (`framed`, `generation`), `events.subscribe` (`terminal`, `labels`), `terminal.attach` (`terminal`, `cols`, `rows`, `history`), `terminal.list`, `terminal.get` (`terminal`; lo usa la historia 16), `terminal.send` (`terminal`, `text`), `terminal.keys` (`terminal`, `keys`), `terminal.interrupt` (`terminal`), `terminal.run` (`argv`, `cwd`, `keep`, `labels`), `doc.get` (`name`). Si `process.env.HERDR_LIVE_HC === "1"`, corre `hc schema` con `node:child_process` y compara con el fixture (mismo `info.version` y mismos métodos y parámetros); si no, ese caso se omite con `it.skipIf` (no cuenta como `it.skip` sin razón).

## Out of Scope

- El gateway HTTP/WebSocket, el proceso `hc connect` y la reconexión: historia 4.
- Los tipos del API del gateway (`gateway-api.ts` con contenido): historia 4 (de chat: historia 13).
- Traducir bytes a UTF-8 o al formato de la app: historia 5.
- `AUTOMATIC_INPUT`, `FOCUS` y `FACTS` más allá de codificarlos: no se usan en la v1.
- Máquinas remotas de `hc machine` (direcciones `machine/t7`): fuera del alcance; cada gateway representa a su servidor.

## Acceptance Criteria

- [ ] `packages/herdr/fixtures/hc-schema.json` existe y `info.version` es `0.1.18`.
- [ ] Dentro de `packages/herdr`: `vp test run src/hc/frames.test.ts src/hc/connection.test.ts src/hc/contract.test.ts` pasa.
- [ ] Dentro de `packages/herdr`: `vp run typecheck` pasa.
- [ ] `grep -rn "@t3tools/\|from \"node:\|Buffer" packages/herdr/src --include='*.ts' | grep -v '\.test\.ts:'` no encuentra nada fuera de archivos `*.test.ts`.
- [ ] Opcional si hay `hc` en la máquina: `HERDR_LIVE_HC=1 vp test run src/hc/contract.test.ts` pasa.
- [ ] Si `scripts/herdr/check-patch-surface.ts` ya existe (historia 1 corre en paralelo), `node scripts/herdr/check-patch-surface.ts` sale con código 0.

_(Feature-level end-to-end verification is NOT done per story — that is the PRD's `## Definition of Done`, run once against the whole feature.)_

## Estimated Effort

M
