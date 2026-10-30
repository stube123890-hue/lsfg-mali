# lsfg-mali

The **lsfg-mali** project ports the most popular feature from the
Windows-exclusive Lossless Scaling program — frame generation — to
Android devices powered by MediaTek/Mali GPUs, the devices the existing
ports had written off.

Where lsfg-vk does this on Linux through a Vulkan layer, lsfg-mali does
it on Android through screen capture — and where stock Mali drivers hit
a wall, it carries its own open-source Vulkan driver with it.

## Why Mali needed its own port

LSFG's engine core requires `VK_EXT_robustness2` (null descriptors).
Stock Mali drivers don't expose it, so on most Mali devices the
existing Android port degrades to a capture-only mirror with **zero
generated frames**. MediaTek's own frame generation (MFRC) doesn't
exist on chips like the Dimensity 8100 at all.

lsfg-mali's answer: don't fight the stock driver — bundle **PanVK**,
Mesa's open-source Mali Vulkan driver, which implements robustness2 on
Valhall GPUs, and pair it with a robustness2-free engine core that
routes null-descriptor writes to a 1×1 fallback image.

## Features

With lsfg-mali, you can:

- Use Lossless Scaling Frame Generation on a MediaTek/Mali Android
  phone — **device-confirmed working** on the OnePlus 10R
  (Dimensity 8100-MAX / MT6895, Mali-G610 MC6), the first recorded LSFG
  frame generation on this chip.
- Generate frames at full resolution: FP32 path produces ~5.5
  generated frames/s at 2412×1080, with capture running at ~27/s
  (input lag is inherent to the capture route).
- Run on the bundled PanVK driver — no root, no custom ROM, no driver
  flashing; the driver ships inside the APK.
- Correct output orientation: the F4.10 rotation bug (swapchain
  preTransform falsely declaring the surface's ROTATE_90 transform) is
  root-caused and fixed.

An FP16 fast path (expected ~1.5–2× faster) is in the works — it needs
a Lossless.dll ≥ 3.2.2.0, which ships the precompiled FP16 SPIR-V set
that 3.2.1.0 lacks.

> **⚠️ FP16 requirement:** the FP16 fast path only works with a
> `Lossless.dll` taken from an up-to-date **Lossless Scaling install on
> Steam (DLL version 3.2.2.0 or newer)**. Older DLLs (e.g. 3.2.1.0) do
> not contain the FP16 shader set, so lsfg-mali falls back to the FP32
> path — fully working, just slower. To unlock FP16: update Lossless
> Scaling in Steam, then re-copy the DLL from the updated install.

## Getting Started

lsfg-mali is designed to work on arm64 Android devices with a Valhall
Mali GPU. Android 10 (API 29) or newer is required.

1. Download `LSFGMali-v1.0.apk` from the
   [Releases](../../releases/latest) page and install it.
   (v1.0 uses a new dedicated release key — uninstall any v0.x debug
   build first, once.)
2. Supply your own `Lossless.dll` from a legitimately owned copy of
   [Lossless Scaling](https://store.steampowered.com/app/993090/Lossless_Scaling/)
   (Steam). The DLL is **not** included in this project and never will
   be — lsfg-mali, like lsfg-vk, is only the port; the frame-generation
   technology itself belongs to Lossless Scaling. Any recent DLL runs
   the FP32 path; for the **FP16 fast path you need DLL 3.2.2.0 or
   newer** from an updated Steam install (see the FP16 note above).
3. Grant the screen-capture (media projection) and overlay permissions
   when prompted, and start generating frames.

Package: `com.syed.lsfgmali` · Version: 1.0 (versionCode 6) ·
arm64-v8a

APK SHA-256:
`2a58369e1151c13b5b36b114d420d306a6e2b706e90b2d82167fddead1ae1162`
Signing certificate SHA-256:
`c141c5ebeff393e0b788ae22cc6fba8ee0a2d64094ff7bb1a1e785d7b3c76dd9`

Permissions requested: foreground service + media projection (screen
capture), overlay window, notifications. No internet, no contacts, no
location.

## Documentation

- [HOW_WE_MADE_IT.md](HOW_WE_MADE_IT.md) — the full story of the port,
  written the night it shipped.
- [PORTING_NOTES.md](PORTING_NOTES.md) — the complete technical record:
  Phases 1–4 × 5 sub-phases, every flaw with file/line evidence, the
  DLL pairing verdict, PanVK binary verification, and the F4.10
  rotation root cause.

## Credits & Inspiration

lsfg-mali exists because of the work it is inspired by, and says so
plainly:

- **[lsfg-vk](https://lsfg-vk.dev) by PancakeTAS** — the original port
  of Lossless Scaling Frame Generation to Vulkan/Linux, and the direct
  inspiration for this project: the engine, the approach, and the idea
  that LSFG belongs on platforms Lossless Scaling itself doesn't ship
  for. This repository's presentation is modelled on theirs, with
  respect. lsfg-vk is licensed CC BY-NC-ND 4.0.
- **[LSFG-Android](https://github.com/FrankBarretta/LSFG-Android) by
  FrankBarretta** — the Android port lineage lsfg-mali builds on,
  including the robustness2-free direction its session layer
  prototyped.
- **[PanVK / Panfrost](https://docs.mesa3d.org/drivers/panfrost.html)
  (Mesa project)** — the open-source Mali Vulkan driver bundled in the
  APK; its Valhall robustness2 support is what makes frame generation
  on Mali possible at all.
- **Lossless Scaling** by THS — the original Windows program whose
  frame-generation technology all of these ports carry to new
  platforms. Buy it on Steam; bring your own DLL.

Port, PanVK packaging and documentation: **Buggy & Syed Muzammil**,
2026. Built for personal use and research on our own devices.
