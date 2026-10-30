# LSFG for MediaTek — Porting Notes (Phases 1–4 × 5 sub-phases)

Project home: `~/workspace/lsfg-mediatek/` · Started 2026-10-30 · Owner: Buggy, for Syed.
Pre-restructure notes preserved at `PORTING_NOTES_v1.md`.
Standing rule: crucial findings escalate immediately (top box below), not at phase ends.

## 🚨 CRUCIAL FINDINGS (escalated as they landed)

1. **A robustness2-free LSFG core already exists — the fork's.** Upstream lsfg-vk
   v1.0.0 hard-requires `VK_EXT_robustness2` (`framegen/src/core/device.cpp:18`,
   in `requiredExtensions`, feature struct chained unconditionally). The
   lsfg-vk-android **fork removed the requirement**: device.cpp probes it,
   enables it only if present, and routes every null-descriptor write to a
   1×1 fallback image. Phase 4.3 is therefore *verify + harden*, not rewrite.
   Full enumeration in §2.5. (The app README's "Mali falls back to mirror
   mode" claim is stale relative to the fork's code.)
2. **DLL pairing verdict — Syed's Lossless.dll 3.2.1.0 WORKS, with one cost.**
   All 48 DXBC shader resources the engine needs (RCDATA IDs 255–302) are
   present → extraction + on-device DXBC→SPIR-V translation fully supported.
   But the precompiled FP16 SPIR-V set (IDs 304–351) and FP32 SPIR-V set
   (353–400) that newer DLL builds ship are **absent in 3.2.1.0** (its
   RCDATA range is 101–302). The loader treats FP16 as best-effort
   (`android_shader_loader.cpp:369`: "a Lossless.dll build that doesn't ship
   the FP16 set must still produce a working FP32 [path]"). Cost = FP32
   pipeline instead of the FP16 fast path on the G610 (performance, not
   function). **Recommendation: Syed updates Lossless Scaling in Steam and
   re-supplies the DLL before Phase 4.5 perf tuning. Not a build blocker.**
3. **PanVK + robustness2 — SETTLED (binary-verified 2026-10-30):** the
   shipped RC4 ICD (`libvulkan_panfrost-android-aarch64.so`, Mesa
   `5a07217f`, sha256 in §4.2) contains the advertised-extension strings
   **`VK_EXT_robustness2`** AND **`VK_ANDROID_external_memory_android_
   hardware_buffer`** (plus timeline_semaphore, vulkan_memory_model,
   synchronization2, descriptor_indexing, external_memory_fd). The
   wrapper tree itself has zero mentions (expected — code lives in
   Mesa) and project docs claim nullDescriptor proven on G615; the
   binary check confirms the support is compiled into the exact build
   we bundle. Per-device exposure on the 10R is still probe-verified
   (P-1) — and thanks to finding #1, the port doesn't depend on it
   either way: PanVK is wanted for driver control, not as a
   robustness2 rescue.
   **Full evidence chain (research sweep, WORKAROUNDS.md §1):** Mesa's
   feature matrix lists `VK_EXT_robustness2` DONE for `panvk/v10+`
   (G610 is v10); Mesa 26.0 promoted core `VK_KHR_robustness2` in
   PanVK v10+; the kbase project records robustness2 buffer +
   nullDescriptor proven on G615 with a **CTS robustness2 run of
   4332 tests / 0 failures**; Collabora brought PanVK to Vulkan 1.2
   conformance **on the G610 itself** (June 2025). Our teardown's
   binary check is consistent with every link. **Consequence: the
   robustness2-free core rework (§4.3) is DEMOTED from centrepiece
   to insurance for the stock-driver fallback path** — under PanVK
   the extension wall is a property of the stock Arm driver, not of
   the G610 silicon.
4. **Licence:** LSFG-Android-Application subtree = custom licence (no store
   publication / commercial use; personal experimentation OK). Our shell is
   original code calling the same native engine; the engine subtree
   (lsfg-vk-android) is MIT. Lossless.dll is **never** packaged — runtime,
   user-supplied only (the sanctioned copy sits in `user-supplied/`, kept
   out of every deliverable).

## Target & constraints

- **Device:** OnePlus 10R 5G (CPH2423), Dimensity 8100-MAX, **Mali-G610 MC6**
  (Valhall, Mesa PAN_ARCH **v10**, **CSF** frontend), kernel 5.10 family,
  stock Mali Vulkan 1.3-class driver, NO root.
- **Driver strategy (Syed's priority change):** **PanVK is PRIMARY** — the
  engine should run on the panvk-kbase ICD bundled in-process, the way
  Adreno apps bundle Turnip. Stock Mali driver = comparison/fallback path.
- **Engine:** LSFG primary (Syed owns Lossless Scaling on Steam; supplies
  his own DLL at runtime). Open DIS engine = documented fallback (DukePS3
  reference, §2.3 note).

## Inventory (all verified in-tree)

| Item | Path | Commit / version |
|---|---|---|
| lsfg-vk upstream v1.0.0 | `lsfg-vk-upstream/` | `7113d7d` (GitHub HEAD is a migration notice, `0af010d`) |
| LSFG-Android umbrella | `LSFG-Android/` | `da867e9` |
| ↳ lsfg-vk-android (MIT) | `LSFG-Android/lsfg-vk-android/` | `b55b182` (+ Patch #1, §4.3) |
| ↳ LSFG-Android-Application (custom licence) | `LSFG-Android/LSFG-Android-Application/` | `b847541` |
| DukePS3 (reference) | `DukePS3/` | `81c109e` (shallow) |
| panvk-kbase-android | `panvk-kbase-android/` (full clone blocked by sandbox proxy — see Phase 1 status; key files API-fetched) | HEAD `dd0f6bf` per GitHub; Mesa pin source-verified via `sources.lock` = `5a07217f034b3e50d8c7c7794f97a2df1742613b` |
| LSFG-Android release APK 0.1.2 | `teardown/LSFG-Android-0.1.2.apk` | 7,963,871 B, sha256 `ec4d16fa…`, pkg `com.lsfg.android` |
| Syed's Lossless.dll | `user-supplied/Lossless.dll` | **3.2.1.0**, 5,435,904 B, sha256 `d8bde39e…` — NEVER packaged |
| Built engine | `build-arm64/liblsfg-android.so` | 18,187,408 B arm64 (unstripped), BuildID `9b69157c…` |

---

# PHASE 1 — PanVK deep study

*Status: doc-verified from the project's own README/docs/device matrix at
HEAD `dd0f6bf`, and — as of 2026-10-30 — **source-verified for the
load-bearing files** (profiles, sources.lock, LICENSES, patch-series
layout, packaging scripts) fetched individually via the GitHub API.
The full-tree clone itself is **blocked in this sandbox**: three
routes failed through the egress proxy (2× git clone — "invalid
index-pack output" pack corruption, ~19 and ~27 min; 1× codeload zip
— curl exit 23 mid-transfer; the repo is ~880 MB). Items below are
marked [docs] or [source] accordingly; [source] = read from the
repo's own files at `main` during this verification pass.*

## 1.1 Source inventory
- [docs] panvk-kbase-android is a **patch + build layer around pinned
  upstream Mesa (26.3.0-devel, commit `5a07217f`)**: per-GPU `profiles/`,
  a numbered patch series applied to Mesa's PanVK/Panfrost, build/package
  scripts. It is not a fork checked into the tree — Mesa is fetched at
  build time (this is why wrapper-tree greps find no extension code).
- [docs] Output: one **universal ICD** covering v6/v7 (JM), v9 (JM),
  v10–v14 (CSF); Android library name `libvulkan_panfrost.so`.
- [docs] Stated G610 support: Mali-G610 = PAN_ARCH **v10**, CSF frontend;
  device matrix (RC3, 2026-10-11) lists G610 "🧪 Tested — works".
- Licences [source: `LICENSES/README.md`]: the tree is **explicitly not
  uniformly MIT** — repo scaffolding (scripts/profiles/packaging) is
  MIT; Mesa sources are upstream Mesa (mostly MIT/X11 + Khronos); the
  committed **kbase UAPI headers are GPL-2.0 WITH the Linux-syscall-note**
  (the exception is what allows MIT userspace to include them without
  relicensing); the userspace kmod backend is MIT. Our use — bundling
  the project's own release binary — is unaffected, but anyone building
  from source should read that file first.
- [source] `sources.lock` confirms the Mesa pin exactly:
  `5a07217f034b3e50d8c7c7794f97a2df1742613b`, 26.3.0-devel, pinned
  2026-09-18, with a `patchSeriesId` hash over commit+profiles+patches;
  `scripts/fetch-mesa.sh` does a depth-1 fetch of exactly that commit.
  The RC4 release we bundle is built from this pin (§4.2).

## 1.2 Kernel bridge
- [docs] PanVK talks **directly to the vendor `mali_kbase` kernel driver**
  via `/dev/mali0` ioctls — the phone's own kernel side stays untouched,
  no root: the app loads the userspace ICD in-process.
- [docs] Per-GPU profiles pin arch, `gpu_id`, frontend (JM/CSF) and the
  **kbase uAPI version**; runtime probes of the GPU ID string select
  behaviour. G610 reference device: motorola edge 40 neo (MT6879,
  Mali-G610 MC3, gpu_id `0xa8670000`, kernel **5.10.218**, **CSF 1.18**) —
  **PanProbe 38/38**, D3D11 cube 47 FPS.
- [docs] Memory ioctls: `MEM_ALLOC` with `MEM_ALLOC_EX` fast path.
  **Confirmed quirk:** that G610's kbase rejects `MEM_ALLOC_EX` with
  `ENOTTY` → the `MEM_ALLOC` fallback (cited as patch 110 in the
  project's docs/release notes) is load-bearing. A Dimensity 8100
  (kernel 5.10, CSF 1.x) should expect the same. [source caveat: at
  current HEAD the patch series is reorganised by family
  (`patches/{kbase-common,csf,csf-v11,jm-v7,jm-v9,android,wsi,…}`,
  small per-family numbers) — the docs' 075/110/122-style numbers do
  not map 1:1 onto the HEAD tree; treat them as release-note
  citations, not file names.]
- [source] Access-model precedent from the G615 profile
  (`profiles/g615-v11-csf.json`): `/dev/mali0` mode `crw-rw-rw-`,
  SELinux label `gpu_device`, and the project's probe recorded a
  **true app UID opening the node O_RDWR successfully under SELinux
  enforcing** (on Poco/HyperOS). Encouraging for our no-root
  in-process model — on a sibling MediaTek ROM, not on OnePlus.
- 8100's exact shipped kbase version: **unproven** — no Dimensity 8100
  device has been tested by the project; OnePlus's kbase build + SELinux
  policy on `/dev/mali0` for unprivileged apps are the open questions
  (answered on-device in Phase 4.5, step P-1).

## 1.3 Command submission (v10 = CSF)
- [docs] v10+ uses the **CSF command-stream frontend**: queue groups /
  subqueues, GPU-managed scheduling, tiler-heap arenas committed up
  front, GPU-side semaphore waits, sync via CSF event-memory sync words
  + `sync_fd` export through the kbase KCPU queue (patch 075).
- [docs] JM arches (v7/v9) submit **atoms** (56/64-byte stride check) —
  not our path: **G610 is CSF**, the same frontend family as the v11
  reference target.
- [docs] Fences/timeline: Phase-5 workloads report **timeline semaphore
  PASS, synchronization2 PASS** on the reference device — the two sync
  primitives LSFG's chain depends on.

## 1.4 Memory & shaders
- [docs] Allocations are kbase `MEM_ALLOC` ioctls (§1.2); placed maps
  re-map the BO's dma-buf at the requested VA (patch 091).
- [docs] Shader path: Mesa's NIR → Panfrost compiler to Valhall ISA,
  in-driver; pipeline cache is Mesa's standard cache. Compute is
  PanVK's strongest tier ("alpha" gate passed early) — LSFG is pure
  compute, which is why PanVK is credible as primary here.
- [docs] Feature claims relevant to LSFG (see also Crucial #3):
  Vulkan 1.4.363 enumeration, 181 device extensions on the reference;
  push descriptors, descriptor indexing, dynamic rendering PASS.
- [docs] Sparse binding: **NO-GO** on kbase uAPI 1.21 — LSFG doesn't
  need sparse; noted so nobody designs on it.

## 1.5 Packaging & device fit
- [docs] Releases ship: Android + glibc driver packages, an **`.adpkg`**
  (built by `scripts/package-android-adpkg.sh`), an EMULATOR zip, and
  **launcher zips for adrenotools, Winlator, cmod, Ludashi, bannerlator,
  GameHub** (RC4) — i.e. the standard driver-package shape (ICD .so +
  ICD json).
- [docs] In-process precedent: **Samba S3** runs it as an "in-process
  custom Vulkan driver" on Android/Bionic; **PanPlay** (their launcher)
  bundles + loads it in-process; **PanProbe** is their capability prober.
  This is exactly the bundling model Phase 4.2 copies.
- Driver identity: `driverName=panvk`, `driverInfo="Mesa 26.3.0-devel
  (git-<sha>)"`.
- **8100-specific gaps (unproven until on-device):** G610 **MC6** variant
  match (tested unit is MC3; core count is runtime-read, product family
  `0xa867` shared — expected OK, unverified); OnePlus kbase/SELinux
  behaviour; **AHB import under PanVK** — the project's tests include an
  `ahb` target; the bundled ICD's binary advertises the AHB extension
  (Crucial #3), but per-device exposure is the probe's job. If absent,
  the PanVK session needs frames via dma-buf fd import
  (`VK_KHR_external_memory_fd`, which PanVK speaks) or a CPU copy —
  decided by the Phase 4.5 probe.
- **MT6895 GAP (research sweep, WORKAROUNDS.md §6):** across the
  project's **88+ device records there is NO Dimensity 8100 / MT6895
  test row at all** — the 8100's kbase is older (2022-era 5.10) than
  the tested edge 40 neo's. Closest analogues say "should work";
  **Syed's 10R run is the first data point**, which is exactly the
  tester-shaped hole the project asks people to fill.
- **v10 stability caveats (proven, build-dependent):** an earlier
  build (115) hung GPU queues on Pixel 7/G710 ("kbase: timeout on
  subqueue 0") — fixed for the tested cases in 115b; a class of
  v10/v11 DEVICE_LOST remains open around **draws without a render
  target → kbase tiler-heap OOM**. Our workload is compute-only (no
  RT-less draws), which sidesteps the known trigger — **marked as
  inference, not a test result.** Corollary warning from the sweep:
  PanProbe passing 39-40/40 on hardware that hangs in games means
  feature probes ≠ workload stability — budget real soak time
  (P-3), don't trust a green probe alone.
- **Phase 1 verdict:** PanVK-on-G610 is real, tested on a sibling
  MT6879/G610 device, CSF-native, compute-strong. The port's PanVK risk
  is concentrated in OnePlus-specific kernel/SELinux behaviour and AHB
  interop — both on-device questions, neither a paper blocker.

### Phase 1 — Flaws & findings

**FLAWS (PanVK project / its documentation):**
- F1.1 *Claims outrun the tree:* the wrapper repo contains no driver
  code for the features its docs advertise (grep of the tree for
  `robustness2` = zero hits — the code lives in upstream Mesa, fetched
  at build). Anyone auditing the repo alone cannot verify its feature
  claims; verification requires the pinned-Mesa fetch (§4.2).
- F1.2 *Reference-device overgeneralisation:* the headline "G610
  works" rests on ONE device — motorola edge 40 neo, **MC3** variant,
  MT6879, Motorola ROM (38/38 PanProbe). The 8100's **MC6** + OnePlus
  ROM/kbase have never been near the test matrix; the docs' framing
  makes this easy to miss.
- F1.3 *robustness2 evidence is arch-mismatched:* the "nullDescriptor
  proven" claim comes from a **G615 (v11-class) beta.3 eval**; the
  v10-specific landings are narrower (`robustBufferAccess2`, patch
  122; `robustImageAccess2` RC2). Reading the docs quickly overstates
  v10 coverage. (Mitigated for us by Crucial #1 — we don't depend on
  it — but recorded so nobody else does either.)
- F1.4 *Kernel dialect fragility:* `MEM_ALLOC_EX` → `ENOTTY` on the
  one tested MediaTek kbase (patch-110 fallback exists) — proof that
  kbase uAPI behaviour varies by OEM build; each new OEM is a fresh
  compatibility question, unlike the KGSL monoculture (§3.4).
- F1.5 *Ambiguous AHB signal:* an `ahb` test target exists but AHB
  extension exposure is nowhere confirmed in docs — the one capability
  our capture path needs most is the least documented.
- F1.6 *Maturity:* project is months old, RC-stage; sparse binding is
  a hard NO-GO on kbase (their own feasibility doc).
- F1.7 *The G610 profile itself is unvalidated [source]:*
  `profiles/g610-v10-csf.json` is a 174-byte stub whose status reads
  **"planned P24 (not yet validated)"** — while the g615 profile is a
  full validated record ("primary bring-up target", probe logs, GPU
  ID, uAPI 1.21) and the README/device matrix reports a G610 device
  passing PanProbe 38/38 on RC3. Both artefacts are the project's
  own; the honest read is that G610 support is *demonstrated on one
  device but not yet promoted to a validated profile*. Our Phase 1
  verdict stands (PanVK-on-G610 is real), with its confidence
  correctly capped: the port is early-adopter territory on this
  GPU, and the 8100/MT6895 doubly so (§1.5 MT6895 gap).

**FINDINGS:**
- CONFIRMED [docs]: G610 = PAN_ARCH v10, CSF frontend; universal ICD
  (v6–v14); `.adpkg` + launcher-zip packaging; in-process bundling
  precedent (Samba S3, PanPlay); timeline-sem + sync2 PASS on the
  reference device; `MEM_ALLOC` fallback for the ENOTTY quirk.
- CONFIRMED (sweep + local binary check, mutually consistent):
  Mesa lists `VK_EXT_robustness2` DONE for PanVK v10+, KHR
  promotion in Mesa 26.0, G615 CTS robustness2 run 4332/0,
  Collabora Vulkan 1.2 conformance on the G610 itself, and the
  bundled RC4 ICD advertises the extension (Crucial #3).
- SUSPECTED: MC6 variant + OnePlus kbase/SELinux will behave like the
  Motorola reference (plausible, unproven — settled by probe P-1);
  PanVK exposes AHB import on the 10R (driver advertises it;
  per-device exposure settled by probe P-1/P-2). **MT6895 has no
  test record anywhere (88+ device records) — the 10R is the first
  data point.** Compute-only workloads sidestep the known v10
  tiler-heap OOM trigger — inference, not measurement.

---

# PHASE 2 — LSFG APK teardown (reverse engineering)

*Evidence: release APK 0.1.2 (aapt2/dexdump/readelf/nm), the full
LSFG-Android source tree at `da867e9`, and a stdlib PE parse of Syed's
DLL (structure only — no shader contents extracted or recorded).*

## 2.1 APK surface
- Package `com.lsfg.android`, versionName **0.1.2**, minSdk **29**,
  targetSdk 35. Application class `com.lsfg.android.LsfgApplication`.
- Permissions: INTERNET, FOREGROUND_SERVICE (+ MEDIA_PROJECTION and
  SPECIAL_USE types), SYSTEM_ALERT_WINDOW, POST_NOTIFICATIONS,
  `moe.shizuku.manager.permission.API_V23`, a signature-level
  DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION. `uses-feature
  android.hardware.vulkan.version`.
- Components: `ui.MainActivity` (launcher), `ui.ProjectionRequestActivity`;
  services `session.LsfgForegroundService` (fg type `0x40000020` =
  mediaProjection|specialUse), `session.RootCaptureService`,
  `session.LsfgAccessibilityService` (accessibility service — the OEM
  untrusted-touch workaround); providers: androidx FileProvider,
  `rikka.shizuku.ShizukuProvider`, androidx.startup.
- Contents: single `classes.dex` (2,467,552 B; Compose UI — androidx
  compose/material3/lifecycle/navigation dominate, plus `com.topjohnwu`
  libsu for the root-capture option), `assets/main.jar` (3,656 B,
  Shizuku user-service), baseline profiles, ~1.9 MB of UI image
  resources. arm64-v8a only.

## 2.2 Native layout
- `lib/arm64-v8a/`: `liblsfg-android.so` (**1,119,376 B**, stripped
  release build), `libc++_shared.so` (1,292,896 B),
  `libandroidx.graphics.path.so` (10,096 B).
- `liblsfg-android.so` DT_NEEDED: liblog, **libneuralnetworks** (NNAPI
  post-processing option), libandroid, libm, libc++_shared, libdl, libc.
  No direct libvulkan NEEDED — Vulkan is loaded via volk (dlopen).
- JNI surface = `com.lsfg.android.session.NativeBridge` (**23 external
  funs**, identical in source and release exports): nativeVersion,
  initCrashReporter, extractShaders, probeShaders, initContext,
  isNpuAvailable, isFramegenFp16Supported, getNpuSummary,
  setOutputSurface, pushFrame(HardwareBuffer, ts), frame counters
  (generated/posted/unique-capture), latency/queue stats,
  setBypass, setAntiArtifacts, setVsyncPeriodNs, setPacingParams,
  setShizukuTimingEnabled, reportShizukuTiming, destroyContext.
- The `.so` also exports the raw C++ engine API (`LSFG_3_1::initialize /
  createContext / createContextFromAHB / presentContext / deleteContext
  / finalize / waitIdle`, plus `LSFG_3_1P::` performance-variant twins) —
  our Phase 4 shell binds through the JNI class (name-matched), not the
  C++ symbols.

## 2.3 Engine core
- Pass chain (LSFG 3.1 and 3.1P): **alpha → beta → gamma → delta →
  mipmaps → generate**; flow scale 0.25–1.0; multiplier 2×–8× via
  repeated generate passes. Synchronization is timeline-semaphore
  based; barriers via sync2 with a sync1 translation fallback for
  pre-1.3 devices (device.cpp comments — written for Mali-G57-class
  parts, our G610 is 1.3-class).
- **Shader supply (the DLL contract):** user picks their own Lossless.dll
  via SAF → copied to private storage → PE resource walk (pe-parse
  `IterRsrc`, RCDATA only) → per-ID cache of SPIR-V in app cacheDir →
  DLL copy deleted. Two production paths (`android_shader_loader.cpp`):
  (a) base **DXBC IDs 255–302** translated on-device by DXVK's dxbc
  compiler (entry `CS`), with the SPIR-V version word capped 1.6→1.3
  for Mali (`cap_spirv_version`) and bindings renumbered dense;
  (b) **precompiled SPIR-V variants shipped inside newer DLLs** — FP16
  at ID+49 (304–351), FP32 at ID+98 (353–400) — cached verbatim (FP16
  after a binding rewrite), magic-validated, **best-effort**.
- **Syed's DLL 3.2.1.0 (structure only):** PE32+ x86-64; `.rsrc` =
  5,110,776 B holding **202 RCDATA leaves (IDs 101–302, all lang 1033,
  5,099,468 B total)**, 1 VERSION leaf (VS_FIXEDFILEINFO confirms
  **3.2.1.0**), 1 MANIFEST (381 B). All 48 required DXBC IDs (255–302)
  present — e.g. ID 256 (`p_generate`) = 3,132 B; the large uniform
  119,284 B entries are the big pass blobs. **No IDs ≥304** → no FP16 /
  FP32 precompiled sets (see Crucial #2).

## 2.4 Capture & composite
- MediaProjection consent → VirtualDisplay + ImageReader produce
  **AHardwareBuffer** frames (zero CPU copy). Kotlin pushes each AHB
  via `NativeBridge.pushFrame`; natively the app session
  (`android_vk_session.cpp`) imports it, a `VK_QUEUE_FAMILY_FOREIGN_EXT`
  transfer hands it to the framegen device's `createContextFromAHB`,
  and the two devices are bridged by `waitIdle()` (no cross-device
  semaphore exists). Outputs composite to the overlay `ANativeWindow`
  (Vulkan swapchain, CPU-blit fallback).
- AHB sharing is mandatory, not a choice: Mali/Adreno drivers refuse
  `vkGetMemoryFdKHR` on AHB-imported memory (`ahb_image_bridge.cpp`),
  which is why upstream's fd path is dead on Android.
- Pacing (`lsfg_render_loop.cpp`, native): vsync alignment, slack
  default 2 ms, target-FPS cap, queue depth 4, EMA jitter smoothing
  α=0.125; optional Shizuku side channel supplies pacing diagnostics
  only. Stated cost of the capture route: **+50–80 ms** vs the Linux
  layer (platform constraint on non-rooted Android).
- Failure behaviour: any `initContext` failure → Kotlin falls back to
  **mirror mode** with a status string ("GPU lacks required Vulkan
  ext", `LsfgForegroundService.kt:444-474`). Mirror mode is a *symptom
  surface*, not a diagnosis — our shell must log the failing gate.

## 2.5 Config & lock-ins — the robustness2 workstream (named finding)

**Genealogy.** Upstream lsfg-vk v1.0.0: `VK_EXT_robustness2` ∈
`requiredExtensions` (device.cpp:15-19), `VkPhysicalDeviceRobustness2
FeaturesEXT{nullDescriptor=TRUE}` chained unconditionally (67-73) →
device creation hard-fails without the extension. The **fork**
(lsfg-vk-android `b55b182`) already reworked this:

- `requiredExtensions` (device.cpp:41) = the AHB set only:
  `VK_ANDROID_external_memory_android_hardware_buffer`,
  `VK_KHR_external_memory`, `VK_KHR_sampler_ycbcr_conversion`,
  `VK_KHR_dedicated_allocation`, `VK_KHR_get_memory_requirements2`,
  `VK_KHR_bind_memory2`, `VK_KHR_maintenance1`. Robustness2 is probed
  (device.cpp:149), enabled iff present, feature chain conditional
  (378/394); `nullDescriptorSupported = hasRobustness2` (527).
- **Every null-descriptor dependency** funnels through ONE site:
  `DescriptorSetUpdateBuilder::add(VkDescriptorType type)`
  (descriptorset.cpp:104), reached from the `std::optional<Image>`
  overload (descriptorset.hpp:105) when the optional is empty. If
  `!supportsNullDescriptor()` it binds `device.getFallbackDescriptor
  Image()` — a 1×1 R8G8B8A8 SAMPLED|STORAGE image created at device
  bring-up (device.cpp:534-538). The app session mirrors the pattern
  (android_vk_session.cpp:311-317, log: "using 1x1 fallback descriptor
  image (no functional impact)").
- **All call sites (14), all `VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE`, all
  temporal-feedback inputs:** v3.1 delta.cpp:119,155,168,169,209
  (optImg1/2/3); v3.1 gamma.cpp:86,122 (optImg); v3.1p delta.cpp:
  119,153,166,167,199; v3.1p gamma.cpp:86,120. alpha/beta/generate/
  mipmaps have none. No optional BUFFER or STORAGE bindings exist —
  the fallback image is type-correct everywhere it's used.
- **Flag-guarding:** passes hand the shader a `!optImg.has_value()`
  flag (delta.cpp:109, gamma.cpp:76); on first-pass the binding is
  never semantically read — validity, not content, is what matters.
  Residual nit: the fallback image is never cleared (Image ctor
  records no clear) — content is undefined but unread; Phase 4.3
  adds a zero-clear for strictness.
- Remaining hard gates (unchanged, all probed in device.cpp): the AHB
  set above, timeline semaphores, Vulkan memory model, shaderFloat16
  (with FP32 shader fallback), storage-image trio
  (ExtendedFormats / ReadWithoutFormat / WriteWithoutFormat).
- **Adreno/Turnip-specific assumptions in the app path: none.** "Turnip"
  appears only in comments in the unused Linux layer sources
  (`lsfg-vk-android/src/context.cpp`). The app contains **no driver-
  selection code at all** (grep for VK_ICD / VK_DRIVER_FILES / setenv
  across cpp+Kotlin: zero hits) — it always gets the system default
  driver from the Android loader. *Driver choice is a slot our shell
  must add* (Phase 3).
- DLL version pairing: §2.3 + Crucial #2 (3.2.1.0 = works, FP32-only).

**Phase 2 verdict:** the engine + app architecture is fully mapped;
the only "lock-in" is the stale README claim. Our rework surface is
small and known.

### Phase 2 — Flaws & findings

**FLAWS (LSFG-Android / lsfg-vk codebases):**
- F2.1 *Stale README (misleading):* the app README still says most
  Mali devices lack `VK_EXT_robustness2` and fall back to capture-only
  mirror mode. The fork's code removed that gate (§2.5). The README
  is the project's front door — this stale claim is what made
  robustness2 look like our headline blocker when it isn't.
- F2.2 *Mirror mode swallows the diagnosis:* ANY `initContext`
  failure lands in the same Kotlin fallback with the same string
  ("GPU lacks required Vulkan ext", `LsfgForegroundService.kt:
  444-474`) — a missing AHB extension, an FP16 failure and a
  DEVICE_LOST are indistinguishable to the user. Diagnostics flaw,
  fixed in our shell by gate logging (§4.4).
- F2.3 *Fallback descriptor image is never cleared:* device.cpp:534
  creates the 1×1 fallback via the plain `Image` ctor; no clear is
  recorded (image.cpp). Safe today ONLY because every consumer is
  flag-guarded (§2.5) — a future pass that reads an optional binding
  unguarded would sample undefined memory. Latent correctness trap.
- F2.4 *Hard-coded DLL resource arithmetic:* the loader assumes THS's
  RCDATA layout forever (base table 255–302; FP16 = id+49; FP32 =
  id+98, `android_shader_loader.cpp:44-58`). The SPIR-V magic check
  guards the variant paths, but a future Lossless.dll that renumbers
  the *base* set would silently extract wrong shaders; `probeShaders`
  is the only tripwire. Version-pairing is by convention, not by any
  version check — the loader never reads the DLL's own FileVersion.
- F2.5 *Blunt SPIR-V patch:* `cap_spirv_version` rewrites the SPIR-V
  version word 1.6→1.3 in translated modules without auditing whether
  the module uses >1.3 features (android_shader_loader.cpp:~95). It
  works in practice (Mali accepts the result), but it is a lie told
  to the driver; a shader that genuinely needs 1.4+ semantics would
  fail opaquely.
- F2.6 *Cross-device sync by full idle:* `waitIdle()` bridges the
  app and framegen devices (§2.4) — correct, but it serialises the
  pipeline and is a structural perf cost of the two-device design.
- F2.7 *Root surface in a no-root app:* the release APK bundles
  libsu (`com.topjohnwu`, 887 dex refs) and a `RootCaptureService`.
  Unused on our path; it widens the app's attack/policy surface and
  is easy to miss in a skim of the manifest.
- F2.8 *Upstream repo moved:* PancakeTAS/lsfg-vk GitHub HEAD is a
  migration notice (no code); the last GitHub source is tag v1.0.0.
  Anyone following old links builds nothing. (And upstream v1.0.0's
  framegen still hard-requires robustness2 — the fix exists only in
  the Android fork, F2.1's other half.)
- F2.9 *Vestigial code:* `src/extract/extract.cpp::getDllPath` walks
  Linux Steam paths — dead on Android (the app uses its own loader);
  harmless, but it misleads a first read of "how shaders load".
- F2.10 *Licence trap (the real one):* the umbrella repo presents as
  MIT, and the engine subtree is MIT — but the **application** subtree
  is a custom no-commercial-use licence. Treating "LSFG-Android" as
  uniformly MIT would be a rights error; our shell is therefore
  original code (§4.4), reusing only the MIT engine.
- F2.11 *DukePS3 doc/tree mismatch:* its README documents a DIS
  engine at `rpcs3/Emu/RSX/VK/dis/`, but the shallow clone at
  `81c109e` contains no `dis/` tree (the `lsfg/` tree and
  VKFrameGeneration wiring ARE present and verified). DIS code
  availability is unconfirmed until a full clone is inspected —
  matters only if the fallback engine is ever needed.
- F2.12 *Release/source skew:* the 0.1.2 release APK's
  `liblsfg-android.so` exports only 22 of the 24 NativeBridge JNI
  functions the current source tree defines — `getAverageQueueMs`
  and `getAverageLatencyMs` are missing from the release binary
  (verified by nm diff against our build, which exports all 24).
  Harmless in the release (its Kotlin predates the calls), but it
  proves the shipped APK ≠ HEAD source; anyone diffing behaviour
  against the release should know which tree they're looking at.

**Verification of Syed's BREAKAGE_NOTES (2026-10-30, file:line):**
- F2.13 *Shader-path / memory model — CONFIRMED in mechanism,
  OVERSTATED for Valhall, and the escape hatch is ABSENT on Syed's
  DLL.* The loader's own comments (android_shader_loader.cpp:55-57)
  state the FP32 precompiled set uses "MemoryModel Logical GLSL450
  (no VulkanMemoryModel)" and is "preferred over the DXBC translator
  path on devices without vulkanMemoryModel" — i.e. translated
  output carries VulkanMemoryModel semantics, and three cache sets
  exist (translated root, `fp16/`, `fp32/`, loader lines 368-425,
  all best-effort). BUT selection is **flag-driven** (initContext's
  `performance`/`framegenFp16`, lsfg_jni.cpp:75,106,160), never
  keyed on device.cpp's memory-model probe — the two layers have no
  wire between them (this is exactly Syed's proposed "auto shader
  path" fix). Two corrections to the note as written: (i)
  "Bifrost/Valhall don't support it" — Valhall stock drivers are
  Vulkan 1.3 (memory model is core) and PanVK advertises
  `VK_KHR_vulkan_memory_model` (§1/Crucial-3 binary check), so
  DEVICE_LOST-on-Valhall is UNVERIFIED and likely wrong for stock
  1.3 builds; the exposure is Bifrost-era / cut-down OEM builds.
  (ii) "Lossless.dll already ships precompiled SPIR-V" — true of
  newer DLL builds only; **Syed's 3.2.1.0 has no IDs ≥304**
  (structural verification, §2.3), so on his DLL the precompiled
  escape hatch does not exist until he updates (Crucial #2).
- F2.14 *Storage-format gate — CONFIRMED exactly as claimed.*
  gamma/delta create `R16G16B16A16_SFLOAT` images used as storage
  (v3.1 gamma.cpp:69, delta.cpp:99,102; same in v3.1p). device.cpp:
  302-310 logs a "hard warning" ("first dispatch will likely
  [DEVICE_LOST]") when `shaderStorageImageExtendedFormats` is false
  and **continues anyway**; `vkGetPhysicalDeviceFormatProperties`
  is never called anywhere in framegen or the app cpp (grep: zero
  hits outside thirdparty) — support for the actual format is
  assumed, never queried.
- F2.15 *Capture feedback loop — CONFIRMED mechanism (upstream
  issue #12 as reported).* `OverlayManager.kt:675-731`
  (`installSkipScreenshot`) reaches the overlay's SurfaceControl by
  **reflection** and calls the @hide `SurfaceControl.Transaction.
  setSkipScreenshot` (the `eSkipScreenshot` SurfaceFlinger flag),
  with multiple fallback strategies and log tag `lsfg-vk-loop` —
  the loop is a named, fought problem in this codebase, and the
  reflection approach is inherently OEM-fragile (hidden-API and
  field-layout variance), exactly as claimed. MediaTek-ROM
  specificity is as-reported (not independently reproducible here).
  **Our shell inherits this risk** — see §4.4 mitigation.
- F2.16 *Fallback image layout — CONFIRMED exactly as claimed.*
  `DescriptorSetUpdateBuilder::add(type, image)` writes
  `.imageLayout = VK_IMAGE_LAYOUT_GENERAL` for every image
  (descriptorset.cpp:57-72), including the fallback (routed via
  descriptorset.cpp:106). The fallback `Image` (device.cpp:535) is
  never transitioned by any code (its layout tracker stays at the
  creation layout, UNDEFINED) — descriptor says GENERAL, image is
  UNDEFINED: a validation violation and a strict-driver risk,
  masked in practice by the flag-guarding (F2.3). Syed's one-shot
  UNDEFINED→GENERAL transition fix is the right cheap repair;
  folded into §4.3(d).

**FINDINGS:**
- CONFIRMED: full JNI surface (23 fns, §2.2) identical in source and
  release exports; pass chain + AHB data flow (§2.3/2.4); the
  robustness2 genealogy and the complete 14-site null-descriptor
  enumeration (§2.5); DLL 3.2.1.0 resource structure — 202 RCDATA,
  IDs 101–302, all 48 required DXBC IDs present, no IDs ≥304 (§2.3);
  zero driver-selection code in the app (§2.5); zero Turnip
  dependency in the app path (§2.5).
- SUSPECTED: translated (DXBC-path) shaders perform acceptably on
  the G610 at 2× / flow 0.7 (DukePS3's Mali results suggest yes;
  proven only on-device, P-3).

---

# PHASE 3 — Turnip→PanVK replacement study

## 3.1 Turnip loading today (Adreno side, ecosystem mechanics)
- **AdrenoTools** (bylaws/libadrenotools): a shim library an app links
  that hooks the Vulkan loader's driver-open path (`vk_icdGetInstance
  ProcAddr` / dlopen hooks) and redirects it to a user-supplied Turnip
  build: a zip containing `libvulkan_freedreno.so` + an ICD json +
  metadata. Apps like Winlator, GameNative/GameHub, Dolphin, and
  Skyline-era emulators expose a "custom driver" picker that unzips
  the package into app-private storage and hands its path to the shim.
- Turnip itself is Mesa's freedreno Vulkan driver speaking to the
  **KGSL** kernel driver (`/dev/kgsl-3d0`) that Qualcomm's kernel
  ships — the userspace half is replaceable because KGSL's uAPI is
  stable and the GPU command submission is fully userspace-driven.

## 3.2 Loader mechanics (what decides which .so is the driver)
- Standard Vulkan loader (desktop + Android's libvulkan): ICDs are
  discovered via ICD json manifests (`VK_ICD_FILENAMES`, modern:
  `VK_DRIVER_FILES`, plus `VK_ADD_DRIVER_FILES`), each json naming a
  library path + api version; the loader dlopens the ICD and calls
  `vk_icdGetInstanceProcAddr`. Env vars are read **once, at loader
  init** — so a bundled-driver app must set them before its first
  Vulkan call (native init / JNI_OnLoad time), or bypass discovery
  via an adrenotools-style hook, or dlopen the ICD directly and use
  it as its own loader (the "in-process custom driver" model PanVK's
  Samba S3/PanPlay consumers use).
- Android specifics: the system loader prefers the vendor driver
  from `/vendor/lib64/hw/vulkan.*.so`; app-bundled ICDs are NOT
  auto-discovered — every custom-driver Android app uses one of the
  three mechanisms above. Env vars set from Java/Kotlin via a tiny
  native `setenv` shim before engine init is the least invasive.
- LSFG-Android today: no selection code (§2.5) → system driver only.

## 3.3 Slot mapping (Turnip package → PanVK equivalent)
| Slot | Turnip (Adreno) | PanVK (Mali) |
|---|---|---|
| Driver .so | `libvulkan_freedreno.so` | `libvulkan_panfrost.so` (universal v6–v14 ICD) |
| ICD manifest | json → freedreno .so | json → panfrost .so (same schema) |
| Package | adrenotools zip | `.adpkg` / launcher zip (same shape, §1.5) |
| Kernel device | `/dev/kgsl-3d0` (KGSL) | `/dev/mali0` (vendor kbase) |
| Selection | adrenotools hook or env | env-before-init or direct-ICD (our shim, §3.5) |
| Per-GPU config | Turnip build per Adreno gen | PanVK `profiles/` (arch/gpu_id/uAPI) |
| Fallback | system Qualcomm driver | stock Mali driver (same app, toggle) |

## 3.4 Kernel delta — what cannot be swapped and why
- KGSL and kbase are **different kernel drivers with different uAPIs**;
  nothing below the Vulkan API line is shared. Turnip cannot run on
  Mali and PanVK cannot run on Adreno — the "replacement" is at the
  *packaging/loader slot*, not the driver binary.
- kbase-specific risks with no KGSL analogue: vendor kbase builds vary
  by OEM (uAPI dialect, the `MEM_ALLOC_EX` ENOTTY quirk, §1.2);
  OnePlus's SELinux policy may restrict `/dev/mali0` ioctls from
  unprivileged app domains in ways Qualcomm devices don't restrict
  KGSL (custom-driver loading is a decade-old norm on Adreno; on
  Mali it is new). Sparse memory is a kbase hard NO (§1.4).
- CSF (v10+) vs JM: PanVK must speak CSF to G610 — it does (§1.3);
  there is no JM fallback for this GPU.

## 3.5 Integration blueprint (our app)
- Bundle in the APK (assets or lib/): `libvulkan_panfrost.so` +
  `panvk_icd.json`, extracted to app-private storage on first run
  (ICD json rewritten with the absolute path).
- A ~20-line native shim (`liblsfgmali_shim.so`, ours) exports one
  JNI call: `usePanVk(path)` / `useStock()` → `setenv("VK_DRIVER_
  FILES", …)` / unset, invoked **before** `NativeBridge` engine
  init; a settings toggle + automatic fallback: PanVK init failure
  → log the gate → retry on stock (the §2.4 mirror-mode lesson:
  never fail silently).
- Both LSFG devices (app session + framegen internal) are created
  after the env is set, so one switch drives both.
- AHB caveat (§1.5): if PanVK lacks AHB import, the PanVK mode needs
  the fd-import or copy bridge for capture frames — probe decides
  (test matrix P-2). Stock mode is unaffected.
- **Test matrix:** {PanVK, stock} × {probe gates pass, LSFG init,
  first generated frame, sustained 2× session} on the 10R (§4.5).

**Phase 3 verdict:** the Turnip slot is a packaging + loader-env
slot; PanVK fills it file-for-file. The analogy breaks only below
the Vulkan line (kernel uAPI), which is PanVK's whole reason to
exist. No licence or loader wall found.

### Phase 3 — Flaws & findings

**FLAWS (in the analogy / in assumptions a porter might bring):**
- F3.1 *The slot doesn't exist yet in our base app:* LSFG-Android
  has NO driver-selection code (§2.5) — "swap Turnip for PanVK"
  understates the work: the selection mechanism (env shim + toggle
  + fallback, §3.5) must be built, not reused.
- F3.2 *KGSL monoculture bias:* Adreno custom-driver culture rests
  on one stable kernel uAPI across a decade of devices. kbase is
  OEM-variant (F1.4) — a porter assuming Turnip-grade uniformity
  will under-budget per-device validation.
- F3.3 *Env-var ICD selection is loader-version-sensitive:* Android's
  libvulkan honours `VK_DRIVER_FILES`/`VK_ICD_FILENAMES` for
  debuggable contexts, but behaviour varies across Android/loader
  versions; the robust route is the direct-ICD (own-loader) model
  PanVK's own consumers use. Our blueprint keeps env-first for
  simplicity but must be prepared to fall back (§3.5) — flagged as
  the Phase 4 integration risk most likely to bite.
- F3.4 *`.adpkg` ≠ drop-in:* PanVK's packages target launchers that
  already implement the adrenotools contract; a standalone app must
  re-implement the extraction/path-rewrite step itself (§3.5) —
  small, but not zero, and easy to overlook.

**FINDINGS:**
- CONFIRMED: the replacement is file-for-file at the packaging
  layer (§3.3 table); both ecosystems converge on "ICD .so + json +
  app-private extraction"; PanVK ships the exact package shapes
  (§1.5); nothing below the Vulkan line is swappable (§3.4).
- SUSPECTED: env-before-init selection will work on the 10R's
  Android build for our own debuggable APK (standard behaviour;
  verified only on-device, P-1).

---

# PHASE 4 — Our version (LSFG-for-Mali, PanVK-led)

## 4.1 Final plan (numbered; each step has its verification)
1. **P-0 Capability probe APK** (stock + PanVK columns): dumps Vulkan
   version, device/extension list, and the §2.5 gate list on the 10R.
   *Verify: log returned by Syed; gates table filled for both drivers.*
2. **Driver bundle** (§4.2): PanVK ICD from the project's release
   artifacts (or a source build if no suitable prebuilt), packaged
   per §3.5. *Verify: ICD enumerates `Mali-G610` under PanProbe or
   our probe on-device.*
3. **Engine** (§4.3): existing arm64 build + robustness2 hardening.
   *Verify: `LSFG_3_1::initialize` + `Generate()` succeed in a
   harness run with `VK_EXT_robustness2` absent (PanVK column of
   P-0 makes this a real condition, whichever way §1/Crucial-3 lands).*
4. **Shell** (§4.4): original framework-only Kotlin shell around
   `NativeBridge`; MediaProjection → engine → overlay; 2× default,
   flow-scale setting; driver toggle per §3.5; manual-pipeline APK.
   *Verify: badging/signature/contents checks + DLL-absence check
   (no Lossless.dll or .spv in the APK, ever).*
5. **Playtest** (§4.5): Syed runs the protocol on the 10R; logs
   decide stock-vs-PanVK default and the FP16/DLL-update call.

## 4.2 Driver bundle — PanVK ICD packaged into the app
- **DONE (artefact):** the project's own RC4 release
  (`panvk-kbase-v0.1.0-rc4`, "v7-v13", built from the pinned Mesa
  `5a07217f`) ships the bare ICD + json — downloaded to `panvk-bundle/`
  and **hash-verified against the release's own SHA256SUMS**:
  `libvulkan_panfrost-android-aarch64.so` = 23,257,872 B, sha256
  `977c0c4c0803b9940c453d691f50f14c9f8f732fe945d93bdde637880befd791`;
  `panfrost_icd.aarch64.json` = 187 B, sha256 `1c27f9363261d7080491b4
  c72ade8805f3f22ab6932123256515d2cf76580ff2` (api_version 1.4.363 —
  matches §1.4). The stock json's `library_path` is a glibc-style
  `/usr/local/lib64/...` path → our packaging rewrites it to the
  extracted absolute path at first run (§3.5). ICD exports verified:
  `vk_icdGetInstanceProcAddr`, `vk_icdGetPhysicalDeviceProcAddr`,
  `vk_icdNegotiateLoaderICDInterfaceVersion`. No source build needed.
- Packaging per §3.5 (assets → private storage, json path rewrite,
  setenv shim, toggle + auto-fallback) — implemented in §4.4.
- **Status: artefact done; in-APK packaging lands with §4.4.**

## 4.3 Engine build — the robustness2-free rework
- **DONE (build):** `liblsfg-android.so` for arm64-v8a/android-29
  built from the fork with NDK r26d (project pins r27; r26d worked):
  configure with the app cpp CMakeLists + `android.toolchain.cmake`,
  `cmake --build` → 100%. Outputs: `build-arm64/liblsfg-android.so`
  (18,187,408 B, BuildID `9b69157c…`) + `liblsfg-vk-framegen.a`
  (22,428,442 B). **One patch — PORTING PATCH #1:** the tree's only
  two `std::format` calls (`framegen/src/common/exception.cpp`)
  replaced with string concatenation: NDK r26d's libc++ exposes
  `std::format` in headers only under `_LIBCPP_ENABLE_EXPERIMENTAL`
  and its prebuilt lacks the `format_error` vtable at link time.
- **Robustness2-free status — DEMOTED to insurance (research
  sweep integration):** PanVK exposes robustness2 on v10+ (Crucial
  #3 evidence chain), so on the primary driver path there is
  nothing to work around. The fork core's robustness2-optional
  state (§2.5 — requirement removed, 1×1 fallback at the single
  descriptor site, all 14 call sites SAMPLED_IMAGE + flag-guarded)
  remains valuable **only as insurance for the stock-driver
  fallback path**, where the extension genuinely may be absent.
  Remaining rework items, in order (insurance-scoped):
  Also recorded from the sweep (B2 closure): **SPIR-V memory-model
  stripping has NO proven tooling** — no spirv-opt pass rewrites
  `OpMemoryModel`; the models are not freely interchangeable
  (WORKAROUNDS.md §2.3). Do not plan on it. The covers are
  precompiled-SPIR-V forcing (§4.3(f)) and PanVK's memory-model
  support (core 1.2+, advertised in the bundled ICD). If a real
  DEVICE_LOST appears on-device, the diagnostic to wire is
  **`VK_EXT_device_fault`** (+ address binding report) into the
  crash reporting — Arm's own recommended practice for Mali.
  (a) zero-clear the fallback image at creation (strictness);
  (b) add an init log line naming `nullDescriptorSupported` and the
  fallback state so field logs prove which mode ran;
  (c) on-device proof via P-0/§4.5: initialize + generate with the
  extension absent. *Success criterion (Syed's): LSFG_3_1::initialize
  runs and Generate() produces frames with VK_EXT_robustness2
  entirely absent.*
  (d) **Fallback layout fix (from F2.16):** one-shot UNDEFINED →
  GENERAL transition (+ zero-clear from (a)) when the fallback
  image is created in device.cpp.
  (e) **Format gate (from F2.14):** query
  `vkGetPhysicalDeviceFormatProperties` for STORAGE_IMAGE on
  `R16G16B16A16_SFLOAT` (and the R8-class fallback) at device
  bring-up; log + select explicitly instead of warning-and-pray.
  Full format substitution (patching the OpTypeImage operand at
  extraction, per Syed's fix list) is the follow-up if the gate
  ever fails on a target driver.
  (f) **Auto shader path (from F2.13):** key shader-set selection
  on the device probe — no vulkanMemoryModel ⇒ precompiled FP32
  SPIR-V set; shaderFloat16 + FP16 set present ⇒ FP16; never
  silently land on translated shaders on a no-memory-model
  device. *Depends on the DLL update (Crucial #2): with 3.2.1.0
  there are no precompiled sets to select.*
- JNI parity: **verified** — our build exports all 24 NativeBridge
  JNI functions; the 0.1.2 release .so exports 22 (F2.12). Our .so
  matches the current source contract exactly.

## 4.4 Shell & build
- **Shell (ours, original code, framework-only Android — no AndroidX,
  so the manual pipeline works):** package `com.syed.lsfgmali`;
  `com.lsfg.android.session.NativeBridge` class declared verbatim
  (JNI is name-matched) with the 23 functions of §2.2.
  - MainActivity: DLL picker (SAF, persistable grant) → copy to
    private storage → `extractShaders` → driver toggle (PanVK/
    stock, §3.5) → multiplier fixed 2× + flow-scale slider
    (0.5–0.75 band, default 0.7) → Start. **Mali tuning defaults
    (Syed's fix list): performance variant 3.1P ON, flow 0.7, ×2.**
  - CaptureService (foreground, mediaProjection type):
    MediaProjection consent → VirtualDisplay/ImageReader →
    `pushFrame(AHB)`; overlay via SYSTEM_ALERT_WINDOW (accessibility
    overlay variant kept as a setting for OnePlus touch filtering);
    `setOutputSurface` from the overlay's SurfaceView.
  - **Feedback-loop mitigation (F2.15):** the overlay attempts the
    skip-screenshot exclusion (our own implementation of the
    SurfaceControl reflection technique, fully guarded — failure
    just logs); v1 also offers a reduced capture-resolution option
    (0.75×) per the tuning notes.
  - **Capture plan — REWORKED (research sweep, WORKAROUNDS.md §4):**
    correction of record first: **upstream LSFG-Android's Shizuku
    use is ONLY a timing side channel, NOT a capture path** (its
    README: Shizuku buffers never feed the visible video path; our
    §2.4 said the same). The shipped v1 shell uses full-display
    MediaProjection + skip-screenshot attempt. The v1.1 capture
    ladder, in priority order:
    (1) **PRIMARY — Android 14+ single-app projection:** scope the
    session to the game's window (`createConfigForUserSelected
    Display`-style consent); the overlay is not part of that
    window, so the feedback loop *cannot form*. Platform-proven;
    costs a per-session app pick in the consent dialog. The 10R's
    current OxygenOS/Android build decides availability — Syed's
    first run tells us.
    (2) **ADVANCED — Shizuku-privileged layer-excluded capture:**
    hidden ScreenCapture/SurfaceControl APIs (scrcpy-class
    machinery) capturing the display while excluding our overlay
    layer by construction. Mechanism proven in the wild; as an
    FG video path it is a **novel assembly, ours to build** —
    nobody has shipped it for LSFG.
    (3) **LAST RESORT — setSkipScreenshot flagging** (what v1
    ships): reflection on @hide APIs, OEM-fragile (F2.15), paired
    with luma-trend loop detection to at least *know* it failed.
    (FLAG_SECURE content stays black under every method — OS
    enforced, out of scope by design.)
  - **Polish scope (from MEDIATEK_FG_STUDY.md):** MediaTek's own
    FG (MFRC) does not exist on the 8100 and is not callable
    anywhere (per-game OEM enablement, no API/SDK — verified by
    the study across MediaTek/MAGT/NeuroPilot material; the one
    8100 phone with interpolation, Realme GT Neo 3, used a
    dedicated display chip the 10R lacks). What IS borrowable:
    **ADPF** — public Android API, works on the 8100:
    PerformanceHintManager hint sessions + thermal headroom
    driving our existing knobs (flow scale, multiplier,
    capture-res shedding) before hard throttling (the MAGT playbook:
    jitter −17%, power −6% in their case study). Also noted: the
    half-rate-render + interpolate power contract (§5.1 of the
    study) as a per-game profile idea, and DIS as a peer engine,
    not a poor relation (§5.2). ADPF integration = post-v1 polish,
    not a v1 dependency.
  - Gate logging: init rc + per-gate results surfaced in the UI log
    view (never silent mirror-mode, §2.4).
  - **App requirement — AUTOMATIC logcat capture (Syed, corrected
    version; supersedes any toggle wording):** log capture is FULLY
    AUTOMATIC — it starts on its own at every app start and every
    framegen session start; no setting, no user action. Design
    constraints honoured: without READ_LOGS/root an app can only
    capture its OWN process logs (`logcat --pid=<own pid>` stream +
    the shell's LogBus ring) — acceptable because every diagnostic
    the phases rely on (the `lsfg-vk-framegen` "Device features
    probe:" line, session/render logs) is emitted by our process
    under greppable tags. Files are timestamped per session
    (`lsfgmali-<yyyyMMdd-HHmmss>.log`, app logs dir), capped
    (~8 MB per part with rotation, max 5 files kept) so capture
    can't eat storage, and the feedback loop is file-back: a
    Share/export action copies the newest log to public Downloads
    and opens the share sheet. Implemented in `LogSaver.kt`;
    started from `MainActivity.onCreate` and
    `CaptureService.onCreate` (idempotent).
- **Build:** manual pipeline (aapt2 → kotlinc → d8 → zipalign →
  apksigner, per `~/workspace/ruby-bounce/build-apk.sh`); APK bundles
  our stripped `liblsfg-android.so`, NDK `libc++_shared.so`, the
  PanVK bundle (§4.2), and the setenv shim. Output:
  `~/workspace/lsfg-mediatek/LSFGMali.apk`.
- **Status: DONE (2026-10-30).** Shell built first-try with the
  manual pipeline (`app-shell/build-apk.sh`). **`LSFGMali.apk` =
  7,623,567 B, sha256 `6eb2df0e872749657490b37ab39923ced86465438
  128e5b36572b758b10ae705`**, debug-signed (cert sha256
  `4dff7661…`), package `com.syed.lsfgmali` v0.1, minSdk 29 /
  targetSdk 35. Verified contents: classes.dex (2,328,048 B) with
  NativeBridge + all shell classes; `lib/arm64-v8a/` =
  liblsfg-android.so (2,130,520 B, stripped), libvulkan_panfrost.so
  (23,257,872 B — **byte-identical to the hash-verified RC4
  bundle**, sha256 re-checked inside the APK), libc++_shared.so,
  liblsfgmali_shim.so (5,872 B, both JNI exports present).
  **DLL-absence check: zero files matching lossless/.spv/dxbc in
  the APK.** Automatic logcat capture (§4.4 requirement) is in
  (`LogSaver.kt`, started from activity + service, no toggle).
  Remaining Phase 4 work is entirely Syed-side (§4.5 + RUN_SHEET).

## 4.5 Playtest package
- **Division of labour (Syed, 2026-10-30): ALL device-side work is
  Syed's.** Nothing in any phase depends on us running anything on
  the 10R — no ADB, no assumed device access. Our side ends at a
  **verified APK + run sheet**. Every step below is executed by
  Syed, when he chooses; the baseline features probe (logcat tag
  `lsfg-vk-framegen`, the "Device features probe:" line) is his
  first step.
- Deliverables: `LSFGMali.apk` (verified: badging/signature/
  contents, DLL-absence check) + **`RUN_SHEET.md`** (what to
  install, what to run, exactly which log lines/results to send
  back) + this notes file.
- **Test protocol (Syed-side, in order):**
  P-1 Probe: run probe mode on stock, then PanVK — return both
  logs, starting with the **Device features probe line** (settles
  §1.2 kbase/SELinux, §1.5 AHB + MC6 variant, Crucial #3 exposure).
  P-2 First frame: pick his Lossless.dll, start capture on a game
  or video, confirm generated frames counter increments at 2×.
  P-3 Sustained: 5-minute session; watch for DEVICE_LOST, thermal
  throttle, pacing jitter; collect the in-app gate log.
  P-4 A/B: same scene on stock vs PanVK; compare counters + feel.
- **Known limits going in:** +50–80 ms capture latency (platform);
  FP32-only shaders until the DLL is updated (Crucial #2); mirror
  mode = failure surface, always send the gate log; PanVK is months
  old — expect rough edges stock doesn't have; debug-signed,
  sideload-only, personal use (licences, Crucial #4).

### Phase 4 — Flaws & findings (living; build in progress)

**FLAWS:**
- F4.1 *Fork code isn't NDK-portable as-is:* `std::format` in
  `framegen/src/common/exception.cpp` fails to LINK on NDK r26d
  (header-visible under `_LIBCPP_ENABLE_EXPERIMENTAL`, symbols
  absent from the prebuilt libc++). Fixed by PORTING PATCH #1
  (§4.3) — but it means "the fork builds" is NDK-version-sensitive;
  r27 (the pinned version) is the upstream-tested path.
- F4.2 *Our engine .so ships fat if unstripped:* 18,187,408 B with
  debug info vs the release build's 1,119,376 B. Stripping is a
  packaging step, not automatic — recorded so it isn't forgotten
  in §4.4.
- F4.3 *Shell is a deliberate subset:* no Gradle in this sandbox →
  the upstream Compose app cannot be built here at all; our
  framework-only shell omits Shizuku pacing telemetry, NNAPI post
  stages and root capture (all optional upstream features, §2.1/
  2.2). Scope cut, documented — not an upstream flaw.
- F4.4 *Two-device design inherited:* the shell keeps upstream's
  session+framegen two-device architecture with its `waitIdle`
  bridge (F2.6) — the latency cost is structural, not tunable away
  in this phase.
- F4.5 *Shell had zero rotation handling — landscape games shown
  sideways (v0.1, found by Syed's playtest 2026-10-30):* the
  capture geometry was frozen once at session start from the
  natural PORTRAIT metrics and never revisited.
  Root cause, all in `app-shell/.../CaptureService.kt` (v0.1):
  size computed once from `wm.defaultDisplay.getRealMetrics()`
  (lines 139–148 — 1080x2412 on the 10R); the output surface was
  bound at those frozen dims (`setOutputSurface(holder.surface,
  cw, ch)`, line 166); `surfaceChanged` was an empty no-op
  (line 172); `ImageReader.newInstance(cw, ch, …)` (line 250) and
  `createVirtualDisplay(…, cw, ch, …, AUTO_MIRROR, …)` (lines
  266–269) inherited the same portrait dims; no DisplayListener /
  orientation callback existed anywhere in the shell. When a
  game rotates the display to landscape, the AUTO_MIRROR virtual
  display squeezes the landscape scene into the portrait buffer
  and the overlay presents it unrotated — Syed's screenshots show
  exactly this (landscape scene sideways in a narrow portrait
  strip, black bars, in two different games).
  FIX (v0.2, same file, shell-only — engine untouched):
  (a) geometry is now computed from the display's CURRENT
  rotation: natural size normalised by min/max (portrait-natural
  panel), axes swapped for ROTATION_90/270, session scale +
  even-rounding applied as before (`computeCaptureGeometry`);
  (b) mid-session rotation is handled by a
  `DisplayManager.DisplayListener` on the default display plus a
  `surfaceChanged` orientation-disagreement safety net, both
  funnelling into one size-checked `onRotationPossiblyChanged`;
  (c) `reconfigure()` resizes the whole chain on the main thread:
  reader detached/closed → engine context destroyed and
  re-initialised at the new size via the factored `initEngine()`
  (the shell's existing init/destroy pattern — engine input
  images are sized at initContext) → output surface re-attached
  → `VirtualDisplay.resize()` + fresh ImageReader bound;
  (d) logging: session start logs rotation + natural size +
  capture size, every rotation event logs old→new rotation and
  sizes, surface changes log overlay size vs capture size — the
  next on-device log proves which path ran.
  NOT device-verified (no device here): the reconfigure path,
  AUTO_MIRROR behaviour across a VD resize on the OnePlus ROM,
  and whether the engine's re-init mid-session is hitless are
  Syed-side checks (P-2 with a landscape game).
- F4.6 *Share/Export exported the wrong log (v0.1, found
  2026-10-30):* `LogSaver.shareLatest()` sent `latestFile()` —
  newest by mtime (`LogSaver.kt` v0.1, `latestFile()` lines
  139–141). The current session's file is appended continuously,
  so after any relaunch "newest" is the fresh launch stub
  (~1 s of log), while the real game-session files sit in the
  private `files/logs/` folder, unreachable without root. Syed
  sent two launch-only exports before this was spotted.
  FIX (v0.2, `LogSaver.kt`): export selection is session-aware —
  `exportFile()` scans `lsfgmali-*.log` newest-first and picks
  the first file containing a session marker (`"capture size "`
  or `"capture running"` — both emitted by CaptureService at
  session start in v0.1 AND v0.2, so surviving v0.1 session logs
  become exportable after the v0.2 upgrade, which preserves app
  data: same package + same signing key). Fallback to newest
  overall only when no session file exists. The in-app log names
  the exported file (and the skipped launch stub, when skipped).
  STRANDED-LOG SURVIVAL (cap analysis, from code): retention is
  `MAX_FILES = 5` files, enforced by `prune()` at every
  `LogSaver.start()` — i.e. once per app process launch — keeping
  the 5 newest by mtime; 8 MB split parts count as separate
  files toward the 5. Files from Syed's 2026-10-30 ~03:2x–03:4x
  testing therefore survive ONLY if still among the 5 newest at
  his last launch: with the 02:50 session plus several 03:xx
  launches/sessions on record, the earliest game-session files
  may already be pruned; the later ones (03:42 onward) very
  likely remain. Which files remain is decidable on-device only —
  v0.2's exporter will name its pick in the app log.
- F4.7 *Overlay presentation unstable after rotation — stale latch
  AND window-geometry flip-back (v0.2, found by Syed's playtest
  2026-10-30, session log `lsfgmali-20261030-035755.log`):* the
  v0.2 rotation mechanism itself worked — session start portrait
  1080x2412, rotation detected 03:59:05, engine re-initialised at
  2412x1080, then generated 0→212 / posted 424 / unique-captures
  611 over ~20 s of landscape play. The ENGINE never flipped back.
  The presenting window did. Syed's eyewitness report, verbatim in
  substance: "The screen got fixed for a split second, then it
  went back to portrait — not one frame, the WHOLE screen shifted
  portrait, now covering the whole screen." Two root causes, both
  in `CaptureService.kt` (v0.2):
  (a) STALE LATCH: `reconfigure()` re-attached the SAME
  SurfaceView's surface (`output surface re-attached at 2412x1080
  (overlay view 2316x1080)`). A SurfaceView's last queued buffer
  stays latched in composition until a new buffer replaces it —
  across the engine destroy/re-init gap the pre-rotation portrait
  frame kept presenting alongside/after the new landscape output
  (Syed's screenshots show the ghost portrait copy next to the
  landscape one).
  (b) WINDOW GEOMETRY: the overlay window was MATCH_PARENT and the
  ROM shrank it by the display-cutout/system insets — overlay view
  2316x1080 vs capture 2412x1080 (96 px lost), later 2316x1032 and
  1080x2268 — and with nothing pinning it, the window re-derived
  portrait geometry from the portrait activity configuration
  mid-session and stretched the (correct) landscape engine output
  fullscreen-portrait: the flip-back Syed saw. Contributing
  suspicion, recorded as such: the overlay's surfaceChanged safety
  net treated size-only jitter (`2316x1080` → `2316x1032`,
  03:59:39) as orientation signal, and a focusable overlay can
  steal foreground status from the game, either of which can feed
  the window instability.
  FIX (v0.3, same file, shell-only): (1) the overlay SurfaceView
  is DESTROYED (output detached + removeViewImmediate) at the
  START of every reconfigure and RECREATED fresh at the new
  geometry — first presented frame afterwards is a fresh engine
  frame, never the pre-rotation buffer; (2) the overlay window is
  PINNED: explicit LayoutParams width/height == capture size for
  the active rotation, gravity top-left 0,0,
  `layoutInDisplayCutoutMode` ALWAYS (API 30+) / SHORT_EDGES —
  the window covers the full display, cutout included, and never
  re-derives size from the activity configuration; (3) the pin is
  RE-ASSERTED (`updateViewLayout`) inside surfaceCreated (atomic
  with every surface re-attach) and on every surfaceChanged (any
  subsequent layout pass), and the window params (w/h/rotation)
  are logged at every change so the next on-device log shows
  whether the window or the surface moved if this persists;
  (4) surfaceChanged is DEBOUNCED: only a change in the display's
  ROTATION VALUE triggers reconfigure consideration — size-only
  changes are logged (`surface size-only change ignored`) and
  ignored; (5) an explicit size check is logged on every surface
  change (`overlay size check: OK / MISMATCH — surface … vs
  capture …`); (6) the overlay is non-responsive to touch for the
  whole session — FLAG_NOT_TOUCHABLE + FLAG_NOT_FOCUSABLE in the
  pinned params, logged once per attach (`overlay flags: …`) —
  touches/focus pass through to the game, and the focus-stealing
  contributor above is closed with it.
  NOT device-verified (no device here): whether the OnePlus ROM
  honours the cutout mode + explicit size on a TYPE_APPLICATION_
  OVERLAY window (the size-check log lines will prove it either
  way), and whether the flip-back is fully gone — Syed's retest.
- F4.8 *Overlay outlives its content — last frame sticks over all
  other apps (v0.2, flagged by Syed 2026-10-30):* when the game is
  closed/backgrounded, capture starves but nothing hid the
  overlay: in the 035755 log the counters freeze at
  generated=212 / posted=424 / unique=611 from 03:59:28 and stay
  frozen while the overlay keeps presenting the last frame,
  blocking visibility over every other app, indefinitely. Same
  gap on the exit paths: v0.2's `teardown()` released the VD and
  destroyed the engine context BEFORE removing the overlay view.
  Also: the overlay presented while our own MainActivity was
  foreground, mirroring the app's own UI into the output stack
  (visible ghosted in Syed's screenshots).
  FIX (v0.3): (1) frame-starvation watchdog (1 s tick): if the
  engine's unique-capture count does not change for ~2 s, the
  overlay is destroyed and `overlay hidden: no new captures …` is
  logged; when captures resume it is recreated (`captures resumed
  …`) — counters resetting across a reconfigure count as life and
  re-arm a grace window; (2) own-app suppression: MainActivity
  publishes its foreground state (`ForegroundState`,
  onResume/onPause — same process), the watchdog follows it, and
  the overlay is held from session start while our UI is up
  (`overlay held: own app in foreground`); (3) one ordered,
  idempotent teardown shared by STOP, MediaProjection.Callback.
  onStop and onDestroy: overlay removed → virtual display
  released → engine context destroyed, each step logged
  (`teardown complete: overlay removed -> virtual display
  released -> engine stopped`). Visibility is centralised in
  `refreshOverlayVisibility()` — the overlay exists only while a
  session is active AND own-app is backgrounded AND captures are
  not starved.
  NOT device-verified: watchdog behaviour against a genuinely
  static game screen (a paused/static game produces no new
  captures — the overlay will hide after ~2 s and the raw game
  shows through, which is the intended trade; Syed's feel check),
  and the ≤1 s latency of the own-app suppression transitions.
- F4.9 *Portrait-born VirtualDisplay presents sideways after a
  landscape reconfigure — resize() does not re-derive the
  AUTO_MIRROR orientation transform (v0.3, found by Syed's
  playtest 2026-10-30, session log
  `lsfgmali-20261030-041623.log` + screenshots ~04:14–04:17 IST):*
  v0.3's F4.7/F4.8 fixes CONFIRMED on-device (watchdog hides the
  overlay on starvation — closing the game no longer leaves a
  frame over other apps; own-app suppression works, log lines
  04:17:01; touch-through flags on; overlay pinned). And a
  session STARTED in landscape (rotation=90 at session start)
  was fully clean: overlay pinned 2412x1080,
  `overlay size check: OK` from the first frame, 200 frames
  generated, zero geometry problems — a VirtualDisplay CREATED
  at landscape geometry mirrors perfectly on this ROM. But the
  portrait-start → rotate-to-landscape session still presented
  the game rotated 90° (sideways, full-frame) even though every
  logged size reported landscape. Syed's verdict: "portrait
  problem partially fixed."
  SUSPECTED root cause (not proven): `reconfigure()` called
  `virtualDisplay.resize()` on a VirtualDisplay CREATED in
  portrait. The AUTO_MIRROR orientation transform is established
  at VD creation; on this ROM `resize()` apparently does not
  re-derive it, so mirrored content keeps arriving pre-rotated
  into the (correctly landscape-sized) engine while all
  shell-side sizes say 2412x1080. Evidence split: born-landscape
  VD → perfect mirror; resized VD → sideways picture at
  correct sizes. The overlay and engine paths are effectively
  exonerated by the clean landscape-start session — same code,
  same pin, same engine; only the VD's birth geometry differs.
  CORROBORATION (Syed, same playtest): opening LSFG's
  MainActivity as a floating/multi window over the sideways
  game made the picture CORRECT; closing the window brought
  the sideways picture back. This is own-app suppression, not
  an orientation fix: MainActivity foreground destroys the
  overlay (F4.8), so the RAW game shows through — always
  correct — and closing it recreates the overlay presenting
  the engine's sideways output again. It rules out the phone's
  rotation state and the game itself: with our overlay gone
  the game renders upright on the same display in the same
  moment, so the defect lives inside the frames WE present in
  the rotate path (the captured/engine input), upstream of the
  overlay — consistent with the VD-transform suspicion and
  with the VD-recreate fix.
  FIX (v0.4, `CaptureService.kt` reconfigure() only): NEVER
  resize the existing VD — release it and CREATE a fresh
  VirtualDisplay at the new geometry with the fresh reader's
  surface, using the exact session-start `createVirtualDisplay`
  call (same name, AUTO_MIRROR flag, density). Logged
  distinctly: `virtual display recreated at WxH (rotation
  reconfigure — never resized)`; session start now logs
  `virtual display created at WxH` too, so the next device log
  proves which path ran. If creation returns null or throws
  mid-reconfigure, the session survives via the old `resize()`
  path as a logged fallback — recreate is the primary path.
  NOT device-verified: whether the recreated VD actually cures
  the sideways presentation — Syed's v0.4 retest is the verdict
  (portrait start → START → rotate into the game → play
  20–30 s → STOP → export immediately). If the picture is
  still sideways with `virtual display recreated` in the log,
  the transform is established somewhere other than VD
  creation and this diagnosis reopens.

- F4.10 *ROOT CAUSE of the sideways landscape presentation —
  the swapchain declares the surface's currentTransform as its
  preTransform while rendering un-rotated content (v0.4 retest
  2026-10-30, session log `lsfgmali-20261030-043057.log` +
  screenshots 04:32:09/12 IST; full native-path inspection
  same day):* the v0.4 session STARTED in landscape
  (rotation=90 at session start), its VirtualDisplay was
  CREATED at 2412x1080, the overlay reported
  `overlay size check: OK`, frames generated (156 by 04:32:10)
  — and the picture was STILL rotated 90°. **F4.9 is
  therefore RETIRED as disproven**: VD creation geometry,
  VD resize, window geometry (F4.7) and surface freshness are
  all exonerated; the rotation is introduced at PRESENT time,
  downstream of the engine, in every configuration. Syed's
  framing is exact: the pipeline behaves as if the display
  were still portrait while it is physically landscape, even
  though every size reports 2412x1080.
  THE MECHANISM (proven by code inspection; the one missing
  device value — `caps.currentTransform` — is logged by v0.5
  to close the proof): the present path is the Vulkan WSI
  swapchain (`lsfg_render_loop.cpp` `blitOutputToSwapchain`,
  1:1 `vkCmdCopyImage` of the framegen output, device log
  04:31:59 "Swapchain ready: 2412x1080 … path=WSI"). At
  creation, `createSwapchain()` copied
  `caps.currentTransform` into
  `VkSwapchainCreateInfoKHR.preTransform` (formerly line 735)
  while the content it presents is rendered axis-aligned in
  window coordinates — the overlay window is created at the
  LIVE rotation's geometry (2412x1080 landscape), so the
  frames are already upright in display space. Declaring
  `preTransform = currentTransform` tells the presentation
  engine "this content has already been rotated by that
  amount". On a WSI that honours the declaration, the
  compositor applies that rotation to the buffers at
  composition — rotating already-upright landscape frames
  90°, full-frame, at correct sizes. That is precisely the
  observed symptom, and it explains every established fact:
  portrait is correct (transform is IDENTITY in portrait);
  session-start orientation and VD birth/resize history are
  irrelevant (the tag is applied per-present); all logged
  sizes agree everywhere (a composition-time rotation tag is
  not a geometry change); hiding the overlay shows the raw
  game upright (the game's own layers are not mis-tagged).
  WHY PANVK/OUR SHELL AND NOT UPSTREAM: (a) the value comes
  from SurfaceFlinger's BufferQueue TRANSFORM HINT for the
  layer — ROTATE_90 for a landscape window on a
  portrait-natural panel — which Mesa's Android WSI (PanVK)
  reports as `caps.currentTransform`; (b) our shell presents
  into a **SurfaceView**, whose BufferQueue consumer IS
  SurfaceFlinger, so the hint is set. Upstream never hits it
  for two independent reasons: its render loop's normal path
  is the CPU blit (`OverlayManager.kt` header: "the native
  render loop already CPU-blits each generated frame") and
  the CPU path NEVER tags buffers with a transform
  (`blitOutputToWindow` sets geometry only); and upstream
  presents into a **TextureView**, whose SurfaceTexture
  consumer publishes no transform hint (currentTransform
  IDENTITY) — plus stock Adreno WSI reports IDENTITY
  regardless. The bug needs exactly our combination:
  SurfaceView + WSI present + a hint-honouring WSI (Mesa).
  THE NATURAL-ORIENTATION HUNT (Syed's framing — anything
  keying off the panel's natural orientation instead of the
  live rotation): (1) VD creation: our shell passes
  `VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR` where upstream
  CaptureEngine uses flags=0 with an explicit warning
  comment; noted, but exonerated as the rotation source by
  the born-landscape v0.4 failure — kept as candidate #2.
  (2) AHB path: `pushFrame`/`AHardwareBuffer_describe`
  carries NO orientation field at all (`AHardwareBuffer_Desc`
  = width/height/layers/format/usage/stride; device log:
  input 2412x1080 stride 2416 fmt=1 usage=0x303) — nothing
  on the input side reads or could apply a transform; the
  final blit is a plain 1:1 copy, no rotation code exists
  anywhere in the native path (grep: no rotation/transform
  handling outside the swapchain create). (3)
  ANativeWindow geometry: the CPU path sets buffers geometry
  to the output dims; the WSI path takes
  `caps.currentExtent` (2412x1080 on-device) — both live-
  rotation values. (4) The swapchain preTransform was the
  ONE place a natural-orientation-derived value (the panel's
  portrait→landscape rotation) entered the pipeline — and
  it was copied blindly.
  RESEARCH CORROBORATION (parallel sourced web research,
  `~/workspace/research_notes/android-mediaprojection-
  vulkan-overlay-r-20261029-2307/report.md`): the model is
  that "portrait" on this panel is not a size but the
  default/identity orientation state — landscape must be
  explicitly negotiated at TWO hand-offs (capture ingestion
  and Vulkan present); in portrait both are IDENTITY so the
  pipeline works by accident, and recreating the VD
  changing nothing points AWAY from the VD and toward the
  output surface/swapchain. Loader mechanics (AOSP
  `vulkan/libvulkan/swapchain.cpp`): `currentTransform` is
  manufactured by the platform Vulkan loader from the
  window's `NATIVE_WINDOW_TRANSFORM_HINT` (computed per
  window as rotationToBufferTransform((installOrientation +
  displayRotation) % 4) → IDENTITY in portrait, ROTATE_90
  in landscape on this panel), and at swapchain creation the
  loader queues buffers with
  `set_buffers_transform(InvertTransform(preTransform))` —
  i.e. it instructs the compositor to perform the INVERSE
  of the app's declared transform so the two cancel when
  the app truly pre-rotated. Declaring
  `preTransform = currentTransform` WITHOUT rotating the
  content is a false assertion and yields a picture rotated
  by exactly that transform; IDENTITY + unrotated content
  is the documented correct-but-slower configuration
  (compositor rotates). That is word-for-word this build's
  defect and fix. UPSTREAM ISSUE #111
  (github.com/FrankBarretta/LSFG-Android/issues/111, same
  app, v0.1.2, portrait-native panel + landscape game)
  independently diagnoses "missing Vulkan pre-rotation
  handling" in this engine — RECONCILIATION: #111 claims
  0.1.2 "never queries currentTransform", but that does not
  describe OUR shipped binary: our `liblsfg-android.so` is
  built in-tree from the vendored source (build-arm64;
  stripped-copy hash chain verified), and that source DOES
  query the caps and DID copy `currentTransform` →
  `preTransform` (`createSwapchain`, pre-fix line 735) with
  an unrotated 1:1 copy as the final present — it simply
  never LOGGED the value (nor did our device logs, until
  v0.5). The reporter's claim likely reflects the upstream
  release binary's logs or the lsfg-vk core layer; both
  builds land in the same defect class — no pre-rotation of
  rendered content reconciled with the surface transform.
  #111's secondary point (geometry-change re-init never
  recreates the swapchain) is mitigated in our shell —
  reconfigure destroys the context and re-attaches a fresh
  surface, and `setOutputSurface` destroys the swapchain —
  but caps are still sampled only at (re)creation, which
  leaves the STALE-HINT RACE open (AOSP b/301238858:
  SurfaceFlinger can publish an incorrect transform hint
  before it learns the display rotation; an app that
  samples once can freeze a portrait hint — this explains
  the born-landscape failure as readily as the rotate
  failure). v0.5 logs the caps at EVERY swapchain creation
  so the next device log shows which value was in force.
  CAPTURE SIDE remains formally open (research Test 1):
  the AHB carries no orientation and raw AHB import
  discards BufferQueue transform metadata; if the
  present-side fix does not cure it, dump one captured AHB
  to PNG in landscape — sideways in the file indicts the
  mirror, upright closes the capture side for good.
  FIX (v0.5, native, `lsfg_render_loop.cpp` createSwapchain):
  `preTransform = VK_SURFACE_TRANSFORM_IDENTITY_BIT_KHR` —
  the truthful declaration for content produced in window
  coordinates, identical in effect to the CPU path — plus
  permanent logging of `caps.currentExtent`,
  `caps.currentTransform` (hex + name), image counts and
  supported usage at every swapchain creation. This is the
  research's RANKED FIX #2 (correctness-first stopgap:
  landscape extent + IDENTITY + unrotated content +
  recreate on change), chosen for v0.5 because it is
  one-line-invasive and doubles as the discriminating test;
  RANKED FIX #1 — proper native pre-rotation (identity
  extent + preTransform = currentTransform + rotate the
  final present blit, recreate on geometry change /
  SUBOPTIMAL) — is the v0.6 candidate once the mechanism is
  device-confirmed, and research fix #3 (key rotation off
  `Display.getRotation()` over JNI) is the immunisation if
  the stale-hint race shows up in the v0.5 log. RANKED
  ALTERNATIVES if the v0.5 log contradicts the mechanism
  (i.e. currentTransform logs IDENTITY and the picture is
  still sideways): #2 VD mirror content rotation — then
  adopt upstream's VD flags=0 (drop AUTO_MIRROR) and/or
  mirror upstream's TextureView output surface; #3 force the
  CPU blit path (kEnableWsiSwapchain=false) as a diagnostic
  A/B — it cannot be rotated by a present-time tag, so it
  splits input-side vs output-side in one run. NOT
  device-verified yet: the actual `currentTransform` value
  on the 10R (the v0.5 log line
  `createSwapchain: caps currentExtent=… currentTransform=…`
  is the proof — ROTATE_90 + upright picture closes F4.10 as
  CONFIRMED; IDENTITY + still-sideways reopens it at #2).

**FINDINGS:**
- CONFIRMED: engine builds clean for arm64/android-29 on r26d with
  exactly one 2-call-site patch (§4.3); JNI parity — our .so
  exports all 24 NativeBridge functions (release exports 22,
  F2.12); PanVK RC4 artefact hash-verified and packaged
  byte-identical (§4.2/§4.4); robustness2 + AHB strings present in
  the bundled PanVK ICD (Crucial #3); complete shell APK built and
  verified with zero proprietary content (§4.4); automatic
  own-process logcat capture implemented per Syed's requirement.
- SUSPECTED: none on our side — every remaining unknown is a
  device-side question (§4.5 P-1..P-4, all Syed's steps per the
  division of labour).

### v0.2 fix build (2026-10-30, from Syed's first playtest)

Playtest of v0.1 on the 10R cleared the port's biggest gates
(PanVK ICD loaded in-process, `probeShaders rc=0` — every shader
accepted on the first MT6895 tested — engine counters live) and
surfaced two shell flaws, fixed in v0.2: **F4.5** (no rotation
handling — landscape games presented sideways) and **F4.6**
(Share/Export sent the current launch stub instead of the
session log). Shell-only changes (`CaptureService.kt`,
`LogSaver.kt`, build version); engine, shaders, DLL handling
and the PanVK bundle are byte-identical to v0.1.

- Deliverable: `LSFGMali-v0.2.apk` — package `com.syed.lsfgmali`,
  versionCode 2 / versionName 0.2, 7,627,663 B,
  sha256 `15a579dd2d6cb245f9d5d7582795d13bb3300d1eb00f2607184b04375fec8b59`.
- Verified here: aapt2 badging (package/version/min/target SDK);
  apksigner verify OK (same debug cert as v0.1 →
  `4dff7661…c57a9`, so it installs as an update and keeps app
  data, including the logs folder); zip listing shows
  `liblsfg-android.so`, `libvulkan_panfrost.so`,
  `liblsfgmali_shim.so`, `libc++_shared.so` and **zero**
  Lossless.dll entries; the dex contains both fixes' log strings
  (`session start: rotation=`, `rotation change (…`,
  `share log: exporting session log …`).
- NOT verified (needs the device, Syed's steps): rotation
  reconfigure end-to-end in a landscape game; VD-resize mirror
  behaviour on the OnePlus ROM; which stranded v0.1 session logs
  survived the 5-file prune (F4.6).
- POSTSCRIPT (device result, 2026-10-30 03:59 IST, log
  `lsfgmali-20261030-035755.log`): the reconfigure DID verify —
  rotation detected via surfaceChanged, engine re-initialised at
  2412x1080, frames flowed in landscape (generated 0→212). What
  failed instead was presentation (F4.7) and overlay lifecycle
  (F4.8) — fixed in v0.3 below.

### v0.3 fix build (2026-10-30, from Syed's second playtest)

Playtest of v0.2 proved the rotation reconfigure (engine healthy
in landscape) and surfaced two presentation/lifecycle flaws:
**F4.7** (stale surface latch + unpinned overlay window flipping
back to portrait geometry and inset-shrunk at 2316x1080) and
**F4.8** (overlay presenting its last frame indefinitely after
the game closed; overlay also presenting over our own UI).
Shell-only changes (`CaptureService.kt`, `Shell.kt`,
`MainActivity.kt`, build version); engine, shaders, DLL handling
and the PanVK bundle are byte-identical to v0.1/v0.2.

- Deliverable: `LSFGMali-v0.3.apk` — package `com.syed.lsfgmali`,
  versionCode 3 / versionName 0.3, 7,627,663 B,
  sha256 `3da001a28085de91171df7b590767fac1e2a7e3ec2721c254b4ad2d33a6586e1`.
- Verified here: aapt2 badging (package/version/min/target SDK);
  apksigner verify OK (same debug cert as v0.1/v0.2 →
  `4dff7661…c57a9`, installs as an update, app data + logs
  preserved); zip listing shows `liblsfg-android.so`,
  `libvulkan_panfrost.so`, `liblsfgmali_shim.so`,
  `libc++_shared.so` and **zero** Lossless.dll entries; the dex
  contains the new log strings (`overlay size check:`,
  `overlay window params (`, `overlay window re-pinned`,
  `surface size-only change ignored`, `overlay flags:
  NOT_TOUCHABLE`, `no new captures`, `captures resumed`,
  `own app in foreground`, `overlay destroyed: view removed`,
  `teardown complete: overlay removed -> …`).
- NOT verified (needs the device, Syed's steps): the portrait
  flip-back actually gone in a landscape game; the ROM honouring
  the pinned full-display window (watch the `overlay size check:`
  lines — OK means surface == capture); no ghost/stale frame
  after rotation; the overlay vanishing ~2 s after the game
  closes and returning when it reopens; no overlay over the app's
  own UI.
- POSTSCRIPT (device result, 2026-10-30 ~04:16 IST, log
  `lsfgmali-20261030-041623.log` + screenshots): F4.8 CONFIRMED
  FIXED (watchdog hid the overlay on starvation; closing the
  game leaves no frame over other apps; own-app suppression and
  touch-through confirmed). F4.7's pin held — a landscape-START
  session was fully clean (`overlay size check: OK` from the
  first frame, 200 frames generated). One path stayed broken:
  portrait-start → rotate presents sideways at correct sizes —
  recorded as F4.9, fixed in v0.4 below.

### v0.4 fix build (2026-10-30, from Syed's third playtest)

Playtest of v0.3 split cleanly: F4.8 confirmed fixed on-device
and the landscape-start path fully clean, leaving exactly one
defect — the portrait-start → rotate path presenting the game
rotated 90° at correct sizes (**F4.9**, suspected:
`VirtualDisplay.resize()` not re-deriving the AUTO_MIRROR
orientation transform established at VD creation). Shell-only
change (`CaptureService.kt`: reconfigure() VD handling + one
session-start log line; build version); engine, shaders, DLL
handling, PanVK bundle and the v0.3 overlay lifecycle code are
untouched.

- Deliverable: `LSFGMali-v0.4.apk` — package `com.syed.lsfgmali`,
  versionCode 4 / versionName 0.4, 7,631,759 B,
  sha256 `1768d6e92ac2287b7f8480283b6952b6ac22f7f9fa1369d018a0cc593a15e003`.
- Verified here: aapt2 badging (package/version/min/target SDK);
  apksigner verify OK (same debug cert as v0.1–v0.3 →
  `4dff7661…c57a9`, installs as an update, app data + logs
  preserved); zip listing shows `liblsfg-android.so`,
  `libvulkan_panfrost.so`, `liblsfgmali_shim.so`,
  `libc++_shared.so` and **zero** Lossless.dll entries; the dex
  contains the new log strings (`virtual display recreated at`,
  `never resized`, `virtual display created at`,
  `falling back to resize`) alongside the retained v0.3 strings.
- NOT verified (needs the device, Syed's steps): the F4.9
  verdict itself — portrait start → START → rotate into the
  landscape game → the picture must be UPRIGHT (the
  `virtual display recreated at 2412x1080` line in the exported
  log proves the new path ran); the resize() fallback path
  (only taken if VD creation fails mid-reconfigure — not
  expected to trigger).
- POSTSCRIPT (v0.4 retest, 2026-10-30 ~04:32 IST, session
  `lsfgmali-20261030-043057.log`): F4.9's verdict came back
  NEGATIVE — still sideways, in a session that STARTED in
  landscape with the VD born at 2412x1080. F4.9 retired as
  disproven; root cause re-investigated end-to-end and
  recorded as **F4.10** (swapchain preTransform false
  assertion). The v0.4 VD-recreate change is KEPT (it is
  strictly more correct than resize and the run sheet's
  protocol depends on its log lines), but it is no longer
  believed to bear on the rotation.

### v0.5 fix build (2026-10-30, from the F4.10 investigation)

First NATIVE change of the port: `lsfg_render_loop.cpp`
`createSwapchain()` presented with
`.preTransform = caps.currentTransform` while rendering
un-rotated content in window coordinates — under a
hint-honouring WSI (the platform loader translates the
declaration into the compositor's buffer transform) the
already-upright landscape frames were rotated 90° at
composition (F4.10). v0.5 declares
`VK_SURFACE_TRANSFORM_IDENTITY_BIT_KHR` and logs the full
surface caps (`currentExtent`, `currentTransform` hex +
name, image counts, supported usage) at every swapchain
creation, so Syed's next log carries the deciding value.
Engine rebuilt in-tree (build-arm64, incremental), stripped,
staged into the shell; all v0.3/v0.4 shell behaviour
(overlay lifecycle, watchdog, pinned window, VD recreate,
session-aware export, automatic logging) preserved —
shell Kotlin unchanged this build.

- Deliverable: `LSFGMali-v0.5.apk` — package
  `com.syed.lsfgmali`, versionCode 5 / versionName 0.5,
  7,631,759 B,
  sha256 `d289ecc61b1ccbe57534a070dc32804735b2ba1741d0e54cd1d8c5b229aa9f36`.
- Verified here: aapt2 badging (v5/0.5, minSdk 29,
  targetSdk 35); apksigner verify OK (same debug cert as
  v0.1–v0.4 → installs as an update, app data + logs
  preserved); zip listing shows `liblsfg-android.so`,
  `libvulkan_panfrost.so`, `liblsfgmali_shim.so`,
  `libc++_shared.so` and **zero** Lossless.dll entries; the
  APK's `liblsfg-android.so` hash equals the freshly built
  + stripped engine (75ad5f5b…) and contains the new log
  strings (`createSwapchain: caps currentExtent=…
  currentTransform=…`, `preTransform=IDENTITY (F4.10 fix…)`).
- NOT verified (needs the device, Syed's steps): the F4.10
  verdict itself — BOTH start orientations were broken, so
  both are in the retest (RUN_SHEET §4): (a) session started
  already in landscape, (b) session started in portrait then
  rotated into the game. Success = upright picture in both
  AND the exported log's `currentTransform=` value
  (ROTATE_90 expected — that value + an upright picture
  closes F4.10 as CONFIRMED). If the picture is still
  sideways, the same log line splits the next step:
  ROTATE_90 logged → the ROM applies an extra rotation
  beyond the spec model → proper pre-rotation (research
  rank #1) becomes v0.6; IDENTITY logged → capture side
  moves up → AHB-to-PNG dump test (research Test 1) next.

---

## Appendix — consolidated blocker list (remapped)

CONFIRMED: (1) robustness2 — solved in fork (§2.5), verify on-device.
(2) fd-sharing dead on Android → AHB path mandatory (already the
design). (3) Remaining LSFG gates: AHB set, timeline sem, memory
model, FP16-or-FP32 shaders, storage-image trio. (4) Platform:
per-session MediaProjection consent, +50–80 ms, sideload-only, OEM
touch filters. (5) PanVK/kbase: `MEM_ALLOC_EX` ENOTTY on 5.10
MediaTek kbase → patch-110 fallback required, exists. (6) PanVK
sparse = NO-GO (LSFG unaffected). (7) DLL 3.2.1.0 lacks FP16 set →
FP32 path (Crucial #2).
OPEN (on-device): (8) OnePlus kbase version/SELinux for PanVK.
(9) G610 MC6 variant match in PanVK profiles. (10) PanVK AHB import
exposure. (11) PanVK robustness2 source-level confirmation (pinned
Mesa extension table, §4.2).

---

## Appendix B — First on-device framegen session (2026-10-30 03:54 IST, v0.1, Syed's OnePlus 10R)

Source: `lsfgmali-20261030-035416.log` (1,257 lines, same-session export). Driver: PanVK (bundled ICD, shim rc=0). DLL 3.2.1.0, FP32 path (fp16=false), 2x, 3.1P, flow 0.7. Session started with the phone already in landscape, so v0.1 capture geometry was correct this run (2412x1080).

**Device features probe (tag `lsfg-vk-framegen`, first MT6895 record):**
`storageImageExtendedFormats=1, storageImageReadWithoutFormat=1, storageImageWriteWithoutFormat=1, shaderInt16=1, shaderFloat16=1, robustness2=0, vulkanMemoryModel(core)=0, timelineSemaphore(core)=0, sync2(core)=0`

Open puzzle: robustness2 and the core 1.2/1.3 features read 0 under the bundled PanVK, contradicting the Mesa-side expectation (WORKAROUNDS.md B1). Either the probe under-reports PanVK's feature structs or the RC4 build genuinely lacks them. It did not matter empirically: engine initialised with no INIT FAILED and no DEVICE_LOST.

**Frame generation CONFIRMED on Dimensity 8100-MAX / Mali-G610 (first such record):**
- 03:54:40 generated=7 posted=15 unique-captures=27
- 03:54:45 generated=93 posted=187 unique-captures=295
- 03:54:50 generated=159 posted=318 unique-captures=479
- 03:54:55 generated=166 posted=332 unique-captures=481, then frozen (rotation back to portrait + game exit at ~03:54:50; starved capture, not a crash — stats lines keep ticking with unchanged counters).

**Confirmed live on this ROM:** skip-screenshot unavailable (NoSuchMethodException) — the F2.15 capture feedback-loop exposure is real on the 10R's ROM; this session's capture stream nevertheless stayed productive throughout. Visual fade not determinable from the log; awaiting Syed's report.
