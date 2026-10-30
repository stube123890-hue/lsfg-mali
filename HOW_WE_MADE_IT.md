# How LSFG Mali was made

*The full story of porting Lossless Scaling frame generation to a
MediaTek/Mali phone that the upstream app had written off. Companion to
`PORTING_NOTES.md`, which holds the complete technical record (Phases 1–4,
every flaw with file/line evidence). Written 2026-10-30, the night it
shipped.*

## The problem

LSFG-Android (FrankBarretta's standalone port of lsfg-vk) does frame
generation on Android through screen capture — but on most Mali devices it
degrades to a capture-only mirror with **zero generated frames**, because
the engine's Vulkan core requires `VK_EXT_robustness2` (null descriptors)
and stock Mali drivers don't expose it. Meanwhile the test target — a
OnePlus 10R with a Dimensity 8100-MAX (MT6895, Mali-G610 MC6) — had **no
PanVK or LSFG test record anywhere**: the G610 profile in the Android PanVK
bridge project was literally a stub marked "planned, not validated", and
MediaTek's own frame generation (MFRC) doesn't exist on the 8100 at all.

The bet: don't fight the stock driver. Carry an open-source one.

## The plan — four phases, five sub-phases each

Work was structured in Syed's order, each phase written up before the next
started, every phase report carrying a *Flaws & findings* section:

1. **PanVK deep study** — Mesa's Mali Vulkan driver: source inventory, the
   kernel bridge, CSF command submission on Valhall (v10), memory/shader
   model, packaging. Headline finding: **PanVK v10+ implements
   robustness2** — the "Mali wall" is a stock-driver property, not a
   silicon property. Counter-finding: the bridge project's G610 device
   profile was an unvalidated stub, so the 10R would be the first real
   data point for the chip.
2. **LSFG APK teardown** — the upstream app reversed apart: native layout,
   engine core, capture/composite path, and the robustness2 workstream
   mapped in full. Key discovery: the app's session layer already
   prototyped the escape for missing robustness2 — bind a 1×1 fallback
   image instead of null descriptors ("visually identical, ~zero perf
   cost") — it just never got pushed into the engine core.
3. **Turnip→PanVK replacement study** — how driver selection actually
   works on Android (loader mechanics, ICD discovery), what a Turnip
   package maps to on the PanVK side, and the kernel delta that can't be
   swapped: PanVK on Android talks to the proprietary `mali_kbase` kernel
   driver already in the phone, via the panvk-kbase bridge (RC4 bundle,
   Mesa 26.3.0-devel).
4. **Our build** — a new shell app (`com.syed.lsfgmali`) bundling the
   PanVK ICD in-process (selected via `VK_DRIVER_FILES`), the upstream
   engine rebuilt from the vendored fork, and a MediaProjection →
   ImageReader → engine → overlay pipeline. Syed's own `Lossless.dll`
   (3.2.1.0, hash-verified genuine) is supplied by the user at runtime —
   never packaged, never redistributed.

**Division of labour:** all device-side work (installs, runs, playtests,
log exports) was Syed's; the build side ends at a verified APK plus a run
sheet naming exactly which log lines to send back. Every session, the app
saves its own-process logcat to a file automatically — that closed loop
(file out, file back) is how every flaw below was diagnosed.

**Build pipeline note:** the sandbox has no working Gradle dependency
resolution, so the APK is built with a manual pipeline —
aapt2 → kotlinc → d8 → zipalign → apksigner — scripted in
`app-shell/build-apk.sh` / `build-release.sh`.

## First contact (v0.1, 2026-10-30 ~02:51 IST)

The first install cleared the port's biggest gates in one screenshot:
PanVK ICD loaded in-process (shim rc=0), the user's DLL copied with a
matching hash, `extractShaders rc=0`, and **`probeShaders rc=0`** — PanVK
accepted every frame-gen shader on the Dimensity 8100-MAX. First MT6895
data point, and the wall was already behind us. FP16 read `false`, exactly
as predicted for DLL 3.2.1.0 (its shader library stops at resource ID 302;
the precompiled SPIR-V sets live at IDs 304–400 in ≥3.2.2.0).

Hours later (03:54 IST), the first full session log proved the point of
the whole project: engine initialised clean (no INIT FAILED, no
DEVICE_LOST) and frames climbing in live gameplay — **generated 7 → 93 →
159 → 166 over ~15 s at 2412×1080**. Frame generation, on a Dimensity
8100-MAX, through screen capture. It worked. Making it *presentable*
took five more versions.

## The flaw chain (v0.1 → v0.5, one night)

Each playtest produced screenshots + a session log; each log produced a
numbered flaw; each flaw produced exactly one build.

- **F4.5 — zero rotation handling (v0.1 → fixed v0.2).** Capture geometry
  was frozen at session start from the phone's natural *portrait*
  metrics. A landscape game was squeezed sideways into a portrait strip.
  Fix: geometry derived from the display's *current* rotation, plus a
  DisplayListener-driven reconfigure that rebuilds the whole chain
  (reader → engine → surface → virtual display) mid-session.
- **F4.6 — stranded logs (v0.1 → fixed v0.2).** The exporter grabbed the
  current launch's stub log; the real game-session logs sat in private
  storage. Fix: session-aware export that scans for the newest file
  containing session markers and names its pick.
- **F4.7 — ghost image + flip-back (v0.2 → fixed v0.3).** On reconfigure
  the overlay re-attached its old surface, so the pre-rotation buffer
  stayed latched next to the new output (a doubled, ghosted picture), and
  the overlay window — created with MATCH_PARENT and shrunk by the
  cutout insets (2316×1080 vs 2412×1080 capture) — snapped back to
  portrait geometry mid-session while the engine stayed landscape. Fix:
  destroy and recreate the SurfaceView on every reconfigure; pin the
  window to the exact capture size (cutout included) and re-assert it on
  every surface event; treat only real rotation-*value* changes as
  rotation signal, ignore size jitter.
- **F4.8 — the immortal last frame (v0.2 → fixed v0.3).** Close the game
  and its last frame stayed pinned over every other app, indefinitely;
  the overlay also drew over our own UI. Fix: a starvation watchdog (no
  new capture for ~2 s → overlay destroyed, recreated when captures
  resume), own-app suppression, one ordered teardown shared by STOP /
  projection-stop / service-destroy, and `FLAG_NOT_TOUCHABLE |
  FLAG_NOT_FOCUSABLE` so the overlay can never take touches or focus.
  Device-confirmed fixed.
- **F4.9 — the VirtualDisplay theory (v0.3 → DISPROVEN by v0.4).** The
  portrait-start → rotate path still presented sideways. Hypothesis: a
  VirtualDisplay *created* in portrait keeps its mirror transform when
  `resize()`d. v0.4 therefore recreated the VD on rotation instead of
  resizing. The v0.4 retest killed the theory cleanly: that session
  *started* in landscape, its VD was born at 2412×1080, every size check
  read OK, 156 frames generated — still sideways. VD, window and surface
  were all exonerated in one log. (Bonus finding: on this ROM a
  MediaProjection token is single-use for `createVirtualDisplay`; the
  recreate path failed at session end and the resize fallback covered
  it — complexity v1.0 no longer needs, kept only as a fallback.)
- **F4.10 — the actual root cause (v0.4 → fixed v0.5, confirmed on
  device).** With the shell fully exonerated, the hunt moved to the
  native present path, in parallel with a sourced web-research pass. The
  research reframed it perfectly: *portrait is not a size, it's the
  default identity orientation state* — landscape has to be explicitly
  negotiated at the Vulkan present hand-off, and upstream's own tracker
  (LSFG-Android issue #111) documents missing pre-rotation handling in
  this engine version. Code inspection then proved the mechanism in our
  shipped binary: `createSwapchain()` copied
  `VkSurfaceCapabilitiesKHR.currentTransform` into the swapchain's
  `preTransform`, while the present path blits frames that are already
  upright in window coordinates (`blitOutputToSwapchain`, a 1:1
  `vkCmdCopyImage`). Declaring "my content is already rotated by
  ROTATE_90" about un-rotated content makes the compositor rotate it —
  the landscape picture comes out portrait-turned at perfectly correct
  sizes, and portrait (transform = IDENTITY) is correct *by accident*.
  Why upstream never hit it: its normal path is a CPU blit into a
  TextureView — the CPU path never tags a buffer transform, and a
  SurfaceTexture consumer publishes no transform hint. Our shell's
  SurfaceView has SurfaceFlinger as its BufferQueue consumer, which sets
  the hint (ROTATE_90 for a landscape window on a portrait-natural
  panel), and Mesa's WSI honours it. The fix (v0.5): declare
  `preTransform = IDENTITY` — the truthful declaration — and log the
  surface capabilities at every swapchain creation. The confirming
  session log (2026-10-30 04:49 IST) contains the smoking gun by name:
  `createSwapchain: caps currentExtent=2412x1080
  currentTransform=0x2 (ROTATE_90)` with
  `preTransform=IDENTITY (F4.10 fix)` — and the verdict from the phone:
  **"This version works perfectly."**

## Performance, honestly

The same confirming session doubles as the performance baseline: unique
captures ~27/s, generated frames **~5.5/s**, posted = 2× generated,
swapchain imageCount 5. The FP32 path at full 2412×1080 on PanVK is the
bottleneck; the capture round-trip and the deep present queue are the
input lag the playtester reported ("not much fps gain… more like input
lag"). None of that is hidden in the release notes: frame generation
here is a working breakthrough with a known, measured ceiling. The
levers, in order: FP16 via a ≥3.2.2.0 DLL (expected ~1.5–2× generation;
the phone reports `shaderFloat16=1`), the in-app 0.75× capture-resolution
option (~44% fewer pixels), and a future v0.6 with true pre-rotation
(removing the compositor rotation pass the IDENTITY declaration costs)
plus a shallower present queue.

## Release packaging (v1.0)

- **Icon:** the lsfg-vk pixel duck, adapted into a full launcher set —
  per-density mipmaps plus an adaptive icon (black background, duck
  foreground), wired via `android:icon` / `android:roundIcon`.
- **Signing:** v0.1–v0.5 used the shared debug key. v1.0 introduces a
  dedicated release key (RSA 4096) — cert SHA-256
  `c141c5ebeff393e0b788ae22cc6fba8ee0a2d64094ff7bb1a1e785d7b3c76dd9` —
  which is why installing v1.0 over a debug build needs a one-time
  uninstall, and why the key itself is kept out of the repo.
- **Verification (every build):** aapt2 badging, apksigner verify with
  cert comparison, zip listing audited for **zero** `Lossless.dll`
  entries, and the packaged engine `.so` hash-checked against the
  verified build (v1.0's engine is byte-identical to v0.5's:
  `75ad5f5b…`).

## What remains open

- FP16 on-device measurement (needs a user-supplied ≥3.2.2.0 DLL)
- v0.6-class polish: true pre-rotation + shallower present queue
- Other Valhall devices (only the 10R / MT6895 is verified)
- The capture feedback-loop exposure on ROMs where `setSkipScreenshot`
  is unavailable (test sessions stayed healthy; noted, not closed)

*Engine and Android session layer: lsfg-vk / LSFG-Android by
FrankBarretta (MIT). Driver: Mesa PanVK via the panvk-kbase Android
bridge. Full technical record: `PORTING_NOTES.md`.*
