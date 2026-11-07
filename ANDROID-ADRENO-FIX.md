# Cinnabar: unofficial Android crash fix for some Adreno devices

> **Unofficial fork.** All credit for Cinnabar goes to the original authors.
> Original project: **cinnabar-mc/cinnabar** — https://github.com/cinnabar-mc/cinnabar
> This fork is not affiliated with or endorsed by the original project, Mojang or Microsoft.
> License: unchanged from upstream (PolyForm Noncommercial). See `LICENSE`. Non-commercial use only.

## What was fixed
On a Redmi Note 13 Pro 5G (Adreno GPU, HyperOS, Android 14) Cinnabar 0.1.31 crashed every launch with
SIGSEGV a few seconds after startup, before any setting could be changed. With this change the app starts and
is playable on that device.

## Cause
The crash report (`tombstone.pb`) shows the crash inside the Qualcomm Vulkan driver
(`libllvm-glnext.so`, `LoadQGLCProgram`), called from wgpu `create_render_pipeline` through Bevy's
`PipelineCache`, on the `Async Compute` thread. Cinnabar reuses a driver pipeline cache between launches
(`crates/render/src/pipeline_compilation.rs`). The working assumption is that loading this cache makes
the Adreno driver crash. Confirmed only by the fix working on the test device, not by a driver-side diagnosis.

## The change
One line in `crates/render/src/pipeline_compilation.rs`: skip creating the persistent Vulkan driver
pipeline cache when `cfg!(target_os = "android")`. The code itself marks this cache as optional.
See commit `21087d4`.

## Tradeoff
Shaders are compiled again on every launch, so the first moments after startup can be slower and warmer.

## Tested
| Device | GPU | Android | Result |
|---|---|---|---|
| Redmi Note 13 Pro 5G (2312DRA50G) | Adreno | 14 | Works. Vanilla FPS reported close to Bedrock; shader packs lower FPS and raise heat as expected. |
| Infinix GT 20 Pro (X6871) | Mali-G610 MC6 | 15 | **Still crashes.** Different bug: SIGSEGV in `libGLES_mali.so` (`cmpbe_v2_compile_multiple_shaders`) while compiling a compute pipeline. |

## What to expect
This may help other Adreno devices with the same crash. It is **not** expected to fix Mali or other
driver crashes. If you try it, please report your device model and GPU.

## Install
1. Download the APK from the Releases page.
2. Uninstall any existing Cinnabar first. CI signs every build with a throwaway key, so updates cannot install over a build signed with a different key.
3. Install and open.

## How it was done
- Crash reports read with Python (raw protobuf decoding of `tombstone.pb`) and analysis assisted by Claude (Anthropic).
- Source browsed and edited on-device in Termux (`git`, `ripgrep`, GitHub CLI).
- APK built by upstream's own `.github/workflows/android.yml` on GitHub Actions (cargo-ndk, pinned Android NDK).
- Scheduled builds were removed from this fork's workflow; builds are manual.

## Reporting upstream
Both crashes should be reported to the original project:
https://github.com/cinnabar-mc/cinnabar/issues
