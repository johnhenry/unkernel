# The SDK (`js/`, npm `unkernel`)

The SDK is the host runtime, shared by the CLI, the page and npm users.
README.md documents its public API. This file covers the rules and the
parts that are easy to get wrong.

## Rules

- Node 24 runs the sources directly (`node js/cli.ts`,
  `node --test test/*.test.ts`). `pnpm install` is needed only for
  `pnpm typecheck` (`tsc`, TypeScript 7) and `pnpm build` (obuild).
- `test/api.test.ts` runs its tests concurrently (one per core,
  `UNKERNEL_TEST_CONCURRENCY=1` for one at a time). Boot with
  `boot(t, opts)`, so that each test stops only its own kernels. Tests that
  can share one kernel's boot options go in a `describe` with
  `{ concurrency: 1 }` and `sharedKernel()`. A test that changes a global
  (`crossOriginIsolated`) goes after the concurrent suite. The two 9p
  suites are the longest, since their tests run in order.
- Use TypeScript's erasable syntax only (no enums, namespaces or parameter
  properties), relative imports with `.ts`, `import type` for types, and
  tabs. Every file starts with the SPDX MIT header and a copyright line.
  There are no runtime dependencies.
- `js/core/` must stay universal: no `node:` imports and no DOM-only APIs
  without a guard. Node-only code goes in `js/node/`, browser-only code in
  `js/web/`.
- Nothing may import `js/node.ts` or `js/web.ts` (the CLI uses
  `js/node/platform.ts`). Rolldown moves an entry module that another
  entry imports into a shared chunk, which would break the kernel URL
  below.
- The page builds from these sources (`web/src/sdk.ts`), so SDK changes
  reach it without `pnpm build`.

## Build (`pnpm build`, `build.config.ts`)

- Entries `js/node.ts`, `js/web.ts` and `js/cli.ts` become `dist/*.mjs`
  and `.d.mts`, with shared code in `dist/_chunks/`. A copy of
  `vendor/build/bundle/unkernel.wasm` (the base bundle) goes along as
  `dist/unkernel.wasm` (export `unkernel/unkernel.wasm`), and the build
  fails if the bundle is missing. `js/unkernel.wasm` is a symlink to that
  bundle. No distro image is in the package (Distro images, below).
- The build checks that `dist/web.mjs` reaches no Node built-in, that
  each entry (not a shared chunk) computes
  `new URL("./unkernel.wasm", import.meta.url)`, and that `dist/node.mjs`
  and `dist/cli.mjs` compute the package root (`new URL("..",
  import.meta.url)`, for `fetchDistro()`) themselves.
- It inlines the kernel workers (Packaging, below) and the CPU workers
  (`js/node/cpu-entry.ts`, started by `js/node/cpu.ts`, and
  `js/web/web-cpu-entry.ts`, started by `js/web/cpu.ts`; CPU threads,
  below) and checks that `dist/node.mjs` and `dist/cli.mjs` carry the
  Node kernel and CPU workers and `dist/web.mjs` the web ones (and not
  the others), and that no dist file mentions `worker-entry` or
  `cpu-entry`. Each CPU worker is inlined into its platform's kernel
  worker bundle too (`nested` in WORKERS), since in worker mode that
  worker starts the CPUs. The WASI program workers (`js/node/wasi-entry.ts`
  started by `js/node/wasi.ts`, `js/web/web-wasi-entry.ts` started by
  `js/web/wasi.ts`; WASI programs, below) are inlined the same way, into
  the Node entries and the web entry, and no dist file may mention
  `wasi-entry`.
- Test the built package with `node dist/cli.mjs` and
  `UNKERNEL_DIST=1 node --test test/api.test.ts`, which imports the package
  by its own name.

## Behaviour to keep

- `createKernel()` (`js/core/instance.ts`) wraps the `Kernel` class
  (`js/core/kernel.ts`: imports, JSPI, IRQs). With no `kernel` option it
  loads the bundle, with no `distro` it boots the base shell
  (busybox, the kw tools, kwtty; kernel/guest.md), and with no `net`
  option there is no network device. Nothing in `createKernel()` downloads
  a distro: the caller passes one (`fetchDistro()` on Node, a URL in
  browsers).
- Programs run in a running guest through `vm.spawn()` and `vm.exec()`
  (kwtty, below; over vsock, which is on unless `vsock: false`), on a pty
  or on pipes; `vm.connect()` and `vm.fetch()` reach TCP servers on the
  guest's loopback through kwtty too. The other channels are the hvc0
  console, vsock streams to guest programs that listen or connect
  themselves, host file systems mounted over 9p (`mounts`, `vm.mount()`;
  below), and guest shmem files (`shm` option). kwtty is the only guest
  service; there is no other guest agent (kw9p is a one-shot program that
  kwtty runs, and kwcmd, the process of a host command, `commands.ts`,
  connects to the host by itself). A one-shot program can be the init (`rdinit=`).
- `stop()` parks the kernel at its next suspension point (kernel.md,
  Tasks). The guest gets no shutdown, so prefer typing `poweroff`.
- `sync()` (`Kernel.requestSync()`) raises IRQ 4 and resolves true once
  every persistent disk has completed a flush issued after the call, or
  false after the timeout (3 s by default). It then clears the bit itself,
  because a kernel without IRQ 4 never does. Host file systems need no
  sync: each guest write(2) reaches the handler before it returns.
- Snapshots (`js/core/snapshot.ts`; kernel/parking.md), inline and in
  worker mode (Worker mode below). `KernelInstance.snapshot({ stop })`
  (`Kernel.snapshot()`) raises IRQ 8 and, when the kernel calls `wasm_snapshot_point` at its consistent
  point, copies the memory (without the pages the kernel reports free,
  `copySnapshotMemory()`: they stay zero, and `KernelSnapshot.freeBytes`
  says how much was left out) and seals every disk. It resolves with a
  `KernelSnapshot` (the whole memory, the kernel's `linux_banner` as its
  fingerprint, the guest's monotonic time, the console size, unread
  console input, each disk's `DiskSnapshotState` and the other devices'
  `DeviceSnapshots`) once the disks are durable. Without `stop` the guest
  keeps running; `stop: true` calls `Kernel.stop()` inside the import, so
  no task runs after the copy. It rejects when a disk cannot be in a
  snapshot, when a task does not park (the error names it), when block,
  net, vsock or shm queues are not empty at that point, or a 9p request
  is in flight (`VsockDevice.blocker()`), and while a mount or unmount
  runs (`MountTable.snapshotBlocker()`; in worker mode while anything is
  mounted, Host file systems below).
  `createKernel({ restore })` checks the disks (`snapshotDiskMismatch()`)
  and the devices (`snapshotDeviceMismatch()`: display, sound, vsock, shm
  and network each configured as when saved, the network with the same
  MAC, since the restored kernel has exactly the drivers it had); a
  mismatch rejects naming it, with no fallback. It then instantiates the
  same build, copies the memory in and enters `kw_resume`;
  `WasiHost.monoOffset` keeps the guest's monotonic clock going from the
  saved value. `encodeSnapshot()`
  leaves out zero 64 KiB chunks (4 KiB pages would save another 6%,
  2026-11-15, so the format stays). `snapshot()` is on `KernelHandle`.
  The CLI's `{save=FILE}`, `{suspend=FILE}` (`--input` tokens) and
  `--restore FILE` are the test hooks; `test/api.test.ts` round-trips a
  shell through one, and one with a data disk.
- Disks in snapshots (core/block.ts, data-disk.md): disks are not copied.
  `BlockDevice.snapshotState()` gives a disk's identity, write generation
  and (files) a stamp; `snapshotSeal()`, called at the consistent point,
  makes the content durable and starts a new generation at the next
  write. The generation goes up before the first write after the disk is
  opened and after each seal, so a restore accepts a disk only if nothing
  wrote it since the snapshot: not the guest running on, not a later boot,
  not another program. CowDisk with a store saves the generation in the
  same transaction as the blocks (`OverlayStore.save()`'s `meta`;
  `StoreIndex.gen` and `uid`); its seal takes the dirty blocks
  synchronously (no flush can be queued: nothing is in flight) and saves
  them as the next save in the chain, so later guest writes go to later
  saves. FileDisk keeps it in `<image>.kwsnap` (`{"uid", "gen"}`, created
  by the first snapshot that includes the image, then rewritten and
  synced before the first write of each generation); the image's size and
  mtime are the stamp. A CowDisk without a store can be in a snapshot only
  while unwritten and over a base with an identity (FileBase: path, size,
  mtime; HttpBase: URL, size, ETag). A disk without these methods (a
  custom BlockDevice) makes `snapshot()` reject. The distro image
  (Disks, below) is the snapshot's first disk; its identity is its
  content (`DistroDisk`), with the underlying disk's stamp.
- Devices in snapshots (`Kernel.saveDevices()`/`restoreDevices()`): each
  host class has `saveState()` and `restoreState()` for what the kernel
  told it once and does not tell a restored host again. A key in
  `DeviceSnapshots` means the device was configured; null means its
  driver had not started (it then starts after the restore, as at boot).
  Host-only, except vsock:
  - display: the attachment (address, mode, the size offered);
    `restoreState()` reports it to `onDisplay` again. The pixels are in
    the memory.
  - input: the open mask, unread events, and the keys and buttons the
    guest saw pressed; the restore releases those (the host's keys are
    not held in the new session) and reports the open state to `onInput`.
  - sound: the stream state; the restore replays open, prepare and start
    to the sink. The kernel's timer goes on on the guest's clock, which
    the restore continues.
  - shm: the exports and their page runs (the kernel keeps the pages
    held), so `openId()` works again; exports the old host opened stay
    held in the kernel until closed by id.
  - vsock: the guest CID and `rx_max`; the restore replays `onVsockInit`.
    Connections do not survive, except host file systems' (`mounts`,
    below): after a restore the kernel resets every connected socket the
    host does not keep (kwvsock's park restore notifier asks
    `wasm_driver_vsock_keep`, answered from `VsockDevice.keep`, kernel/parking.md),
    so kwtty's programs, `--forward` and `connect()` relays and guest
    clients end, and listeners stay. The CLI's `--forward` listeners and
    the SDK's `vsock.listen()` are the new host's own.
  - network: the MAC kwnet got; the restore calls `NetDevice.attach()`
    again. The stack keeps no state across it: a guest TCP segment of an
    old flow gets an RST (`NetStack`, no such connection), so a flow ends
    at its next send; an idle one lingers until the guest's own timeouts.
    DHCP and ARP need nothing (fixed addresses and gateway MAC).
  `test/api.test.ts` restores glimmer mid-test (picture, open devices,
  stream, a key) and a machine with network and vsock (old sessions end;
  wget, exec() and fetch() work after).
- Host file systems in snapshots (inline only; mounts.ts, 9p.ts): the 9p
  server runs on the kernel's thread, so `VsockDevice.saveState()` saves
  the mounts at the snapshot point (`VsockSnapshot.mounts`:
  `MountTable.save()`). Every guest task is parked and a 9p request holds
  its task (killable wait) until answered, so the sessions are quiet
  there; one that is not (`MountTable.snapshotBusy()`, through
  `VsockDevice.hostBusy`) makes the snapshot point busy. Per mount: its
  path, read-only flag, export name, the vsock connection's ports and
  credit counters (`Connection.saveState()`), msize and the fids (path,
  type, for an open file how it was opened and its qid.path, a directory
  listing being read). The files are not copied: `createKernel({
  restore, mounts })` must give a file system for each saved path,
  read-only as saved (`snapshotMountMismatch()`, checked before boot;
  worker mode and `vsock.raw` refuse such a snapshot). Before the kernel
  runs, `MountTable.restore()` serves each under the saved name on the
  connection `VsockHost.adopt()` takes over, opens the open files again
  by path (`Session.restore()`) and sets `VsockDevice.keep`; other given
  mounts are mounted once the kernel runs. A file gone, another file now
  (another qid.path), unlinked or replaced by the guest while open
  (`Fid.orphan`), or not opening within the request timeout leaves its fid
  stale: its reads, writes, getattr, setattr and fsync answer ESTALE
  ("Stale file handle" in the guest). A MemoryFs passed again keeps the
  guest's changes; a new one has its own content, which the guest sees
  within 1 s, as any host change. `test/9p.test.ts` saves and restores a
  session on a real VsockHost (GuestVsock: the guest's end of the vsock
  protocol), and `test/api.test.ts` a process with files open on a nodeFs
  mount, and a MemoryFs mount, across a restore.
- Any host that instantiates the kernel itself must provide the three
  `unkernel.kw_host_jit_*` imports (caps returning 0 is enough), and the
  display, input, sound, vsock and shm imports (kernel/devices.md; `_init`
  returning -1 is enough, as `test/wasi-node.ts` does, which also answers
  `wasm_driver_vsock_keep` with 0). `Kernel.imports()`
  fills a missing import with a stub that throws when called, so an old
  host on a new kernel traps at that device's init.
- Display, input and sound (`js/core/display.ts`, `input.ts`, `sound.ts`)
  are off unless asked for; `display` also enables the evdev devices.
  Presses and motion are queued only while a program has the device open;
  a release always goes through if the guest saw the press, and keys still
  down when the device closes are released, so nothing sticks. Sound's
  `onChunk` returns the frames the sink has queued, which the guest sees
  as delay. The CLI's `--display`, `--sound`, `--sound-wav` and the
  `--input` tokens are the test hooks (`--help`).
- `KernelHandle` (instance.ts) is the surface shared by `KernelInstance`
  and `WorkerKernel`; a new device member goes there.
- `Kernel.boot()` retries with the other memory type if instantiation
  fails, so `jit.ts` and the imports read the memory and its type at call
  time.

## Worker mode (`createKernel({ worker: true })`)

- Works on Node and in browsers, with disks, network, display, evdev,
  sound, vsock and shm. Owner side `js/core/worker-host.ts`
  (`WorkerKernel`), worker side `js/core/worker-serve.ts` (`serveKernel()`,
  which calls `createKernelOn()`, so defaults, progress and errors match
  inline mode), messages `js/core/worker-protocol.ts`; all universal, no
  DOM. Bindings (`Platform.spawnWorker`): Node `js/node/worker.ts` and
  `worker-entry.ts`; browsers `js/web/worker.ts` (a module Web Worker) and
  `web-worker-entry.ts`, with `js/web/platform.ts` (shared by `js/web.ts`
  and `web/src/sdk.ts`). `autoMemoryMiB()` reads `navigator`, so it runs
  on the owner's thread and goes in the boot message.
- Types: `worker: true` gives `WorkerKernel`, none or false
  `KernelInstance`, a boolean `KernelHandle`. The option is its own
  interface, `WorkerOption`, not part of `CreateKernelOptions`, so code
  passing plain `CreateKernelOptions` (`web/src/sdk.ts`) still gets
  `KernelInstance`. `WorkerKernel` has no `kernel`; its `stats()` and
  `disks` (`DiskState`) are the worker's latest snapshot (every 500 ms and
  at exit), with `uptimeMs` brought up to date.
- Kernel loading: `createKernelInWorker()` spawns the worker and, while
  it starts, compiles the kernel on the owner's thread
  (`loadKernelModule()`: compileStreaming for URLs, a file read on Node)
  and posts the `WebAssembly.Module` in the boot message, so the
  `worker` progress stage may come before, during or after `compile`,
  `download` and `compiled`. If posting the Module throws and the source
  was a location or bytes, the worker gets those and loads the kernel
  itself (a `[host]` log line); this path has never run, since no such
  engine is known. `compileInWorker: true` always sends the location (an
  absolute URL via `Platform.absoluteUrl`) or transferred bytes; the
  initrd always goes that way. The worker never uses its own
  `import.meta.url`. `load.ts` tests for a URL or path before `instanceof
  Response`: on Node the first touch of `Response` loads undici (~23 ms).
- Messages (`worker-protocol.ts`) are objects tagged with `t`; each side
  ignores unknown tags. The worker's first message is `ready` (with
  whether it has JSPI; a missing JSPI is a `boot-error` of stage
  "support"). With the shared transport the owner sends no `input`,
  `resize`, `evdev` or (while a frame buffer exists) `frame-want`.
- Console output from a worker is posted once per microtask, so only when
  the kernel suspends. Posting every write made `seq 1 200000` take 38 s
  instead of 6 s; keep the batching. With more than one CPU and the shared
  transport it does not come this way: the owner reads the console ring
  (CPU threads, below).
- `ConsoleOutput` (`output.ts`, the `output` stream and `onOutput()` of
  both modes) keeps up to 1 MiB for a stream nobody has read yet. It
  drops old chunks by index: `Array.shift()` on that array (some 300k
  chunks of a line or less) copied it every time, so once 1 MiB was out
  every chunk cost about 100 µs. A UP kernel inline in the CLI (which
  only uses `onOutput`) took 95 s for `seq 1 300000` instead of 6.3 s
  (2026-11-17).
- Measuring console speed: keep a consumer's per-chunk cost constant.
  Appending each chunk to one string and slicing its end flattens the
  whole string every chunk, which made `seq 1 200000` on CPU 1 (one
  chunk per write then) look like 270 s.
- Device specs (`js/core/devices.ts`): disks and the network as plain,
  structured-cloneable data tagged with `kind` (`DEVICE_SPEC_VERSION` 1).
  A `DeviceRegistry` builds them (`Platform.devices`, passed to
  `serveKernel()` too): `coreDevices` has the universal kinds,
  `js/node/devices.ts` and `js/web/devices.ts` add the platform ones. A
  `data` store the page cannot use gives a fresh in-memory disk with a
  `notice`; nothing is cleared. An `http` spec with `readOnly` has no
  overlay (`BaseDisk` over a strict `HttpBase`). The `distro` option
  crosses as one more spec (`WorkerBootSpec.distro`), made on the owner
  (`Platform.distroSpec`, so a relative URL resolves against the page);
  the worker's disk indices skip it. Inline `createKernel` builds specs the
  same way, owns those disks and closes them at exit after their last
  save. Worker mode takes specs only: a `BlockDevice` or `NetDevice` object
  rejects with a `BootError` naming the spec to use. A `NetBackend` object
  is the exception: it stays on the owner's thread and the worker's
  NetStack reaches it over a port (Network, below).
- `sync()` in worker mode is a round trip to `Kernel.requestSync()`. A
  request open at exit is answered false; one left when the worker died
  resolves false if a persistent disk was attached, else true; after exit
  it resolves true at once. The worker reports the exit only after saves
  in flight settled (at most 10 s), because ending a worker aborts its
  IndexedDB transactions; `stop()` waits for that (12 s at most). A worker
  crash is a 255 "fatal" exit. `detachDisk(i)` stops disk `i`'s saves,
  waits for the one in flight and closes its store, so the owner may then
  write the store; `exportDisk(i)` returns `CowDisk.exportParts()`.
- Snapshots in worker mode: `WorkerKernel.snapshot()` checks the mounts
  here (the 9p servers run on the owner, which the worker's snapshot
  point cannot save, so it refuses while anything is mounted) and posts
  `snapshot`; the worker
  runs `KernelInstance.snapshot()` and answers `snapshot-result` with the
  memory transferred (no second copy). It posts that before `exit`, which
  waits for it, so a snapshot with `stop` always arrives. Every device
  state is the worker's `Kernel` (`saveDevices()`), including vsock's,
  because the worker's kwvsock holds the CID; the owner's `VsockHost`
  starts fresh on the `vsock-ready` the restore replays. `createKernel({
  worker: true, restore })` sends the snapshot in the boot spec and the
  worker restores as inline (disk and device checks there). The memory is
  copied with `slice()` and transferred (about 120-170 ms for 512 MiB on
  Node 24; a structured clone took about 340 ms, 2026-11-15), or with
  `transferRestore: true` (WorkerOption) the caller's buffer is handed
  over and detached, for a snapshot read for one boot (the CLI's
  `--restore`, the page). A snapshot from either mode resumes in either
  (`test/api.test.ts`: a worker's snapshot resumed inline and in a worker,
  one with a data disk resumed in a worker, saved again there and resumed
  inline).
- Devices in a worker (`js/core/worker-devices.ts` owner side,
  `worker-serve.ts` worker side, both described at their top). The worker
  handles messages only while the kernel is suspended (idle, or its 20 ms
  yields), so everything the owner sends waits for that unless the shared
  transport carries it.
  - Display: frames are "latest frame", sent against credits (at most 2,
    each carrying a buffer to reuse). The worker copies packed XRGB rows;
    the owner converts to RGBA, off the kernel's thread. No OffscreenCanvas
    in the worker: it would make `frame()` platform-dependent and draw on
    the kernel's thread. The `display` size function stays on the owner.
  - `evdev` is `RemoteEvdev`: InputHost's methods forwarded in order to
    the worker's InputHost, which applies the queueing rules. Open state
    and held keys are mirrored, so return values are synchronous (possibly
    a yield stale). Do not merge consecutive moves: glimmer-check expects
    every tablet position.
  - Sound: the owner's queued-frames answer goes back stamped on a clock
    both threads share (`performance.timeOrigin + now()`). kwsnd only
    reports it as delay, so a few ms of error changes no timing.
- Shared transport (`js/core/worker-shared.ts`, described at its top).
  Where a SharedArrayBuffer can be posted (`sharedMemoryAvailable()`:
  Node always, browsers when `crossOriginIsolated`) and `shared` is not
  false, the owner sends a control block in the boot spec and `transport`
  is `"shared"`. Console bytes and evdev calls go through two
  single-producer rings, in order. `SharedServer` is `Kernel.poller`: `wasi.ts` calls
  `poll()` in every monotonic `clock_time_get`, which the kernel reads at
  each slice end just before `wasm_poll_irqs` (kernel.md, IRQs and time),
  so input lands in the same slice. Idle: `poll_oneoff` calls `idle()`
  (sets IDLE, then checks the doorbell; the owner bumps the doorbell, then
  checks IDLE, both seq-cst) and the owner posts one `kick` per idle wait.
  Frames: a SharedArrayBuffer per display size, three XRGB slots as a
  triple buffer (the worker never touches the front slot). A failed
  allocation falls back to credits. Sound, vsock and shm stay messages.
  With more than one CPU, console output comes through a SharedArrayBuffer
  of its own, the console ring (CPU threads, below), sent in the boot spec
  as `console`.
  - Sharing the kernel's own memory is ruled out: the module imports an
    unshared memory (a shared one is a LinkError), the posted compiled
    Module cannot be rewritten, kwemu's JIT modules import the same memory,
    the host's read-modify-write of `wasm_host_irqs` would race the
    kernel's plain stores, and browsers refuse shared views in
    `TextDecoder.decode` and `crypto.getRandomValues`. A seqlock over one
    frame buffer forces retries, since the owner converts while the worker
    copies the next frame.
- vsock in a worker: the protocol (VsockHost) runs on the owner, so
  `WorkerKernel.vsock` is the same object as inline; the worker's kernel
  only passes packets (`createKernelOn` with `vsock: { raw }`), in order.
  Credit bounds what is in flight.
- Packaging: from source, `js/node/worker.ts` starts
  `new URL("./worker-entry.ts", import.meta.url)`. `pnpm build` bundles
  each entry with rolldown, and the `unkernel:inline-worker` plugin
  replaces that expression (it must occur once) with the code as a string
  (`new Worker(code, { eval: true })`), so the dist has no worker file and
  needs nothing from a downstream bundler. In browsers `js/web/worker.ts`
  uses `new Worker(new URL("./web-worker-entry.ts", import.meta.url), {
  type: "module" })`, which Vite bundles when the page builds from source
  (`worker.format: "es"`); for the dist the plugin replaces it with
  `inlineWorkerUrl(code)`, a Blob URL revoked at the first message. Each
  `WORKERS` entry has its own mark (`/*unkernel-worker:node*/`, `:web*/`,
  `:cpu*/`, `:webcpu*/`, `:wasi*/`, `:webwasi*/`).
  Vite's `?worker&inline` is not usable: plain Node cannot load it.
- A worker's WebSocket sends the page's origin, so the Wisp allow list
  works as inline; Node workers send `NODE_WISP_HEADERS`.

## CPU threads (SMP kernels, `js/core/cpu.ts`)

- An SMP kernel (kernel/smp.md) imports `wasm_init_flag`,
  `wasm_cpu_count` and `wasm_cpu_start`. `CreateKernelOptions.cpus` (CLI
  `--cpus N`, which implies `--worker`; `WorkerBootSpec.cpus`) sets how many
  CPUs `Kernel` offers. A UP kernel ignores it with a log line, so does a
  platform without `Platform.spawnCpu` or one whose `Platform.whyNoCpus()`
  gives a reason for the kernel's thread (browsers: below);
  `stats().cpus` says how many it got. The memory is shared (the
  module's import says so).
- `Kernel` is CPU 0, in whatever thread runs it, with every device.
  `wasm_cpu_start(cpu, idle)` calls `KernelConfig.spawnCpu` (from
  `Platform.spawnCpu`: Node `js/node/cpu.ts`, a `worker_threads` Worker
  on `cpu-entry.ts`, unref'd) with a `CpuInit`: the compiled module, the
  shared memory, the idle task, the CPU count, the address of
  `__wasm_init_memory_flag` (the module's export) and `epochMs`, so that
  the CPU's monotonic clock, `performance.timeOrigin +
  performance.now() - epochMs`, is CPU 0's (one clock across Node's
  threads: a worker's value never ran ahead of the main thread's,
  2026-11-16). In worker mode the kernel worker starts the CPU threads
  itself (`ServeKernelOptions.spawnCpu`). Exit terminates them.
- Defaults: the SDK and the CLI boot one CPU on the calling thread
  (snapshots, `--suspend` and the kwtest and abitest runs of testing.md
  expect the UP kernel). For a host that serves other work, `worker:
  true` costs nothing (same speed, the event loop free) and `cpus: 4` is
  where parallel work stops gaining: `node test/profile.ts` (testing.md,
  Profiling CPUs and worker mode; i7-10700K, 2026-11-19: 4 CPUs 2.9-3.3x,
  8 2.9-3.8x, 16 no better, boot 0.39 s on 1 CPU against 0.81 s on 4).
- `runCpu()` checks that the flag reads 2, instantiates the module with
  its own `wasm_switch_to` (its own resumers: JSPI stacks stay in their
  thread), clocks, `random_get`, `sched_yield`, kwemu's JIT imports and
  `fd_write`, and enters `kw_secondary_entry(idle)`. Panic text (fd 2,
  one CPU's) goes to CPU 0's host as a message; `log` lines are host
  messages, not console output. Its
  imports share one `Uint8Array` and one `DataView` made at start (the
  memory never grows): a new `DataView` per `clock_time_get` made
  `clock_gettime` on CPU 1 cost 500 ns against CPU 0's 365
  (2026-11-16; 358 with the shared view). Its
  `proc_exit` posts the exit; any device import is a kernel bug on that
  CPU and ends the session ("fatal", naming it). After an exit, a trap or
  a fatal error it also sets the stop IPI (bit 26) in CPU 0's word
  (`CpuInit.irqWord0`) and notifies it: CPU 0's host takes the message
  only once CPU 0 suspends, and CPU 0 may be spinning or blocked
  (`memory.atomic.wait32`, which the notify ends) waiting for this CPU; the
  kernel takes that bit even with interrupts off and parks CPU 0
  (kernel/smp.md). Its idle blocks in the kernel
  (`memory.atomic.wait32`), so it needs no event loop.
- Console output with more than one CPU (`js/core/console-ring.ts`,
  described at its top): every CPU's console writes, CPU 0's too, go into
  one shared ring of 256 KiB (`CpuInit.console`), each write a record
  whose space it reserves inside `fd_write`, while the kernel still holds
  the lock that orders the writes (hp->lock, the console lock), so the
  ring's order is the kernel's. A reader takes every committed record in
  one chunk.
  - Worker mode with the shared transport (the CLI's `--cpus`, the page):
    the owner reads it (`WorkerKernel`, `ConsoleRingReader`), woken by
    `Atomics.waitAsync` on any commit when it found nothing last time,
    else when an eighth of the ring is full or after 2 ms (a poll of 4 ms
    where a thread has no `Atomics.waitAsync`). It needs nothing from CPU
    0, so a full ring makes a writer wait (`Atomics.wait`, "wait"): the
    owner never waits for the kernel, so that cannot deadlock. At exit
    the owner takes what is left before it closes the output.
  - Inline and `--worker=messages`: CPU 0's host reads it (`Kernel`):
    the same loop while its thread is free, a take in its clock import
    (`Kernel.poller`, at most every 1 ms unless 16 KiB wait; the messages
    transport posts that at once, `KernelConfig.flushOutput`), and its own
    writes go out at once (directly while the ring is empty and no batch
    is on its way, else through the ring, which it then empties). A
    writer may not wait for CPU 0 there: CPU 0 may spin for a lock that
    writer holds. So another CPU that finds the ring full takes the
    records out and posts them as a batch ("post"); every take numbers its
    batch and `ConsoleOrder` (cpu.ts) writes batches by number. At most 16
    may be on their way; then the writer waits up to 50 ms for CPU 0's
    host to take one, and if it took none, drops (logged) without waiting
    until it takes one again.
  - Measured (2026-11-17, Node, 2 CPUs, riscv64 guest):
    `seq 1 200000` on CPU 1 as fast as on CPU 0 in every mode; output of
    a task on CPU 1 reaches the host about 0.1-0.3 ms after it was
    written also while CPU 0 runs user code (before: at CPU 0's next
    yield, 10 ms at the median).
- The JIT per CPU (kwemu.md, JIT on several CPUs): each CPU's thread
  has a `JitHost` of its own (`CpuInit.jit`, `memory64`), attached to
  its instance's `__indirect_function_table` before the kernel runs
  there, so a CPU compiles into its own table and never sees another's
  modules. It mirrors its counters into a `Float64Array` on a
  `SharedArrayBuffer` (`CpuInit.jitMirror`, `JIT_MIRROR_WORDS`) after
  every compile and free, and logs through a `log` message. CPU 0's
  `Kernel.jitStatsPerCpu()` reads them: `stats().jit` is the sum over
  CPUs, `stats().jitCpus` per CPU, `kwemuCpus` the -Dkwemu-stats
  counters per CPU (copied through CPU 0's emit buffer from the emit
  buffer address in the mirror); the CLI's `--stats` prints a line per
  CPU. `JitStats.maxCompileMs` is the longest `new Module` +
  `new Instance` + first call + `table.set`: with kwemu's `flags` bit 0
  the host calls the new `f` once with `cpu` = 0 (it returns at once), so
  that V8's lazy compile of it happens inside the import, in kwemu's
  compile window, where other CPUs do not wait for this one (kwemu.md,
  JIT on several CPUs).
- `JitHost.caps()` offers SIMD (bit 5, `CAP_SIMD`) when
  `WebAssembly.validate` accepts a small module with v128 code
  (`jitSimd()`, exported by the SDK, asked once per host). Every engine with JSPI has SIMD; V8
  on an x86-64 without SSE4.1 refuses it, and kwemu then keeps the i64
  register locals and calls its interpreter for AdvSIMD and crypto
  (a64/jit.md, AdvSIMD and crypto). The test host
  (`test/kwemu-jit-host.mjs`) offers it unless `KWEMU_JIT_SIMD=0`.
- `sched_yield` is a Suspending import here too (the kernel yields on
  every CPU, kernel/smp.md): it returns through `yieldToEventLoop()`.
  This thread otherwise never reaches its event loop (it idles in the
  kernel), and V8 runs FinalizationRegistry callbacks and the second
  pass of its collections (which frees a collected module's code) only
  as tasks there. Without it a secondary reported nothing collected, so
  after 256 frees every compile and free allocated a nudge of up to
  4 GiB (a shootdown once waited 0.58 s for that CPU), and at 2048 it
  would have refused every compile (2026-11-16, compile churn on CPU 1;
  kwemu.md, Numbers).
- A task the kernel moves to another CPU starts there on a fresh stack
  (`isNew`), and its old stack's CPU then calls `wasm_stack_drop(task)`
  (CPU 0's `Kernel` and `runCpu()` both answer it): the host forgets
  that resumer, whose stack never runs again (kernel/smp.md).
- Devices stay with CPU 0's `Kernel`: the kernel calls every device
  import there (kernel/smp.md, Devices), and the host reads and writes
  their memory (block data, pixels, shm pages, PCM) while other CPUs run.
  Every IRQ bit goes through `Kernel.setIrqBits` (`BlockHost.raise()`
  too): a plain read-modify-write of `wasm_host_irqs` would drop an IPI
  another CPU sets meanwhile. Copies already guard the APIs that refuse a
  shared view in browsers (`TextDecoder`, `getRandomValues`, the JIT's
  `WebAssembly.Module`); the device paths hand on copies too (block
  writes gather into a new buffer, PCM chunks and worker frames are
  copied, shm `read()` copies), except `InlineShmFile.views()` and
  `DisplayHost.pixels()`, which are views of the memory by design.
- CPU 0's host sets IRQ bits with `Atomics.or` (`Kernel.setIrqBits`),
  and its idle `poll_oneoff` also waits on `Atomics.waitAsync` of
  `wasm_host_irqs` (`Kernel.watchIrqWord`), which another CPU's
  `memory.atomic.notify` (an IPI) ends; the wait's `done` runs once.
- Both kernels in one bundle: a bundle made with `kernel/tools/bundle.mjs
  --smp <SMP vmlinux32.wasm>` (same tree; `kernel/build.sh` passes it once
  the SMP kernel is built, and bundle.mjs refuses kernels of two different
  builds: kernel.md, Two kernels) carries
  `unkernel.kernel.smp.kwpatch` last, a patch from its UP kernel to the
  SMP one (`kernel/tools/kwdiff.mjs --remap`: the base's function, global
  and type indices and data addresses renumbered to the target's by
  `js/core/kremap.ts`, then a bsdiff-style byte diff; 1.37 MB raw, 0.76
  MB gzip, 0.67 MB brotli, 2026-11-17). With more than one CPU asked
  (`cpus`, CLI `--cpus N`), `loadKernelModule({ smp: true })` (and the
  CLI) reads the bundle's bytes (no streaming compile), applies the patch
  (`js/core/kpatch.ts` `smpKernelFromBundle()`: base length and SHA-256,
  result SHA-256 checked; about 0.2 s and +50 MiB transient in Node) and
  compiles the SMP kernel followed by the bundle's other `unkernel.*`
  sections, so `bundledBoot()` reads it as the bundle. Without the
  section, with one CPU, or with a compiled Module as `kernel`, nothing
  changes. A patch that does not fit the bundle's kernel (another length,
  another SHA-256, or a result whose SHA-256 is wrong) is an error, never a
  wrong kernel: `loadKernelModule()` rejects with a `BootError` at the
  `compile` stage ("Deriving the SMP kernel from the bundle failed: ..."),
  and the CLI says it cannot load the kernel and exits 1.
  `withoutSmpPatch()` gives the bundle without the section (the page's
  split, web.md). `pnpm test:kpatch` (`test/kpatch.test.ts`; with
  `KPATCH_UP` and `KPATCH_SMP` set, also on the real kernels and
  bundle.mjs's checks).
- Browsers (`js/web/cpu.ts`, `web-cpu-entry.ts`; `webPlatform` and the
  web kernel worker pass `spawnWebCpu` and `webCpuBlocker`): each CPU is
  a module Web Worker (`unkernel-cpu<n>`) running the same `runCpu()`,
  started by the kernel's own thread, so in worker mode the CPU workers
  are nested workers of the kernel worker (Chrome, Firefox and Safari
  15.5+ have nested dedicated workers). The init message carries the
  compiled Module and the shared `WebAssembly.Memory` (both
  structured-cloneable; the memory only where the page is cross-origin
  isolated). `webCpuBlocker()` is why a thread cannot run CPUs: no
  SharedArrayBuffer / cross-origin isolation, no `Worker` in that
  thread, no `Atomics.waitAsync` (CPU 0's idle wait on its IRQ word), or
  a thread that may not block: CPU 0 waits for another CPU in
  `memory.atomic.wait32` (step 6), which traps on a browser's main
  thread, so inline mode on a page gives the SMP kernel one CPU and SMP
  needs `worker: true` (or a worker of the caller's own).
  `webCpuBlocker({ worker: true })` is the page's view before it starts
  a kernel worker (no main-thread test). `runCpu()` checks JSPI in its
  thread and reports a missing one as fatal; `web-cpu-entry.ts` turns any
  rejection into a `fatal` message, because a browser worker reports no
  unhandled rejection to its owner. Clocks agree across a worker and its
  nested worker (`performance.timeOrigin + performance.now()`): in
  headless Chromium (2026-11-17) a nested worker that read its parent's
  latest clock value and then its own found its own later by 5 µs at the
  median and at the minimum (the timers' resolution in an isolated page),
  never earlier. From source, Vite bundles the nested worker
  (dev server and build); the dist inlines it as a Blob URL
  (`inlineCpuUrl()`, revoked at the first message). The page (web.md,
  Several CPUs) is the only browser user so far.

## WASI programs (prototype; `js/core/wasi-exec.ts`, `wasi-program.ts`)

- The guest's executable wasm32-wasi files run on the host's engine:
  binfmt_misc hands them to kwwasi (`guest/src/kwwasi.c`, whose top
  describes the protocol; a change goes in kwwasi.c and both files), which
  connects to vsock port 1021, sends argv, environment, the preopen and
  the module's identity (the module itself only when asked), and then does
  the program's file calls in the guest as that process. The program's
  argv[0] is the caller's, except one without a slash (found through
  $PATH): then it is the path exec ran (a symlink's own name), so CPython
  finds its stdlib from $PATH and a multicall program (uutils) still sees
  the name it was called by. `serveWasiPrograms(vm.vsock, opts)` (exported
  by `js/node.ts` from `js/node/wasi.ts`, by `js/web.ts` from
  `js/web/wasi.ts`; core's `serveWasiProgramsWith()` takes the spawner)
  listens there; the CLI's `--wasi` calls it, and the page always
  (web/src/boot.ts).
- The owner (`wasi-exec.ts`) is wherever `vm.vsock`'s VsockHost runs: the
  kernel's thread inline, the owner's thread in worker mode and with
  several CPUs. It parses START, gets the module (below), checks its
  imports and gives each running program a worker of its own, started by
  the owner, not by the kernel worker, since the connection is the owner's
  (one thread boundary per request, not two).
  The worker (`wasi-program.ts`, `serveWasiProgram()`) gets the compiled
  `WebAssembly.Module` in its "start" message, so V8 compiles once and its
  tiered-up code serves every later run.
- Modules (`ModuleCache`): compiled ones are kept by the SHA-256 of their
  bytes (16 and 512 MiB of module files at most by default,
  `moduleCache`, least recently used first; the newest stays even when
  larger), and START's identity of the file (size, dev, ino, mtime and
  ctime, from kwwasi's fstat of the file it opened) maps to one of them
  (256 identities at most). The host answers MODULE 0 (it has it) or 1:
  kwwasi then sends the file in 256 KiB DATA frames, reading it as it goes
  (it never holds it whole), and END with the identity after reading; the
  host maps the identity only if END's is START's. A START whose identity
  is being sent waits for that transfer, so simultaneous starts send it
  once. Identities belong to one machine (one `serveWasiPrograms()`, one
  VsockHost; a restored snapshot is another VsockHost and gets its own
  map). Timestamps are only as fine as the file system's (the initramfs's
  rootfs: a clock tick; three in-place rewrites of the same size in a row
  kept one ctime, 2026-11-18), so a file whose ctime is less than 2 s old
  when kwwasi opens it is UNSETTLED: sent, never mapped (as git treats
  racily clean files; this also covers a rewrite while kwwasi reads it).
  On a 9p mount a host-side edit shows once the attribute cache expires
  (1 s), as for reading the file. `stats.received` counts modules sent,
  `stats.cached` compiled modules kept.
- Calls kwwasi answers are a round trip to the owner, which writes the
  request frame to the connection and hands the REPLY body back. Two
  paths, one body of code (`#call()` returns a Reply or a promise of one):
  - Shared, chosen when `wasiSharedChannelAvailable()` (SharedArrayBuffer
    that can be posted, growable, and `Atomics.waitAsync`: Node; browsers
    only when cross-origin isolated): one growable SharedArrayBuffer per
    program (`CH_*` in wasi-program.ts; 256 KiB, grown up to kwwasi's
    64 MiB frame limit). The import writes the frame there and bumps
    `CH_REQ`, spins up to 200 µs (`SPIN_MS`), then sleeps in `Atomics.wait`;
    the owner wakes on `Atomics.waitAsync` (it never blocks), writes the
    body over the frame and sets `CH_REP`. The imports are plain functions,
    so no JSPI is needed in the worker; `_start` runs to its end inside the
    "start" message's handler. `terminate()` ends a worker that sleeps in
    `Atomics.wait` at once.
  - Messages, the fallback (`sharedMemory: false`): JSPI Suspending
    imports, the frame goes as a "request" message and the body comes back
    as "reply" (transferred). No cross-origin isolation needed.
  Args, environ, clocks, random and preopens are answered in the worker.
  Copies: a write's data is copied once on this side, from the program's
  memory into the channel or the request message (transferred); the owner
  writes request frames to the connection as they are (the connection
  copies what it is given, and a channel frame is overwritten only after
  kwwasi has read it all and replied), and reads a REPLY body straight
  into the channel, or hands the connection's own chunk on when it is
  exactly the body (messages).
- Paths and the cwd (kwwasi.c's top, `resolve()`): a WASI program has no
  cwd. wasi-libc's starts as "/" (no PWD; upstream main and zig 0.16's
  copy alike), a relative path is made absolute against it, the leading
  "/" is stripped and the longest matching preopen name wins, so
  "/etc/x" and "etc/x" both reach path_open as (3, "etc/x"); only in a
  program without chdir() linked does "./x" arrive as "./x" (Rust's std and
  CPython 3.15 strip it too; all three traced 2026-11-18). So kwwasi
  preopens one directory, "/" at descriptor 3, and in a cwd other than "/"
  picks per path: "/x" absolute; ".", "./x", "../x" the cwd's; the first
  component where it exists (a new name in the cwd); in both, where the
  whole path (else its parent) exists in only one; else the cwd. Such a
  collision, and creating an entry whose name exists only in "/" ("mkdir
  lib": "/lib", EEXIST), warn once per run on stderr. `KWWASI_CWD`: `auto`
  (no warning), `root` (undecided collisions to "/"), `off` (descriptor 3
  is "/", every path from "/"). Known wrong cases: `ls /` lists the cwd
  (the program's "/" and "." are one), a relative "tmp/x" in a cwd without
  `tmp` is "/tmp/x" (ENOENT on Linux), and an inherited descriptor 3 is
  replaced (4 and up stay).
- Preview 0 (`wasi_unstable`, older toolchains such as wasienv): a module
  may import it (or both); it gets the preview 1 functions adapted where
  the preview 0 witx (wasmtime `crates/wasi/witx/p0`) differs: fd_seek's
  whence (CUR 0, END 1, SET 2), the 56-byte filestat with a u32 nlink, and
  poll_oneoff's 56-byte subscriptions (a u64 `identifier` before the clock
  id); events, dirents, fdstat and errno are the same, and preview 0 has
  no sock_accept. `test/wasi/p0.c` (no libc) checks those three.
- Ending: "exit" (proc_exit, return, trap 134, failed instantiation 126)
  becomes kwwasi's EXIT; that worker is idle and is kept for the next
  program (`idleWorkers`, default 2; a worker is reused only after its
  program ended by itself). When the connection ends first (kwwasi killed,
  the kernel exited or stopped) the worker is terminated at once, mid
  computation or not. `stats`: `running` programs, live `workers` (Node:
  until its exit event), `idle` ones, `channels` whose owner loop still
  runs (shared path; 0 a task turn after the program ended). `close()`
  stops listening and ends the idle workers.
- Snapshots: programs are not in them and do not hold the snapshot point
  up (no `hostBusy`, no `keep`), so the restored kernel resets kwwasi's
  connection and kwwasi exits 126 ("lost the connection to the host"); it
  watches the connection while it waits on a blocking read, write or poll,
  so a program waiting for its stdin ends too. Reads and writes that cannot
  block skip that watch (`may_block()`: directories, /dev/null, zero, full
  and urandom, regular files on tmpfs, ramfs, ext4, overlayfs, 9p and a few
  other disk file systems; cached per descriptor), as preads and pwrites
  always did. An EXIT that comes while a request is in flight (the
  program's worker failed) is still obeyed when the reply's send then fails.
- kwwasi's cost per request is system calls, each slow under emulation:
  one recv() for a frame that has arrived whole, one send() per frame (its
  header kept in front of the payload), buffers kept between requests and
  no copies of the program's data (kwwasi.c's top).
- `test/wasi-binfmt.ts` (`--messages` forces the fallback; state and
  numbers in status.md).

## vsock (`js/core/vsock.ts`)

- The host side of kwvsock (kernel/devices.md). `VsockDevice` (`Kernel.vsock`) is
  the kernel's packet pipe; `VsockHost` is the host's end of virtio-vsock
  at CID 2, behind `vm.vsock`.
- Credit: each connection advertises `bufferSize` (256 KiB default) and
  counts as consumed only what the reader took from `readable` (pull
  stream, highWaterMark 0). CREDIT_UPDATE goes out when the guest's view
  of free space falls below half of the guest's window, which is the
  smaller of `bufferSize` and the guest's `buf_alloc` (Linux caps sends to
  its own buffer and never asks; measuring against `bufferSize` alone
  stalls transfers with large buffers). Sending keeps exactly to
  `buf_alloc - (tx_cnt - fwd_cnt)` (Linux drops the excess), asks with
  CREDIT_REQUEST once when stalled and then every 500 ms (re-asking on
  every CREDIT_UPDATE ping-pongs and stalls), and splits payloads to the
  kernel's `rx_max` less the header. Data beyond the host's credit, or an
  unknown op, resets the connection (`stats.violations`).
- Closing follows Linux: SHUTDOWN with both flags is answered with RST.
  Once both directions are done the host sends SHUTDOWN with both flags
  and waits up to 8 s (`VSOCK_CLOSE_TIMEOUT`) for the guest's RST, then
  resets. The guest sees a host RST as EOF on reads and EPIPE on writes; a
  refused guest connect is ECONNRESET.
- User callbacks run from a microtask, never inside the kernel's import.
  Packets from the guest are copies.
- Do not feed a throughput test from busybox `head -c`: it copies a byte
  at a time (~22 MiB/s by itself).
- Test: `test/api.test.ts` (inline and worker) with
  `guest/src/kwvsock.c`; `VsockDevice.refuse` is the hook that refuses
  guest sends to exercise the kernel's retry path. Skipped when the kernel
  under test has no `wasm_driver_vsock_init` import;
  `UNKERNEL_TEST_KERNEL=path` runs the file against another bundle.

## Programs on a pty or on pipes (`js/core/tty.ts`)

- The host client of kwtty (`guest/src/kwtty.c`, `/usr/bin/kwtty
  serve`, respawned by inittab), which gives each connection its own pty
  and runs a user's login shell, or a program, on it, or with PIPES runs it
  on stdin, stdout and stderr pipes. On a pty the API follows zigpty /
  node-pty (`spawn(file, args, options)`, `resize(cols, rows)`,
  `kill(signal)`, `pid`, `cols`, `rows`, `exitCode`); keep it that way.
  vsock port 1022 (`TTY_PORT` here, `KWTTY_PORT`
  there; each names the other): a privileged port, so no unprivileged
  guest program can take it. The server accepts only the host (CID 2).
- `Pty.close()` is `conn.close()`, not a half-close: the guest closes the
  master, so the session gets SIGHUP, and it closes its socket at once, so
  `close()` resolves without vsock's 8 s close timeout. A Subprocess's
  `close()` sends SIGHUP to the program's process group.
- Protocol: described at the top of tty.ts and kwtty.c; a change goes in
  both. Unknown frame types and options are skipped unless marked
  critical, and the host marks every option an old guest must not ignore
  critical, so a new feature can come within version 1 with a feature bit
  (as PIPES and CONNECT did).
- The user (USER, OPENED feature bit 2 `TTY_FEATURE_USER`;
  `SpawnOptions.user`, a name or uid): the host always sends USER, empty
  for the guest's default (`user`, uid 1000, on the distro images; root
  on the base shell), and marks it critical only for a user other than
  root, so a guest from before USER (whose sessions are root's) refuses
  only that; a guest that gets no USER (a host from before it) runs root's
  session, as before. The SDK's own programs run as root (`guestRun()`:
  kw9p, umount, `kwcmd sync`, which links into `/run/kw/bin`), so do
  `guest/tools/vm-run.ts`'s build steps. The console is not a kwtty session:
  its user is `kw.user=` on the command line (the CLI's `--user`).
- Connections into the guest (CONNECT, OPENED feature bit 1
  `TTY_FEATURE_CONNECT`): `vm.connect(port, { timeout, signal, ttyPort })`
  on both handles (waits for createKernel's `mounts`, as spawn() does;
  ENODEV with `vsock: false`), or `connectGuest(vsock, port, opts)`. kwtty
  connects to TCP 127.0.0.1:port (IPv4 only: the kernel has no IPv6)
  before it sends OPENED, so the promise resolves connected. A refusal is
  `PtyError` ECONNREFUSED (the guest's ERROR message starts with the errno
  name), another connect error EREMOTE, and a guest from before CONNECT
  refuses the critical option: EREMOTE. After OPENED no frames: a
  `GuestSocket` (`port`, `readable`, `writable`, `close()`, `closed`,
  `[Symbol.asyncDispose]`), which is a `ByteDuplex` for `fetchOver()`.
  Bytes that came behind OPENED go out first (`TtyFrameParser.rest()`);
  `readable` is a pull stream of its own, also because the connection's
  has been read and `new Response()` refuses a disturbed stream. `writable`
  is the connection's: `close()` is a half-close, `abort()` a reset.
  `signal` only aborts connecting.
- The guest's relay (`relay()` in kwtty.c, apart from `session_loop()`,
  which is built around a program): one forked process per connection,
  64 KiB in flight per direction, EOF passed on as `shutdown(SHUT_WR)`
  once the bytes before it are written, both ways. It ends when both
  directions are done, or at the first POLLERR or failed send or recv
  (then it closes both sockets). POLLHUP only means both directions of
  that socket are shut, with data possibly still unread, so it ends
  nothing. Linux vsock shows no event when the host closes or resets after
  it shut down its sending side, so while nothing else is asked of the
  vsock socket the relay sends an empty message every second, which fails
  with EPIPE once the host no longer reads.
- CLI `--forward HOSTPORT:GUESTPORT` (repeatable): listens on the host's
  127.0.0.1 before booting (a taken port exits 1), each connection is
  `vm.connect()` (node:net with `allowHalfOpen`, so half-closes pass
  through), and a refused guest port resets the host connection.
- Credit: `output` is a pull stream (highWaterMark 0) over the
  connection's `readable`, so vsock credit follows the reader. If nobody
  reads `output`, the guest program blocks once the vsock window
  (`bufferSize`, 256 KiB by default) and the pty buffer are full, and
  `ended` does not settle: it settles when the reader asks past the last
  output (EXIT comes behind it). Cancelling `output` discards the rest and
  lets `ended` settle. Every frame goes out through one writer in call
  order, so RESIZE and DATA never interleave or reorder. A Subprocess's
  `stdout` and `stderr` share the connection: it is read while either has
  a read waiting, and the other's frames queue on the host until read
  (no limit), so read both or cancel one. The pump stops in the same turn
  it decides to, so a pull right after starts a new one (a stream whose
  pull resolves without data is not pulled again).
- On pipes the guest takes turns reading stdout and stderr, and ends the
  session as on a pty: EXIT once both pipes hit EOF after the exit, or
  after 50 ms without output, or after 1 MiB.
- Hangup: Linux signals only the session leader (the login shell); when
  it exits, the foreground process group gets SIGHUP. A HUP trap in busybox
  ash's own interactive shell does not run once its tty is hung up
  (2026-11-14), so the test sets the trap in a foreground `sh -c`.
- Test: `pnpm test:tty` (`test/tty.test.ts`, no kernel: the protocol
  against a fake `Vsock` and a JS stand-in for the server, CONNECT's
  handshake and raw bytes included) and `test/api.test.ts` (inline and
  worker; for CONNECT a busybox `nc -lk -e` server in the guest: echo with
  half-close, 1 MiB each way, ECONNREFUSED, the relay's exit after an
  idle close, `vm.fetch()` from a shell HTTP server). Skipped when the
  kernel has no kwvsock.

## Host file systems (`js/core/fs.ts`, `9p.ts`, `memfs.ts`, `mounts.ts`, `js/node/nodefs.ts`)

- One 9P2000.L server per VM on vsock port 564, started at the first
  mount. Each mount is its own export with an attach name the server makes
  up (`fs1`, `fs2`, ...) and exactly one guest connection: the first
  connection that attaches to an export owns it for the export's life,
  and a Tattach naming it from any other connection fails with EBUSY,
  also after the owner ended (the mount is then gone anyway). The owner's
  own later Tattaches (the kernel attaches again for each uid it acts as)
  work. An unknown name fails Tattach with ENOENT, and so does a later
  Tattach on a bound connection that names another export. One backend
  mounted at two paths is two exports with a connection each. The guest's `kw9p NAME DIR [ro]`
  (`guest/src/kw9p.c`) makes DIR (`mkdir -p`), refuses one that is
  already a mount point (EBUSY), connects to CID 2 and mounts 9p with
  `trans=fd` on the socket, source `kw9p:NAME`, and fixed options
  `cache=readahead,metatimeout=1000,noxattr,access=client,msize=131072`.
  It prints errors as `kw9p: what: text (errno N)` and the SDK maps N to a
  `MountError`. kwtty and kw9p are in the base's kw layer, so every
  distro and the base shell have them; a root without the base (a
  `--disk` root, `initrd: null`) does not.
- The server and the handlers run on the owner, as vsock does. A mount
  runs `kw9p` through `execSubprocess()`. `spawn()` and `exec()` wait
  until createKernel's mounts are done, so a first `exec()` sees them. An
  object is a `{ fs, ... }` spec only when it has `fs` (an object) and
  nothing but option names, so a tree may contain a file or directory
  called `fs`.
- The public types (`Fs`, `MemoryFs`, `FsError`, `MountError`, ...) carry
  no protocol names and their doc comments no protocol terms; the server,
  framing, the errno table and helpers are internal, and nothing with a
  `P9` prefix is exported.
- Lifetime: a guest `reboot` ends the kernel (exit 2) and its mounts with
  it; booting again with the same `mounts` mounts them again. A mount the
  guest unmounts by itself (its connection ends) becomes `mounted: false`
  and frees its path. A backend's `close()` runs once its last mount is
  gone (the same `fs` may be mounted at several paths), at the latest when
  the kernel exits.
- Connection (`Connection` in 9p.ts): framing with at most one copy
  (`P9FrameAssembler`), Tversion answered there (msize the smaller of the
  client's and 128 KiB), the first Tattach binds the export. At most 16
  requests run per connection, each holding its slot until its reply is
  sent (replies alive <= 16 x msize); 16 more wait parsed, then the
  connection stops reading, so vsock credit stops the guest. Tflush always
  goes through: a queued request is dropped, a running one answered Rflush
  at once (slot freed, the late reply dropped; a flushed walk or attach
  makes no fid, a flushed open closes its file). A read-only export gets
  EROFS for every change before the backend sees it (Tremove still drops
  the fid).
- Session (`Session`): path-based handlers, sync or async. Required
  `getattr`, `readdir`, `open(path, flags)` returning an `FsFile` (`read`,
  optional `write`, `getattr`, `setattr`, `fsync`, `close`), used for that
  fid's I/O until it is clunked, so it must keep working after the path is
  renamed or removed (MemoryFs holds the node, nodeFs the FileHandle).
  Optional `create` (returns the file open), `mkdir`, `symlink`,
  `readlink`, `remove`, `rename`, `setattr` (by path: chmod, utimes,
  truncate by name), `statfs`, `close`. An FsError or an error with a Node
  errno `code` answers that errno, anything else EIO plus one log line.
  Missing methods: EROFS for changes, EOPNOTSUPP for `symlink`, mknod and
  xattrs (kw9p mounts with `noxattr`); Tlink is always EPERM (no hard
  links); a file without `write` is not opened for writing (EACCES);
  Tfsync without `fsync` (a directory) succeeds. Tlcreate without O_EXCL
  of a name that exists (the host made it after the guest's lookup) opens
  it, honouring O_TRUNC. Writes, truncation, fsync and clunk through one
  fid run in order; everything else concurrently. A rename moves the fids
  below it. Names are made only below directory fids,
  and walks never pass a non-directory, so no path goes through a symlink.
- Hung handlers: each request has a deadline (`requestTimeout`, 30 s by
  default, 0 for none). When it passes, the guest gets ETIMEDOUT and the
  log a line; whatever the handler returns later is dropped (a file it
  opens is closed), and the fid's ordered chain moves on. A guest process
  blocked on a request dies at once when killed (the client sends Tflush);
  one that waits gets the error after up to about three deadlines, because
  netfs retries a failed read-ahead with two synchronous reads (2026-11-15,
  `requestTimeout: 1000`: about 3 s). Poweroff completes with such a
  reader left.
- Names: names and link targets from the guest are decoded as strict
  UTF-8, and one that is not is answered EILSEQ (lookup, create, rename,
  unlink, symlink), never decoded lossily, so two byte names cannot alias
  one string. nodeFs leaves host names that are not UTF-8 out of listings
  and answers EILSEQ for such a link target.
- Inode numbers: qid.path is the backend's (dev, ino) (dev << 40 xor ino),
  else a 63-bit hash of the path with the top bit set, which a rename
  changes. Readdir entries carry the same numbers (nodeFs lstats each
  entry, so `ls -i` matches `stat`). qid.version is the backend's
  `FsAttr.version`, else from mtime and size; a revalidation that sees it (or
  the size or mtime) change drops the file's cached pages. 0 makes the
  guest open the file P9L_DIRECT (no page cache; `fs/9p/fid.h`), which
  MemoryFs uses for computed files, whose size can differ from the
  guest's cached i_size (a cached read stops there).
- MemoryFs: the Fs methods are the guest's side and report changes to
  `onChange()` listeners; the host API reports nothing and bumps
  `version`. A function in the tree is a computed file,
  evaluated once per open: `open()` calls it and the open file serves that
  one result for its life (reads slice it, its getattr reports its size),
  so a `cat` of a file whose content or length changes from call to call
  gets one whole value, and the next open a fresh one. A getattr by path
  evaluates it for the size. Computed files are read-only (EACCES) and
  have version 0, so the guest reads them uncached: its cached size can be
  up to 1 s old (metatimeout).
- nodeFs(dir) (Node only): `node:fs/promises`; lstat, never follows
  symlinks (they are listed and read as links; the guest resolves them in
  its own tree), opens with O_NOFOLLOW and O_NONBLOCK and only regular
  files, one FileHandle per open fid, owners shown as uid and gid 1000
  (`FS_DEFAULT_OWNER`, the server's answer for a backend that reports
  none, MemoryFs's too until chowned: the distro images' user may then
  write; root may anyway) and chown
  ignored, setattr on a symlink changes only its times. CLI: `--mount
  HOSTDIR:GUESTPATH[:ro]`, repeatable.
- Cache mode (the only one): `cache=readahead` (CACHE_FILE) with
  `metatimeout=1000` (the `9p: add metatimeout` patch, kernel/devices.md), which adds CACHE_META with
  a 1 s limit. A name or attribute the guest has looked up is reused for
  up to 1 s without asking the host; the next lookup or stat after that
  revalidates it with one getattr, which drops a name that is gone or now
  holds another file and the pages of a file that changed. Negative
  dentries expire after 1 s too, and listings always come from the host.
  So a host change (rewrite, shrink, grow, create, delete, rename) shows
  within 1 s, also for a file a guest process holds open (tested). Within
  the window the guest works from what it saw: `stat` shows the old
  attributes, a read stops at the old size, a name the host created or
  deleted is still missing or present (opening a deleted one fails with
  ENOENT from the host). Writes go unbuffered (P9L_NOWRITECACHE): each
  write(2) reaches the handler before it returns, so the guest sees its
  own changes at once, `sync()` has nothing to wait for and `stop()`
  loses nothing already written. Exec from a mount works (read-only
  mmap).
- Known limits of this mode:
  - Shared writable mmap fails with EINVAL (`v9fs_file_mmap_prepare`
    allows it only with CACHE_WRITEBACK), so sqlite in WAL mode and LMDB
    do not work on a mount.
  - Files the guest creates get the backend's gid, not the guest's.
- Benchmark: `node test/9p-bench.ts [--worker] [--no-jit] [--node]
  [--quick]` (MemoryFs or nodeFs, tmpfs as baseline, 16 MiB files, a
  workload of `find | wc`, `tar | wc` and `ls -lR` over 30 x 100 small
  files). 2026-11-15, x86 host, Node 24, JIT on, MemoryFs unless noted;
  RT = 9p round trips:

  | run | stat µs (RT) | open+close µs (RT) | cat µs (RT) | ls -l 1000, cold/warm ms | read bs=128k / 4k MB/s | write 1M MB/s | create µs | workload ms (RT) |
  | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
  | inline | 7 (0) | 120 (3) | 132 (3) | 190 / 102 | 739 / 1025 | 518 | 357 | 1791 (21.7k) |
  | worker | 13 (0) | 285 (3) | 287 (3) | 275 / 105 | 389 / 961 | 455 | 659 | 3185 (22.6k) |
  | inline, nodeFs | 2 (0) | 256 (3) | 262 (3) | 263 / 107 | 289 / 801 | 388 | 709 | 2890 (22.6k) |

  The same workload takes about 720 ms on tmpfs. One round trip is about
  50 µs inline and 100 µs in a worker; run to run the throughput columns
  vary by about 20 %. The 4k read is a cache hit. Plain `cache=readahead`
  without metatimeout took about twice the round trips and time.
- Rejected: `cache=mmap` with the `sync` mount flag (shared writable mmap
  through CACHE_WRITEBACK) cost as many round trips as plain readahead,
  wrote no faster, created files more slowly, and loses mapped writes on
  `stop()`.
- msize: 128 KiB. 64 KiB to 1 MiB were measured (2026-11-15): no
  consistent gain above 128 KiB inline or in a worker, and each request in
  flight costs the guest two msize buffers.
- Tests: `pnpm test:9p` (`test/9p.test.ts`, no kernel: the server against
  a fake Vsock and a 9P2000.L client written in the test) and
  `test/api.test.ts` (inline and worker; skipped when the kernel has no
  kwvsock or no 9p, found by the "9P2000.L" string in the module). Tests
  that need the window to pass wait 1.2 s; a guest program that must hold
  a file meanwhile waits on its stdin (`hold()`), not on a sleep.

## shm (`js/core/shm.ts`)

- The host side of kwshm (kernel/devices.md): guest shmem files read and written
  in place. `ShmDevice` (`Kernel.shm`) is the kernel's end (each live
  export's page list merged into runs of consecutive pages); `ShmClient`
  is the API (`vm.shm`) on a backend that runs `ShmCall`s: the ShmDevice
  itself inline (`inlineShm()`), messages in worker mode.
- Inline, `read()`/`write()` copy at once, and `InlineShmFile.views()`
  returns one `Uint8Array` per page run over kernel memory, no copy, valid
  only until the kernel runs again: use them synchronously. In worker mode
  each call is a round trip, taken when the kernel next suspends (up to
  20 ms while the guest is busy); `views()` throws EOPNOTSUPP.
- Lifetime: the kernel unpins an export's pages only after its detach
  import returned, and a refresh only after the new list's attach, so
  ShmDevice never copies from a freed page. At exit everything is dropped
  and handles end with "exit".
- Test: `test/api.test.ts` (inline and worker) with `guest/src/kwshm.c`.
  Skipped when the kernel has no kwshm.

## Disks (`js/core/block.ts`)

- `BlockHost` serves kwblk. Backends: `FileDisk` (`js/node/disk.ts`,
  `readSync`/`writeSync` straight into kernel memory, flush =
  `fdatasync`, no discard), `HttpBase` (read-only HTTP Range base in
  256 KiB chunks, kept in memory; `open()` fetches the first chunk, then
  the `prefetch` below), `CowDisk` (a 4 KiB copy-on-write overlay
  over a base, in memory or in IndexedDB through `IdbStore` in
  `js/web/disk.ts`), `ZeroBase`, `SparseBase` and `BaseDisk` (a base read
  directly, read-only). `HttpBase` with `strict` refuses a range whose
  total size or ETag (or Last-Modified) differs from what `open()` saw,
  so a changed image fails reads instead of mixing two images.
- `HttpBase` `prefetch` (default: the image's distro header, distro.ts;
  a number overrides it, `false` turns it off; the `http` spec's and
  `openDistroDisk()`'s `prefetch`): `open()` starts one streaming request
  for `[0, prefetch)`, a plain GET without Range when it is the whole
  image. Each chunk is kept as its bytes arrive; a read inside the window
  waits for them, a read after it is a lazy range. The response is
  checked for size and validator, strict or not; a failure leaves the
  missing chunks to lazy ranges. `stats` (`HttpBaseStats`, also on
  `BaseDisk` and `DistroDisk`): requests, bytes, lazy requests, the
  prefetch's size, bytes and state.
- The distro image (`js/core/distro.ts`; `createKernel({ distro })`, the
  CLI's `--distro`): an erofs or ext4 file system labelled kwroot that the
  base initramfs of the bundle (`unkernel.wasm`) mounts under the root
  overlay (contract: guest/README.md, kernel/guest.md). It is attached as
  /dev/wda, before `disks`, whose indices (DiskState, `onDiskError`,
  `detachDisk()`) stay those of `disks`. A path or URL becomes a spec
  through `Platform.distroSpec`: Node a read-only `file` (or `http`),
  browsers a read-only `http` spec, read in 256 KiB chunks with no
  IndexedDB copy, after the prefetch its distro header asks for (written by
  guest/lib/image.sh: Alpine's whole image; Debian's first 3 MiB, its
  metadata and the data a boot reads, which the build lays out in front;
  a boot from a URL then makes one lazy request, for the device's last
  block, which is probed at boot; 2026-11-19). Without a header (lazy reads only), Alpine's erofs needed 8
  requests, 1.6 MiB, to a working `exec()`, Debian's 42 requests,
  13.5 MiB (local server, 2026-11-19). `DistroInfo` carries the header's
  distro, image id and prefetch.
  `DistroDisk` wraps any of them: writes fail, and `open()` reads the
  first 64 KiB for the file system, label, UUID and a hash (cyrb53, since
  `crypto.subtle` needs a secure context), the identity a snapshot
  records. `openDistroDisk()` (node.ts, web.ts) opens one ahead, for
  `snapshotDiskMismatch()`. Without an initramfs and `cmdline`,
  `createKernel` refuses a distro. `test/distro.test.ts`
  (`pnpm test:distro`) boots each built image.
- Distro images by name (`js/node/distro-fetch.ts`, `fetchDistro(name,
  opts)` in node.ts, the CLI's `--distro NAME`): in a source checkout
  (the package root has `kernel/build.sh`) `vendor/build/distro/NAME.erofs`
  (or `.ext4`), never a download; otherwise `distros.json` (guest/README.md)
  and `NAME.erofs` from the GitHub release `v<package.json version>`
  (`UNKERNEL_RELEASE_URL`: another release directory), into
  `$UNKERNEL_CACHE_DIR`, else `$XDG_CACHE_HOME/unkernel`, else
  `~/.cache/unkernel`, else `<tmpdir>/unkernel-cache`, under `v<version>/`
  (`url-<hash>/` for an overridden URL). The download goes to a part file,
  hashed on the way, and is renamed once size and SHA-256 match the index;
  a cached file is used after the same check, and the index from the cache
  when the network fails. Errors are `DistroFetchError` (`offline` when the
  network failed and nothing usable was cached). The package root and
  version come from the entry's `new URL("..", import.meta.url)`. The web
  entry has no downloader: GitHub release assets send no CORS headers, so a
  page hosts its images and passes their URLs.
- `CowDisk` with a store loads lazily: memory holds only the blocks written
  since their last committed save. Writes to one block apply in submission
  order, so a stale fetch never overwrites a newer write.
- Store reads read ahead (`#loadAhead`): each store read is one IndexedDB
  transaction, an IPC round trip in WebKit, and ext4 reads its orphan file
  block by block at mount. A fetched-ahead block must equal the store's
  record: every write drops it (`#dropAhead`), including one that lands
  while the fetch is in flight (`aheadPending`).
- The persistence rules for the data disk are in data-disk.md.
- kwsparse (`js/core/kwsparse.ts`, CLI `kernel/tools/kwsparse.mjs`):
  magic `KWSPARS1`, u64 LE size, u32 LE block size, u32 LE count, the
  sorted u64 LE block indices, then the blocks. Usually gzipped.

## HTTP into the guest (`js/core/http.ts`)

- `fetchOver(connect, input, init)` is an HTTP/1.1 client over any
  `GuestConnect` (`(port, { signal }) => Promise<{ readable, writable }>`);
  `vm.fetch()` is this over `vm.connect()`. It knows nothing about vsock or
  kwtty, so its tests (`test/http.test.ts`, `pnpm test:http`) need no
  kernel.
- http: only. The connection always goes to the guest's 127.0.0.1 at the
  URL's port (default 80); the hostname goes only into `Host` (`url.host`,
  so `localhost:8000`; Vite checks it). A caller's `Host` header wins where
  the platform lets one through.
- One connection per request, `Connection: close`, no redirects followed,
  `Accept-Encoding: identity` unless the caller set one (browsers drop a
  caller's, so it is always identity there). Hop-by-hop request headers
  are dropped and set by the client.
- Request bodies: an `init.body` of known length (string, buffer, view,
  Blob, URLSearchParams, FormData) and a Request's own body are read in
  full and sent with Content-Length; any other `init.body` (a
  ReadableStream, an async iterable) is sent chunked (`duplex: "half"` is
  added). The upload runs while the response is read, so an early response
  (413) is not stuck behind it.
- Never half-close before the response is in: a guest server, or kwtty's
  splice, may take EOF as the end of the connection. After a complete
  response the client closes the writable and cancels the readable; on an
  error, abort or body cancel it aborts the writable (a vsock reset).
- Response: 1xx are skipped (101 rejects), the head (all heads together)
  is capped at `HTTP_MAX_HEADER_BYTES` (64 KiB). No body for HEAD, 204,
  205 and 304; else chunked (trailers dropped), Content-Length, or until
  EOF. Connection, Keep-Alive, Transfer-Encoding, Trailer, Upgrade and
  headers named in Connection are dropped, Content-Length is kept only
  when it framed the body. A malformed head rejects a TypeError
  (`fetchOver: malformed response: ...`), a truncated or malformed body
  errors the body stream.
- The body stream has highWaterMark 0 and reads the connection only on a
  pull, so vsock credit follows the consumer. Keep it that way.
- Browsers drop `Set-Cookie` from a constructed Response (the "response"
  header guard), and `res.url` is empty; Node keeps `Set-Cookie`
  (`headers.getSetCookie()`).

## Network (`js/core/net.ts`, `net-wisp.ts`, `js/node/net.ts`)

- `NetStack` is a user-mode stack that terminates the guest's frames:
  guest 10.0.2.15/24, gateway 10.0.2.2, DNS 10.0.2.3. It handles ARP,
  DHCP, DNS A (AAAA gets an empty answer), ping to the gateway and DNS
  addresses, and TCP (MSS, window scaling, flow control, go-back-N
  retransmit, FIN, RST). Each TCP connection goes to a backend (any object
  with `connect` and `lookup`). `net` may also be a spec (Worker mode).
- `connect(addr, events)` gets `addr.source`, the guest end as the stack
  sees it: 10.0.2.15 and the guest's source port. A guest that NATs (a
  container namespace behind the machine's masquerade) shows the
  translated address and port, so a backend cannot tell containers apart
  by it. `end()` is the guest's FIN. `wispBackend()`'s `end()` is a no-op:
  Wisp v1 has no half-close.
- A backend object in worker mode (`js/core/net-port.ts`):
  `createKernelInWorker()` makes a `MessageChannel`, serves the backend on
  one end (`serveNetBackend()`) and gives the worker's NetStack the other
  as `{ kind: "net", backend: { kind: "port", port } }`
  (`portBackend()`); the same spec, with a port of the embedder's, is how
  a backend is served from elsewhere. The messages are in the header of
  net-port.ts. Guest to backend: the worker sends each write at once and
  counts it in `buffered` until the owner reports that the backend no
  longer holds it (`written - buffered` after each write and on
  `onDrain`), which shuts the stack's window as for a local backend.
  Backend to guest: at most `NET_PORT_WINDOW` (256 KiB) unacknowledged per
  connection; past it the owner calls `pause()` and queues what the
  backend sends anyway, and the worker acknowledges what it gave the stack
  unless the stack paused the connection (`pause()` and `resume()` there
  hold back the acknowledgements; they are not messages). Chunks are
  transferred; one that is not all of its ArrayBuffer (a pooled Buffer) is
  copied first (`new Uint8Array(u8)`: a Buffer's `slice()` shares memory).
  The backend, its connections and the port end with the kernel
  (`vm.exited`), and a worker that is terminated closes its port, which
  has the same effect. `serveNetBackend()` closes the backend
  (`close()`), so an embedder that reuses one backend across kernels
  passes a wrapper without it.
- `directBackend()` (Node only, the CLI's default) uses host sockets and
  `dns.lookup`. With `hostLoopback`, 10.0.2.2 goes to the host's 127.0.0.1.
- `wispBackend()` multiplexes connections over one WebSocket to a Wisp v1
  server. `DEFAULT_WISP_SERVERS` is the project's
  `wss://wisp.pi0.io:6680/`. With several URLs, all are opened at once and
  the first handshake wins.
- The project's Wisp server has an allow list of Origins
  (`http://localhost` is on it) and answers anything else, including no
  Origin, with 404. Node's WebSocket sends no Origin, so the Node entry
  and the CLI's `--net-wisp` send `Origin: http://localhost`
  (`NODE_WISP_HEADERS`, overridable with `headers`; browsers ignore it).
- Wisp has no DNS, so the guest gets stable fake addresses from
  198.18.0.0/15 and a bad name fails at connect time. It also has no
  connect acknowledgement (a refused connect becomes a reset), no
  half-close, and flow control in one direction only.
- Not implemented: IPv6, IP fragments, UDP beyond DHCP and DNS, ICMP beyond
  10.0.2.2/3, SACK, timestamps and inbound connections. A raw-Ethernet
  relay backend is not useful: v86's relay gave DHCP but no upstream DNS
  or TCP.
