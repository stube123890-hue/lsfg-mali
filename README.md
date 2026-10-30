# LSFG Mali

Lossless Scaling Frame Generation (LSFG) working on MediaTek/Mali Android
devices — a port of [lsfg-vk](https://github.com/PancakeTAS/lsfg-vk) /
LSFG-Android to phones the upstream app had written off.

On most Mali devices, upstream LSFG-Android degrades to a capture-only
mirror with zero generated frames, because the engine's Vulkan core
requires `VK_EXT_robustness2` and stock Mali drivers don't expose it.
LSFG Mali solves this by carrying an open-source driver instead of
fighting the stock one: it bundles **PanVK** (Mesa's Mali Vulkan driver,
which implements robustness2 on Valhall) and a robustness2-free engine
core that routes null-descriptor writes to a 1×1 fallback image.

## Status — v1.0

- **Device-confirmed working** on OnePlus 10R (Dimensity 8100-MAX /
  MT6895, Mali-G610 MC6) — the first recorded LSFG frame generation on
  this chip. MediaTek's own MFRC frame generation doesn't exist on the
  8100 at all.
- FP32 path: ~5.5 generated frames/s at full 2412×1080 (capture ~27/s;
  input lag is inherent to the capture route).
- FP16 fast path is expected ~1.5–2× faster but needs a Lossless.dll
  ≥ 3.2.2.0 (the current build pairs with 3.2.1.0, which lacks the
  precompiled FP16 SPIR-V set). Performance work, not a function blocker.
- v1.0 is signed with a new dedicated release key — installing over any
  v0.x debug build needs a one-time uninstall first.

## Install

Download `LSFGMali-v1.0.apk` from the
[Releases](../../releases) page and install it. You must supply your own
`Lossless.dll` from a legitimately owned copy of Lossless Scaling
(Steam) — it is **not** included in this project.

- Package: `com.syed.lsfgmali`
- Version: 1.0 (versionCode 6), min Android 10 (API 29), arm64-v8a
- APK SHA-256: `2a58369e1151c13b5b36b114d420d306a6e2b706e90b2d82167fddead1ae1162`
- Signing certificate SHA-256: `c141c5ebeff393e0b788ae22cc6fba8ee0a2d64094ff7bb1a1e785d7b3c76dd9`

Permissions: foreground service + media projection (screen capture),
overlay window, notifications. No internet, no contacts, no location.

## Documentation

- [HOW_WE_MADE_IT.md](HOW_WE_MADE_IT.md) — the full story of the port.
- [PORTING_NOTES.md](PORTING_NOTES.md) — the complete technical record:
  Phases 1–4, every flaw with file/line evidence, the F4.10 swapchain
  preTransform rotation root cause and fix, DLL pairing verdict, PanVK
  binary verification.

## Credits

- Upstream engine: lsfg-vk by PancakeTAS; Android port lineage:
  LSFG-Android by FrankBarretta.
- Vulkan driver: PanVK / Mesa (Panfrost project).
- Port, packaging and notes: Buggy & Syed Muzammil.
