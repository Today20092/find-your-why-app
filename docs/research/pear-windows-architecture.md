# Pear Windows architecture decision

## Decision

Build the first MVP as a standard Windows Electron application based on Holepunch's official [`hello-pear-electron`](https://github.com/holepunchto/hello-pear-electron) scaffold.

Use three boundaries:

1. **Renderer:** responsive HTML/CSS/JavaScript UI with no Pear or storage imports.
2. **Electron main/preload:** a thin, explicit IPC bridge and Windows application shell.
3. **Bare worker (the Pear-end):** identity, signed events, Corestore persistence, replication, local indexes, and trust/discovery calculations.

This follows Pear's documented production architecture: the Electron main process hosts the window and forwards IPC, while a Bare worker owns `pear-runtime` and the P2P data plane ([desktop architecture](https://docs.pears.com/explanation/pear-desktop-architecture/), [runtime and languages](https://docs.pears.com/explanation/runtime-and-languages/)). It also preserves the project's established guarantees: Following remains chronological and complete for available followed posts; Discover remains separate, bounded, positive-trust-based, explainable, and local.

Do not start from PearPass itself. Pear's docs say PearPass uses the same official Electron template shape, but PearPass is a mature password manager with vault encryption, localization, browser integration, and a multi-package workspace that this MVP does not need ([template guide](https://docs.pears.com/getting-started/from-a-template/start-from-hello-pear-electron/), [PearPass desktop source](https://github.com/tetherto/pearpass-app-desktop)).

## Concrete runtime and modules

Keep the first dependency set close to the official scaffold:

- `electron` and Electron Forge for the Windows shell and distributable;
- `pear-runtime` to spawn the Bare worker, provide application storage, and later apply P2P OTA updates;
- `framed-stream` for message boundaries over the worker duplex stream;
- `corestore` for persistent Hypercores;
- `hyperswarm` for peer discovery and encrypted connections;
- small lifecycle/runtime helpers already used by the template (`graceful-goodbye`, `paparam`, `which-runtime`).

The current official manifest is the version source of truth rather than versions copied into this decision ([`hello-pear-electron/package.json`](https://github.com/holepunchto/hello-pear-electron/blob/main/package.json)). Add Autobase/HyperDB or Hyperbee only when implementing the replicated event materialization ticket; they are not required merely to boot the vertical slice. Pear's own production-chat tutorial introduces persistent `Corestore`, an Autobase room, blind-pairing invitations, and a HyperDB view at that stage ([production reshaping tutorial](https://docs.pears.com/getting-started/build-a-peer-to-peer-chat/reshape-into-a-production-app/)).

The worker receives its storage root as an argument from `PearRuntime.run(...)` and reads it from `Bare.argv`; the renderer reaches it only through the preload bridge ([`pear-runtime` reference](https://docs.pears.com/reference/pear/runtime/)). Validate every renderer command at this boundary and expose named product operations rather than a general Node/Pear API.

## Minimum repository shape

Start by retaining the template's production-shaped minimum:

```text
package.json              scripts, dependencies, upgrade link
forge.config.js           Electron packaging and Windows MSIX maker
pear.json                 deployment metadata
build/                    icons and AppxManifest.xml
electron/main.js          window, storage location, worker lifecycle
electron/preload.js       narrow contextBridge API
workers/main.js           Pear-end entry point
renderer/index.html       accessible application document
renderer/app.js           UI state and bridge calls
renderer/app.css          responsive presentation
```

This is deliberately not a frontend framework decision. The renderer is a normal web view, so semantic HTML, CSS, and browser JavaScript are sufficient for the first vertical slice. Introduce a framework only if the UI prototype demonstrates state complexity that materially reduces with one.

The template's current source is authoritative for the detailed worker plumbing, storage paths, update flow, and deep-link/single-instance handling ([Electron main](https://github.com/holepunchto/hello-pear-electron/blob/main/electron/main.js), [preload bridge](https://github.com/holepunchto/hello-pear-electron/blob/main/electron/preload.js), [Forge configuration](https://github.com/holepunchto/hello-pear-electron/blob/main/forge.config.js)).

## Windows development and packaging

Official prerequisites are Node.js 22.17 or newer, npm 10.9 or newer, and a POSIX-style terminal on Windows (the documentation recommends WSL for the tutorial workflow) ([getting started](https://docs.pears.com/getting-started/)). The template workflow is:

```powershell
git clone https://github.com/holepunchto/hello-pear-electron
cd hello-pear-electron
npm install
npx pear touch
# Put the returned pear:// link in package.json#upgrade
npm start
```

`npm start` runs Electron Forge with `--no-updates`, preventing a released build from replacing local development code ([template guide](https://docs.pears.com/getting-started/from-a-template/start-from-hello-pear-electron/)). Run two isolated local identities by forwarding separate `--storage` paths, as shown by the production tutorial.

For a Windows distributable:

```powershell
npm run make
Add-AppxPackage .\out\<App>-win32-x64\<App>.msix
```

The official template produces an MSIX and requires Windows SDK plus PowerShell 7+. An unsigned development build gets a machine-local self-signed certificate; production builds need a persistent code-signing certificate, and its publisher must match `build/AppxManifest.xml`. OTA-compatible updates must keep the same certificate ([template release instructions](https://github.com/holepunchto/hello-pear-electron#3-make-distributables)). Application deployment is a later, separate path: make the distributable, run `pear build`, then stage/provision/multisign the deployment directory ([migration guide](https://docs.pears.com/how-to/operate-an-app/migration/), [deployment guide](https://docs.pears.com/how-to/operate-an-app/manual-deployment/deployment)).

## PearPass patterns worth reusing

PearPass confirms these patterns at product scale:

- a thin desktop shell around local-first P2P state;
- separate desktop UI and portable core packages;
- offline access with explicit device pairing rather than a central account service;
- a narrow bridge between UI and the local engine;
- explicit build output, staging exclusions, and platform packaging;
- separate mobile hosts rather than assuming Electron runs on phones.

PearPass's official description identifies its reusable backend ingredients as Autobase, HyperDB/Hyperbee, Blind Pairing, and Hyperswarm, with the local engine owning encryption, storage, pairing, and replication ([official PearPass announcement](https://pears.com/uncategorized/introducing-pearpass-a-fully-local-open-source-password-manager/)). Its desktop manifest shows a mature TypeScript/bundled renderer and workspace packages, but those are evidence of a possible later scale, not an MVP requirement ([PearPass manifest](https://github.com/tetherto/pearpass-app-desktop/blob/main/package.json)).

Do **not** copy PearPass's vault encryption model, browser-extension bridge, localization stack, telemetry/error-reporting integrations, custom bundlers, or broad workspace layout. Social posts are public signed data, and those features solve different requirements.

## Unsupported assumptions and platform boundary

- **A normal browser is not a Pear peer.** The portable Pear-end runs in Bare and the desktop host uses `pear-runtime`; a browser-only deployment would need a gateway, extension/native companion, or a different protocol implementation. None is part of the Windows MVP.
- **The desktop renderer is reusable design, not guaranteed reusable code.** HTML/CSS concepts and domain message schemas can carry forward, but mobile uses React Native with Bare iOS/Android worklets. `pear-runtime` is not currently supported on mobile ([runtime and languages](https://docs.pears.com/explanation/runtime-and-languages/), [one core, many platforms](https://docs.pears.com/explanation/bare-on-native/)).
- **PearPass is proof of architecture, not proof that its UI repository can be dropped in.** Its mobile and desktop hosts are distinct products.
- **Serverless does not mean always available.** If every holder of a post is offline, a new reader cannot retrieve it. Host peers/blind peers remain a later availability feature ([availability and blind peering](https://docs.pears.com/explanation/availability-and-blind-peering/)).

## First implementation slice

Clone the official scaffold's structure, rename it, and replace the sample worker with one product operation flowing end to end:

1. renderer requests local identity creation;
2. preload sends a named, validated command;
3. worker persists the identity under the supplied Pear storage root;
4. worker returns the public identity and key-derived discriminator;
5. renderer displays it.

Then add signed text publishing and two isolated local instances. Defer OTA release plumbing beyond keeping the template intact, replicated multiwriter views, QR rendering, recovery, global lookup, browser/mobile hosts, host peers, and production signing until their mapped decisions are resolved.

## Newly surfaced fog

1. Which event-log topology best preserves independent publisher ownership: one writable core per identity, or an Autobase-backed social space?
2. Is Blind Pairing appropriate for following a public publisher, or should public invite links carry only the publisher's discovery information while recovery/device pairing uses a separate protocol?
3. What exact command schema crosses the renderer/worker boundary, and which validation/size limits apply?
4. Should the first Windows artifact be an installable MSIX or a development-only Electron run until code signing is funded?
5. What browser architecture, if any, can preserve the direct-follow guarantee without creating a trusted central gateway?
