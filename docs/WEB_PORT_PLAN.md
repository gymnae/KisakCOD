# KisakCOD → Web Browser Port Plan

Goal: run KisakCOD (Call of Duty 4: Modern Warfare) in a modern desktop browser,
with game assets **streamed on demand** instead of downloaded up front.

Status: planning document. Nothing in this plan is implemented yet.

---

## 1. What mw2web actually does (and what we should copy)

[mw2web](https://github.com/bettertech-97/mw2web) is **not** a compiled port of the
original IW4 engine. It is a fork of IW4L, a from-scratch **Rust + Bevy** re-implementation
that targets `wasm32-unknown-unknown` and renders with **WebGPU** (a vendored `wgpu`).
It uses KisakCOD, OpenAssetTools and IW4x as *references* for formats and behavior.

We have something mw2web did not: a complete, buildable C++ engine. So we should **not**
copy its architecture (a rewrite). We should **compile KisakCOD itself to WebAssembly with
Emscripten**, and copy mw2web's *runtime* ideas, which are engine-agnostic and proven:

| mw2web technique | Measured effect (from its `PERFORMANCE.md`) | Adopt? |
|---|---|---|
| Engine runs in a Web Worker, not the main thread | Loads no longer freeze the tab | Yes |
| Game files streamed in **16 KiB blocks** into **OPFS** (Origin Private File System) | Small texture reads stop pulling whole files | Yes |
| Per-map **access manifests** → server builds one "pack" per map; client fetches **8 MiB ranges, 6 in parallel** | Cold download ~28 s → ~10 s; the bottleneck was ~1,100 tiny requests, not bandwidth | Yes |
| No read-ahead on large zip packs | Avoided pulling 184 MB to read 45 MB | Yes |
| Sounds pre-warmed into memory before first play | Removed 100–170 ms stalls | Yes |
| Audio on its own thread → AudioWorklet ring buffer | Zero underruns, even while loading | Yes |
| Threaded build for Chrome/Edge (SharedArrayBuffer), single-thread fallback for Firefox | Firefox build works, but slower | Yes |
| Shader pipeline compile cap `max(2, cores/2)` | Lower CPU and GPU-process peaks while loading | Yes, if/when on WebGPU |
| gzip'd, symbol-stripped `.wasm` streamed with `instantiateStreaming` | 114 MB → 16 MB transfer | Yes |
| Users supply their own game install; the repo ships no assets | Legal requirement | **Mandatory** |

mw2web's memory notes matter too: an MW2 map uses 2.3–3.3 GB of a 4 GB wasm32 address space.
CoD4 assets are smaller and the original game targeted 32-bit Windows, so we have more headroom.

### 1b. Prior art: mw.slqnt.dev, an existing KisakCOD browser build

[mw.slqnt.dev](https://mw.slqnt.dev/) (by slqnt, who also made the Half-Life 2 browser port) already
runs a **KisakCOD singleplayer** build in the browser. Facts below come from inspecting its shipped
`kisakcod-sp.js` (246 KB) and `kisakcod-sp.wasm` (6.0 MB). No source link or license notice
appears in either file. KisakCOD is GPLv3, so the author owes the corresponding source to anyone who
receives the binary. **Ask for it before building any of this from scratch.**

| Area | What mw.slqnt.dev does | Effect on this plan |
|---|---|---|
| Toolchain | Emscripten, pthreads (shared memory, 1 GB initial / 4 GB max), `PROXY_TO_PTHREAD` (`_emscripten_proxy_main`), `simd128`, wasm exceptions via `invoke_*`/setjmp | Confirms §3 |
| Renderer | D3D9 → WebGL2 shim named `d3d9_gles`. Shaders are translated by **Valve's ToGL** (`thirdparty/togles/linuxwin/dx9asmtogl2.cpp`, "D3DToGL"), adapted to GLSL ES 3.00. Uses `WEBGL_compressed_texture_s3tc`, `EXT_clip_control`, OffscreenCanvas | **Use ToGL instead of MojoShader**: it is proven on CoD4's shaders, MIT-licensed, and built for D3D9 SM2/3 |
| Assets | **No server streaming.** An installer page uses `showDirectoryPicker` (fallback `<input webkitdirectory>`) on the user's CoD4 folder and copies files into **OPFS**. The engine reads them through **WasmFS's OPFS backend** (`createSyncAccessHandle`). Optional categories: per-map zones, audio, video. Required zones: `code_*`, `common*`, `ui*`, `localized_*` | The import path works. Streaming from a host (§4 Phase 2) is the part **not yet done**, and it is what this plan adds |
| Mods | Only mods are fetched from the server: `assets/mods/manifest.json`, then the files, 6 at a time | Reuse that pattern for asset packs |
| Cinematics | Bink decoded with **FFmpeg** `libavcodec`/`libavformat` (`binkvideo`, `binkaudio_rdft/dct`), YUV→RGB in a GLSL shader | Replaces the "transcode to WebM" idea in Phase 6 |
| Audio | Emscripten OpenAL (`al*` imports), so the `KISAK_OPENAL` path | Confirms Phase 4 |
| Network | A WebSocket relay at `/lan?v=1` with join codes (`kisak_lan_init/send/recv`), a 13-byte frame header carrying a 6-byte address, and a 28-byte `browser_ticket` | A working Phase 5 transport. WebRTC remains an upgrade for lower latency |
| Input | Pointer lock, Gamepad API, touch events | Confirms Phase 4 |
| Low-spec mode | Detects software GL (SwiftShader/llvmpipe) and forces `r_customMode 512x360`, `r_specular 0`, `r_normal 0`, `r_drawDecals 0` | Copy it |
| Known gaps | Depth of field disabled on WebGL (`r_postFxDof`) | — |

**Revised strategy:** best case, get slqnt's source (GPL) and contribute streaming and MP on top of it.
Otherwise, follow the plan below with ToGL, FFmpeg-Bink and the WebSocket relay pattern, which this
build proves work on CoD4.

---

## 2. Why Emscripten fits KisakCOD well

Findings from a pass over `src/` (562 `.cpp` files, about 26 MB):

* **wasm32 has 32-bit pointers.** CoD4 fastfiles (`.ff`) are pointer-fixup images of
  32-bit structs, and the code has many `static_assert(sizeof(...) == 0x...)` checks (444 of them).
  An x64 port has to fight this ("x64 gremlins", commit `8aadf94`). wasm32 matches the original layout.
* **There is already an OpenAL sound backend** (`KISAK_OPENAL`, `src/sound/snd_driver_openal.cpp`).
  Emscripten ships an OpenAL implementation built on Web Audio. `deps/dr_libs` already has
  `dr_mp3`/`dr_wav` for decoding.
* **CMake already has platform overrides** (`scripts/platform_override.cmake`,
  `scripts/platform/{win32,linux}`). We add `scripts/platform/web`.
* **The loaders are centralized.** All fastfile I/O goes through `DB_ReadXFileStage()`
  (`src/database/db_file_load.cpp`, 256 KiB `ReadFileEx` reads on the database thread). All
  loose and IWD file I/O goes through `src/qcommon/files.cpp`. Those are the two places
  where streaming plugs in.
* ODE, zlib and speex are portable C. They compile as-is.

The hard parts, roughly in order of effort:

| Area | Current state | Web work |
|---|---|---|
| Renderer | Direct3D 9, about 620 `dx.device->…` call sites in `src/gfx_d3d` | Translation layer to WebGL2, with WebGPU later (§4.3) |
| Shaders | Precompiled **D3D9 SM2/SM3 bytecode** stored inside fastfiles (`Load_CreateMaterialPixelShader`, `src/gfx_d3d/r_material.cpp:286`) | Translate the bytecode at load time (MojoShader) |
| Threads | Win32 `CreateThread` + events (`src/qcommon/threads.cpp`): render backend, worker, database, server, cinematic threads | pthreads plus a portable event primitive |
| Win32 / MSVC-isms | 15 files include `windows.h`; `VirtualAlloc` in 21 files; x86 inline asm in about 10 (e.g. `fld/fistp` in `qcommon.h`, `__asm` memset in `common.cpp`); SEH `__try` in 2; SSE intrinsics in 8 | Portable replacements; SSE → `-msimd128` |
| Networking | Winsock UDP (`src/win32/win_net.cpp`) | Browsers cannot send raw UDP. Use WebRTC DataChannel or WebSocket plus a server-side gateway (§4.6) |
| Cinematics | Bink (`binkw32.dll`, closed source) | Stub, or transcode offline to WebM and play through `<video>` (§4.7) |
| Steam / Miles / DirectInput | Windows DLLs | Stub Steam; Miles stays off (OpenAL path); browser input events |

---

## 3. Target architecture

```
 Main thread (UI only)              Engine Worker (wasm, pthreads)              VFS Worker (JS)
 ───────────────────────           ──────────────────────────────────          ───────────────────────────
 • lobby / setup page               • Com_Frame main loop                      • OPFS block cache (16 KiB)
 • canvas → transferControl-        • render backend thread owns the           • HTTP Range fetch to asset
   ToOffscreen() → engine             WebGL2 context (OffscreenCanvas)           host or local import
 • pointer lock, keyboard/mouse     • database thread: zone loading ──sync──►  • per-map manifest prefetch
   events → postMessage / SAB       • sound mixer → OpenAL → AudioWorklet        (8 MiB × 6 parallel)
 • COOP/COEP headers set by host    • netchan → WebRTC / WebSocket shim        • zip central-dir index for IWDs
```

* The engine stays **synchronous C++**. File reads block on a `SharedArrayBuffer` +
  `Atomics.wait` handshake with the VFS worker, which does the async `fetch`/OPFS work.
  This avoids Asyncify, which bloats code and slows hot paths. JSPI (`-sJSPI`) is the
  alternative for the single-thread Firefox fallback if it is needed.
* Cross-origin isolation (`Cross-Origin-Opener-Policy: same-origin`,
  `Cross-Origin-Embedder-Policy: require-corp`) is required for SharedArrayBuffer. A static
  host with custom headers is enough, e.g. nginx, Cloudflare Pages, or a tiny Rust/Go server.

---

## 4. Work breakdown

### Phase 0: Portable, clang-clean native build (prerequisite)

Emscripten is clang. Get KisakCOD compiling **with clang on Linux, 32-bit (`-m32`)** first,
using the existing `scripts/platform/linux` stub. That way most errors are fixed with a
native debugger available, before wasm is in the picture.

1. Add a `src/_platform/posix/` layer (the override mechanism already exists) for: `Sys_Milliseconds`,
   `Sys_CreateThread`/`Sys_CreateEvent`/`Sys_WaitForSingleObject` (pthread mutex+cond),
   `Interlocked*` → `std::atomic`/`__atomic`, `VirtualAlloc/VirtualFree` → `malloc`/`mmap`
   (`src/universal/physicalmemory.cpp`, `com_memory.cpp`), and Win32 file APIs → POSIX.
2. Replace inline x86 asm with C (`fld/fistp` → `lrintf`; `__asm` memset → `memset`).
   The PPC `__asm` blocks in `cg_ents.cpp`/`cg_servercmds_mp.cpp` are Xbox 360 leftovers. Confirm
   they are already `#if`'d out, or delete them.
3. Replace `__try/__except` with plain code or `setjmp`. Map `__cdecl`, `__int16`, `_stricmp`
   and similar through a compat header.
4. Build a renderer-less **dedicated server** (`scripts/dedi`) on Linux. It exercises everything except
   gfx, sound and input, and it becomes the server for browser clients later (§4.6).
5. Add CI: Linux clang build of dedi + MP client (with a null renderer) on every PR.

*Exit criterion:* a Linux dedicated server loads `mp_crash` and runs bots. The MP client links
against a null renderer and can `devmap mp_crash` headless.

### Phase 1: Emscripten bring-up (headless)

1. Add `scripts/platform/web/platform.cmake`. Toolchain `emcmake cmake`, flags
   `-pthread -sPTHREAD_POOL_SIZE=8 -sALLOW_MEMORY_GROWTH=1 -sMAXIMUM_MEMORY=4GB
   -sOFFSCREENCANVAS_SUPPORT=1 -sUSE_ZLIB=1 -msimd128 -msse2` (Emscripten emulates
   SSE intrinsics over wasm SIMD, which covers `r_model_skin_sse.cpp` and similar).
2. Use **WasmFS** with a custom backend (or plain MEMFS for now) holding a tiny test set:
   `zone/english/code_post_gfx_mp.ff`, `common_mp.ff`, `mp_crash.ff`, `localization.txt`.
3. Use a null renderer and null sound. Get `devmap mp_crash` to load and tick in node or a headless browser.

*Exit criterion:* the server + client sim loop runs in wasm, and the console prints to the browser devtools.

### Phase 2: Asset streaming VFS

CoD4 data on disk (about 7 GB for a full install, much less per map):

* `zone/<lang>/*.ff`: zlib-compressed fastfiles, read **sequentially** by the database thread.
* `main/iw_00.iwd … iw_13.iwd`: zip files holding `images/*.iwi` (loose textures), `sound/*`, etc.
* `main/localized_*.iwd`, `players/`, `mods/`, config files.

Design:

1. **Asset source.** The user supplies their own CoD4 files. Two supported modes:
   * *Local import*: the user picks their CoD4 folder (File System Access API / `<input webkitdirectory>`).
     Files are copied lazily into OPFS. Nothing leaves their machine.
   * *Private asset host*: the user (or their community, for a server they run) serves their own install over HTTPS
     with `Range` support. **We never host or ship Activision assets.**
2. **Block cache.** The VFS worker keeps a sparse OPFS file per source file plus a bitmap of
   present 16 KiB blocks. A read for missing blocks becomes one HTTP Range request (coalesced).
3. **Fastfiles.** Hook `DB_ReadXFileStage()` so its 256 KiB reads go through the VFS. Because zones are
   read front to back, prefetch sequentially (streaming `fetch` body → OPFS), so inflate overlaps download.
   Later, store fastfiles pre-inflated server-side, so the browser skips zlib work and can resume ranges.
4. **IWDs.** On first touch, fetch only the zip **central directory** (end of file) and build an
   index. `FS_FOpenFileRead` for an IWD entry then becomes a Range read of exactly that entry's
   bytes. No read-ahead over the whole pack (the mw2web lesson).
5. **Manifests + packs.** In dev builds, log every `(file, block)` read during a map load.
   Then an offline tool (`tools/webpack/`) emits `manifests/<map>.json` and a concatenated
   `packs/<map>.bin`. The loading screen fetches the pack in 8 MiB ranges, 6 at a time, before
   the engine asks for anything. Also build a `common.bin` pack for menus and `common_mp.ff`.
6. **Prewarming.** Using the manifest, read sound aliases and the first-frame images into memory
   during the loading screen.
7. **Integrity + versioning.** Each manifest lists file hashes. A changed hash invalidates the OPFS blocks for that file.

*Exit criterion:* a cold load of `mp_crash` from an HTTP asset host completes, and a warm
reload reads 0 bytes from the network.

### Phase 3: Renderer: D3D9 → WebGL2

Do **not** rewrite `gfx_d3d`. Insert a translation layer below it:

1. **`IDirect3DDevice9` shim** (`src/_platform/web/d3d9_webgl/`). Implement only the subset KisakCOD
   uses (audit the 620 call sites; expect around 60 distinct methods): buffers, textures (2D, cube, volume),
   render targets and depth-stencil surfaces, render states, sampler states, vertex declarations,
   `DrawIndexedPrimitive`/`DrawPrimitive(UP)`, `StretchRect`, `Clear`, queries (occlusion → `EXT_disjoint_timer`/
   `ANY_SAMPLES_PASSED`), and `Present`. The D3D9 → GL conventions to handle:
   * clip-space z (0..1 vs -1..1): use `EXT_clip_control` where available, otherwise patch in the vertex shader;
   * the half-pixel offset; Y-flip for render targets; the `D3DCOLOR` BGRA vertex format;
   * depth textures for shadow maps (CoD4 uses hardware PCF on D24S8 → `sampler2DShadow`).
2. **Shader translation.** Fastfiles carry D3D9 bytecode. Translate it at `Load_CreateMaterial{Pixel,Vertex}Shader`
   time with **[ToGL](https://github.com/ValveSoftware/ToGL)** (MIT; `dx9asmtogl2.cpp`), retargeted to GLSL ES 3.00.
   mw.slqnt.dev ships exactly this for CoD4 (§1b). [MojoShader](https://github.com/icculus/mojoshader) (zlib) is the fallback.
   Cache translated programs by bytecode hash (in IndexedDB/OPFS), so later loads skip translation and
   most driver compile time.
3. **Textures.** `.iwi`/fastfile images are mostly DXT1/3/5. Desktop WebGL2 exposes
   `WEBGL_compressed_texture_s3tc`, so upload them directly. For devices without S3TC, decode to RGBA8 in a worker.
4. **Linking + state caching.** GL needs VS+PS program pairs. Link them lazily on first draw and
   cache them. Precompile the pairs listed in the per-map manifest during the loading screen.
5. **Threading.** `r_smp_backend` / `r_smp_worker` already split frontend and backend. Create the
   WebGL2 context on the backend pthread via `OffscreenCanvas`. Start with `r_smp_backend 0` for bring-up.
6. **Later: WebGPU backend.** Once WebGL2 works, MojoShader's SPIR-V output can go through Tint/Naga
   to WGSL for a WebGPU device shim. It brings better multithreaded command recording and fewer driver stalls.
   Treat it as an optimization, not a blocker.

*Exit criterion:* `mp_crash` renders correctly next to the PC build in a comparison screenshot (world, models,
sun shadows, spot shadows, FX, HUD and menus).

### Phase 4: Input, audio, UI shell

* **Input.** Replace `src/win32/win_input.cpp` with a web version. Main-thread `keydown/keyup`,
  `mousemove` (with **Pointer Lock** and `movementX/Y`) and `wheel` events go to the engine worker through a SAB ring buffer.
  Map `KeyboardEvent.code` to engine keynums. Add Gamepad API support as a bonus.
* **Audio.** Build with `KISAK_OPENAL=ON` against Emscripten's OpenAL. Finish its TODOs
  (`snd_driver_openal.cpp`). If OpenAL-on-WebAudio latency or main-thread coupling becomes a problem,
  replace it with a mixer writing into an **AudioWorklet** ring buffer (mw2web did this, and underruns dropped to zero).
  Streamed music/ambience: decode MP3 in a worker with `dr_mp3`.
* **Shell page.** A setup/lobby page covering the asset source, a server browser and settings. Fullscreen + pointer lock
  on click. `beforeunload` guard. `cfg` files persisted in OPFS (`players/profiles/...`).

### Phase 5: Multiplayer networking

Browsers have no raw UDP, so the Quake 3-style unreliable netchan needs a transport shim:

1. **Client side.** Replace `Sys_SendPacket`/`Sys_GetPacket` (`src/win32/win_net.cpp`) with a
   **WebRTC DataChannel** (`ordered:false, maxRetransmits:0`), which keeps UDP semantics. Use WebSocket as the fallback
   (it adds head-of-line blocking, but it works everywhere).
2. **Server side.** Run the native KisakCOD dedicated server from Phase 0 unchanged, and add a small **gateway**
   (Rust or Go, e.g. using `webrtc-rs`/`pion`) that terminates DataChannels/WebSockets and forwards datagrams to the
   server's UDP port, one local UDP socket per browser client. Native PC clients and browser clients can then share
   a server.
3. **Discovery.** Replace the master-server query with a small HTTPS server list API served by the gateway.
   Steam auth is stubbed. Add simple account/GUID tokens if needed.
4. **Listen server in browser.** Optional. Loopback already works in-process, so private matches can be hosted
   in-tab, but only through the gateway (as a relay), because browsers cannot accept inbound connections.

### Phase 6: Cinematics, voice, polish

* **Bink.** Bink is closed source, but FFmpeg's open `binkvideo`/`binkaudio` decoders compile to wasm. Implement
  the `Bink*` API used by `r_cinematic.cpp` on top of `libavcodec`/`libavformat` (LGPL; build with only the bink
  demuxer and decoders enabled), and do YUV→RGB in a shader. mw.slqnt.dev does this (§1b). For the MVP, skip videos.
* **Voice.** speex compiles to wasm. Mic capture via `getUserMedia` → AudioWorklet → speex encoder. Low priority.
* **Steam.** Compile `win_steam.cpp` out behind a `KISAK_STEAM` option.

### Phase 7: Performance and memory

* `-O3 -flto`, `-msimd128`, strip names, serve `.wasm` with brotli/gzip and `instantiateStreaming`.
* Measure memory per map. Free load-time-only data. Lower `com_hunkmegs`-style pools for web defaults.
* Cap concurrent shader translation/linking during loads (mw2web: `max(2, cores/2)`).
* Dynamic resolution (render scale steps) when GPU-bound.
* Firefox/Safari: verify that a SAB + OffscreenCanvas + WebGL2 build works. If not, ship a single-threaded build
  (`r_smp_* 0`, the database thread run cooperatively, JSPI for blocking reads).

---

## 5. Suggested milestones

| # | Milestone | Depends on |
|---|---|---|
| M0 | Linux clang 32-bit dedicated server runs bots on `mp_crash` | Phase 0 |
| M1 | wasm headless `devmap mp_crash` ticks in the browser console | M0, Phase 1 |
| M2 | Streams `mp_crash` from an HTTP asset host with an OPFS cache | M1, Phase 2 |
| M3 | First rendered frame (menus); then the world | M1, Phase 3 |
| M4 | Playable offline vs bots: input, audio, HUD | M2, M3, Phase 4 |
| M5 | Browser client joins the native Linux dedicated server through the gateway | M4, Phase 5 |
| M6 | Manifest packs, shader cache, perf pass; Chrome/Edge release, Firefox fallback | M5, Phases 6–7 |

Start with **MP with a local `devmap`**. It exercises the full client + server stack over loopback, with no
networking shim, and matches the eventual multiplayer goal. SP campaign maps reuse everything afterwards
(`scripts/sp`), mostly needing larger memory budgets and Bink.

---

## 6. Risks and open questions

* **Legal.** Assets are Activision IP. The project must keep a strict "bring your own files" model
  (local import or a self-run private asset host). No public mirror of `.ff`/`.iwd` data.
* **D3D9 feature edge cases.** Shadow-map PCF, sRGB handling, the half-pixel offset, `StretchRect`
  between MSAA surfaces, and per-vertex `D3DCOLOR` swizzles are the usual sources of visual bugs. Budget time for a
  comparison harness (same demo, same frame, PC vs web screenshots).
* **MojoShader coverage.** It needs verifying against every shader in `code_post_gfx_mp.ff` and the map
  fastfiles. Write a dump tool that translates and compiles all of them offline and reports failures before
  in-engine testing.
* **Thread/event semantics.** Win32 auto/manual-reset events and `WaitForMultipleObjects` must be emulated
  exactly. Bugs there show up as rare hangs on load.
* **Main-thread blocking.** Emscripten forbids `Atomics.wait` on the main thread, so all engine code must run in workers
  (the architecture in §3).
* **Safari.** Its OffscreenCanvas WebGL and OPFS support has historically lagged. Treat Safari as best-effort.
* **Anti-cheat / exploits.** The README already warns about old-engine exploits. A browser sandbox helps
  the client side, but the dedicated server still needs hardening.

---

## 7. Immediate next steps

0. Contact slqnt (https://slqnt.dev) for the GPLv3 source of the mw.slqnt.dev build (§1b). If it is
   provided, rebase this plan onto it. Phases 0, 1, 3 and 4 are largely done there, and Phase 2
   (host streaming) plus Phase 5 (MP against native servers) become the contribution.
1. Land the Phase 0 portability layer behind `KISAK_PLATFORM=linux`, and get dedi building with clang `-m32`.
2. Write `tools/shaderdump`: extract every vertex/pixel shader from a fastfile and run it through ToGL → GLSL ES →
   `glslangValidator`, to measure translation coverage early (the biggest technical unknown).
3. Prototype the VFS worker (OPFS block cache + Range fetch + `Atomics.wait` bridge) as a standalone
   Emscripten test, independent of the engine.
4. Audit `dx.device->` usage into a list of distinct D3D9 methods and states, to size the shim.
