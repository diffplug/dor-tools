# Dor Tools

A small integration protocol for terminal-launched web tools and the hosts that
run them. A Tool can announce its local web server, report unsaved changes, and
ask its host to open a local file. An iframe channel lets a cooperating host ask
an editor to save before closing it.

The TypeScript library is MIT-licensed, has no runtime dependencies, and builds
to ESM JavaScript with type declarations. [Dormouse](https://github.com/diffplug/dormouse)
is its first host; the library can also be embedded in other hosts.

> **Draft extraction — V1 is still evolving.** This repository is a snapshot,
> not yet the development source of truth. Protocol changes continue in
> Dormouse's `dor-tools-lib/` alongside its host implementation. The package
> remains `private: true` at `0.0.0`; there is no stable npm release or API
> compatibility promise yet. The repository is named `dor-tools`, while the
> package and import paths retain `dor-tools-lib` for now.

## Try it locally

Use Node.js 24 or newer and pnpm 12.6.0 (also recorded in `package.json`).

```sh
git clone https://github.com/diffplug/dor-tools.git
cd dor-tools
pnpm install
pnpm test
```

`pnpm test` builds the package and runs the Node.js tests. `pnpm build` produces
`dist/`; `pnpm pack` creates a local package archive without publishing it. To
experiment in another project, build this checkout, then run
`pnpm add /absolute/path/to/dor-tools` in that project. Rebuild after source edits.

## Announce a web tool

A Node.js Tool writes OSC 367 sequences to its terminal output. Announce the
actual port only after the server is listening:

```js
import { createServer } from 'node:http';
import { serveSequence } from 'dor-tools-lib/osc';

const server = createServer((_request, response) => {
  response.writeHead(200, { 'content-type': 'text/html; charset=utf-8' });
  response.end('<!doctype html><title>My Tool</title><h1>Hello from a Tool</h1>');
});

server.listen(0, '127.0.0.1', () => {
  const { port } = server.address();
  process.stdout.write(serveSequence({ port, path: '/' }));
});
```

Save this as `server.mjs` in the consuming project and declare it in that
project's `dormouse.yml`:

```yaml
tools:
  # A minimal local web Tool; announces its port after listening.
  hello:
    run: node server.mjs
    render: iframe
    port: announced
    prespawn_dedupe: [hello, $PROJECT_ROOT]
```

Run `dor tool hello` in Dormouse and approve the project configuration if
prompted. Dormouse presents the Tool's web page while retaining its terminal.
An OSC announcement alone does not turn an ordinary terminal into a Tool or
grant access to an arbitrary port: Dormouse checks the designated Tool's process
tree. This example serves fixed content; a real Tool owns its HTTP access and
file-access checks.

## Report unsaved changes and open files

These calls run in the Tool's terminal process, not in its browser page:

```js
import { openSequence, stateSequence } from 'dor-tools-lib/osc';

process.stdout.write(stateSequence({ dirty: true }));
// After the current edits have been saved:
process.stdout.write(stateSequence({ dirty: false }));

// Use an absolute local path. Omit preview to open normally.
process.stdout.write(openSequence({ path: '/absolute/path/notes.md', preview: true }));
```

A browser editor needs its own route to tell its terminal process about edits.
Dirty state is independent of serving metadata, and a clean report is not a
save acknowledgement. Dormouse retains the last report when the command exits
and resets it when a new command starts. Open requests are fire-and-forget;
Dormouse acts only on live output from a running, designated Tool.

## Connect an iframe editor

`connectToolFrame` implements the Tool side of the save channel:

```js
import { connectToolFrame } from 'dor-tools-lib/frame';

// editor is your application's document model.
const frame = connectToolFrame({
  dirty: () => editor.loading ? undefined : editor.dirty,
  save: () => editor.save(),
});

// Call after loading and whenever the document's dirty state changes.
editor.onChange(() => frame.report());
// When disposing the page's integration, call frame.close().
```

This is an integration sketch: supply your own editor and persistence logic.
`save()` returns a promise that resolves after writing the document; it must
leave `dirty()` true if newer edits remain. Errors are returned to the host.
Outside a frame, or before a host connects, reports go nowhere. A reconnect or
`close()` retires pending replies from the previous connection.

**Current Dormouse limitation:** the host connects this channel only to its
built-in file editor. Third-party Tools can use OSC dirty reports now, but must
save through their own UI; importing `frame` does not enable Dormouse's Save
button for them. Other hosts can implement the channel using `protocol`.

## API entry points

There is no root export. Import one of these subpaths:

| Import | Exports | Use |
| --- | --- | --- |
| `dor-tools-lib/osc` | `serveSequence`, `stateSequence`, `openSequence` | Encode terminal announcements and requests; invalid inputs throw. |
| `dor-tools-lib/osc` | `parseToolAnnounce`, `parseToolState`, `parseToolOpen`, `parseToolPayload`, `validToolServePath`, `validToolOpenPath` | Parse or validate Tool output in a host. |
| `dor-tools-lib/frame` | `connectToolFrame` | Connect a browser editor to its parent host. |
| `dor-tools-lib/protocol` | `DOR_TOOL_VERSION`, `readHostMessage`, `readFrameMessage` | Validate the iframe channel's messages in either direction. |

Types are exported alongside their functions. Start at [src/osc.ts](src/osc.ts),
[src/frame.ts](src/frame.ts), or [src/protocol.ts](src/protocol.ts) for the exact
signatures, field shapes, and validation rules.

### Host integration notes

- Feed OSC parsers the content after `367;`, excluding the escape introducer and
  terminator: for example, `state;{"v":1,"dirty":true}`. They return `null` for
  malformed or unsupported messages. The host supplies terminal framing;
  encoders emit BEL-terminated sequences.
- All encoders emit version 1. `serve` also accepts an omitted version for
  compatibility; `state` and `open` require `v: 1`. JSON payloads are bounded to
  4,096 JavaScript string code units. A serve path is a path/query on the
  discovered port, never another authority.
- Announcements are untrusted hints. Port ownership, Tool designation, process
  lifecycle, permissions, and open-file dispatch belong to the host. Replaying
  terminal history must not repeat open requests.
- The iframe host sends `connect` with a per-mount connection nonce, then `save`
  with a request id. The frame sends `ready`, `state`, and `saved`. Every message
  carries `dorTool: 1` and the connection nonce.
- Message validators check data shapes; hosts must also check the sending
  window, origin, current connection, and outstanding save request. A Save
  closure requires a successful matching acknowledgement with no newer edits;
  timeout or disconnection is not success. The library contains no host UI or
  host-side connection manager.

## Development status and provenance

The initial `src/` and `test/` trees are copied unchanged from
[`diffplug/dormouse` at `716aac2574afcd6ec8e2b756f52eaeb4aabe2856`](https://github.com/diffplug/dormouse/tree/716aac2574afcd6ec8e2b756f52eaeb4aabe2856/dor-tools-lib).
The original package remains in Dormouse and its consumers continue using it.
There is no automatic synchronization between repositories. Until the cutover,
make protocol changes in Dormouse, then copy reviewed snapshots here and update
this provenance record. This repository adds standalone build metadata, a
lockfile, CI, and this README.

References to `docs/specs/...` in the copied source and tests refer to the
Dormouse repository. Its [Tool contract](https://github.com/diffplug/dormouse/blob/716aac2574afcd6ec8e2b756f52eaeb4aabe2856/docs/specs/dor-tool.md)
and [library spec](https://github.com/diffplug/dormouse/blob/716aac2574afcd6ec8e2b756f52eaeb4aabe2856/docs/specs/dor-tools-lib.md)
describe this snapshot; their `Future` sections describe unbuilt work.

Before a stable release, planned work includes third-party save coordination,
theme subscriptions, additional host capabilities, a standalone protocol
reference, and publishing setup. Parsed `name`, `dehydrate`, and `persist` serve
fields are reserved and inert in Dormouse today; no dehydration handshake is
implemented. OSC 367 still needs its collision review before the wire contract
is frozen.

## License

[MIT](LICENSE), copyright 2026 DiffPlug LLC. Only the MIT Tool protocol package
is copied here; Dormouse's application code and built-in Tools are separate.
