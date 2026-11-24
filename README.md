# ViceFrame

ViceFrame is a Windows compatibility project built on XCL. The immediate goal is to load Grand Theft Auto VI's story on an NVIDIA PC, render a 3D world and demonstrate player input. This checkpoint reaches the menus and illustrated loading screens. **Controllable gameplay is still unverified.**

## Current 60 FPS trial

The local 60 FPS trial build on an RTX 5090 accepted a 33,333 µs interval and set a 16,667 µs frame clock. The paired 82-second opening runs in the distributed build recorded 2,135 presents at stock and 4,102 with the override. The fastest 300-frame samples were 30.0 and 60.0 FPS respectively.

A separate instrumented story run on 24 November 2026 reached the illustrated loading screen and reported 13,488 presents. Its 43 measured windows ranged from 13.8 to 48.5 FPS. D3D12 reported a hung device at 494.463 s. The first logged two-second memory wait timeout followed at 495.027 s. Controllable gameplay and its frame rate remain unverified.

Build the Rust workspace using the Windows instructions below. To opt in for one PowerShell session, set `$env:XCL_FRAME_RATE_OVERRIDE = '60'` before launching XCL. Remove that environment variable for a stock-timing run. This import starts from the project checkpoint `beb505cf` and applies the distributed frame-clock patch. The measured trial executable has SHA-256 `10fd57064a6e2220a12a2824d75f1474a628d1718544d4790211ed35dcb60a0b`. The source import has its own build verification.

[Loading-screen investigation](docs/STORY-ROADMAP.md) · [Measured trial summary](docs/rtx5090-trial.json) · [Trial comparison tool](scripts/summarize_trial.py)

## Earlier RTX 5090 checkpoints

| Area | Verified on 24 November 2026 |
| --- | --- |
| Startup and controls | The title displays its menus, accepts the local profile selection and enters story loading. |
| Audio | Menu clicks are audible. Clear, synchronised audio throughout the intro and gameplay still needs verification. |
| Graphics and streaming | In the first patched story trial, XCL presented 8,048 frames and reported 66,570 storage requests with zero request failures. The final capture shows loading artwork. |
| Cross-queue waits | All 11 targeted tests passed. During story loading, eight outstanding waits across six submitted lists received the expected value. All 24 traced submissions sent after the last present completed on the GPU. |
| Story transition | The final present returned at 345.308 s. The title exited at 375.653 s without presenting a world frame. |
| Shader and pipeline preparation | XCL can compile previously recorded shader inputs before launch. At the crash, one game worker was waiting for a pipeline and a compiler worker was active. The dependency between that work and the presenting thread is still under investigation. |

These timings are from trial `play-p5oqowfc`. Its shader warm-up found 602 cached inputs. Earlier trials used a different cache, so compare the sequence of GPU events before comparing wall-clock times.

## Current investigation

The queue-side wait fix lets a game thread submit a producer after queuing the consumer that depends on it. The GPU completed both sides in the story trial. At the crash, the presenting thread was in a Windows single-object wait. Identifying what signals that object is the next diagnostic milestone.

A second trial, `play-bdk_557q`, started with 668 cached shader inputs and reached 9,023 presents. Twelve thread snapshots completed during loading. At 380.015 s the sampling thread itself faulted, about 0.11 s after a successful present. This trial ends before it can explain the original freeze. The current source adds `pso-bind-begin` and `pso-bind-end` to the opt-in sync trace. They record the pipeline key, stage, build owner and elapsed wait directly in XCL. These events can be collected without suspending game threads for stack snapshots.

The `experiment/driver-cache` branch contains the independently tested driver-cache candidate. With driver 621.24 on the RTX 5090, a synthetic compute pipeline took 28.280 ms to create cold and 0.573 ms in a fresh process using the saved 5,898-byte blob. The native call accepted the blob and the GPU returned matching output. A control that rebuilt the same pipeline without the XCL blob in the reuse process took 0.454 ms. This synthetic test demonstrates reuse and does not establish a speed-up. The translated XCL path separately recorded a store, a hit in a new process, and zero cache activity with the switch disabled. A stale blob recovered successfully. The read-only ACL setup still permitted writes on this PC, so that specific test is inconclusive. A separate unwritable-path test preserved correct GPU output.

In the first story run with the native cache enabled (`play-hkmi00zw`), XCL started with an empty driver-cache folder and saved 115 entries. It reported one reuse during the same run and zero rejected or damaged entries. A 498,724-byte DXIL compute pipeline took 41.737 s to create on the driver after 63 ms in the DXIL stage. The final present returned at 434.622 s, with 9,881 presents in total. At 467.584 s D3D12 reported a device hang (`0x887a0006`) and the title took the same null-write exit. Eleven queued waits had received their value. One timed out, with no confirmed GPU completion for the recorded producer. The earlier eight-wait result stands for the earlier trial. The cache's effect on story loading time remains unmeasured. Gameplay verification requires a recognisable 3D world, a response to player input and continuing frames.

A separate replay on the same RTX 5090 opened the exact 498,724-byte DXIL from that run. Each of six fresh processes created one pipeline with the production root signature. Every native call succeeded. The first process used no XCL blob and took 12,785.786 ms. The remaining no-blob controls took 3.831 and 4.045 ms. The three processes using the 665,818-byte saved driver blob took 3.303, 3.113 and 3.088 ms. The NVIDIA driver cache was left in place. This verifies that the driver accepts the captured story pipeline across launches. Whole-game loading time and the first-use benefit with a fully cold driver cache remain unmeasured.

Two subsequent, time-limited title launches on the same RTX 5090 also reused the large pipeline. Native creation took 232 ms in `run-3wqtfhgv` and 212 ms in `run-23byj_mx` using the saved disk entry, compared with 41.737 s in the original full trial. Those launches logged 116 and 126 disk hits respectively. Driver state and concurrent load varied between runs. Both launches ended by design during loading, before the previously observed device hang.

The last independently confirmed GPU submission in the cold story trace was 50781. Submission 50782 recorded one waiting list with 6,605 compute dispatches. Submission 50783 comprised 29 lists, with the matching write on the last list. Both fence watchers stopped without confirming completion. The trace records submission and completion at batch granularity. To locate the hang within a batch, correlate the next device-removal breadcrumb capture with the affected queue and command list.

## Build and test

Use an x64 Developer PowerShell with the MSVC C++ tools, Windows SDK and Rust MSVC toolchain. From the repository root, run

```powershell
cargo build --release --locked -p xcl
cargo test --release --locked -p xcl --lib wait -- --nocapture --test-threads=1
```

For a local run, set the window position to coordinates inside the intended monitor. The example takes the path to your own local package or executable as input. XCL writes the sync trace to the per-title logs under the data root.

```powershell
$env:XCL_DATA_ROOT = Join-Path $env:LOCALAPPDATA 'XCL-Story-Lab'
$env:XCL_DXIL_CACHE = Join-Path $env:XCL_DATA_ROOT 'shader-cache'
$env:XCL_SHADER_PROFILE = Join-Path $env:XCL_DATA_ROOT 'shader-profile'
$env:XCL_ASYNC_TEXTURE_UPLOADS = '1'
$env:XCL_ADAPTER = 'RTX 5090'  # Adjust for a different adapter
$env:XCL_ACTIVITY = '1'
$env:XCL_SYNC_TRACE = '1'
$env:XCL_GPU_WAITS = '1'
$env:XCL_WINDOW_POS = '2592,32'  # Screen 2 on the measured PC
$gamePath = Read-Host 'Path to your local game package or executable'
.\target\release\xcl.exe run $gamePath
```

On subsequent launches, run `./target/release/xcl.exe warm-shaders $env:XCL_SHADER_PROFILE` before starting the title to prepare inputs recorded on earlier runs. A fresh checkout starts with an empty shader profile.

The measured runs used the lab's local display and intro-audio launcher, optional BCPack worker and existing shader profile. The worker binaries and media are external to this repo. A fresh checkout can reproduce the synthetic wait test without any game files. Standard XCL usage, optional runtime dependencies and other supported titles are described in the [upstream README](README.upstream.md).

## Source and scope

The first commit preserves the upstream source tree from `ivall/xcl` at `7a033c8cd13194900cd49c01ed60a59c6c9559c7`. The next checkpoint adds the local graphics, streaming, audio and synchronisation changes. This repository contains compatibility-layer source and its test fixtures. The game installation, personal saves, recordings and machine caches stay on the development PC. The vehicle and FiveM converter is a separate project.

[Upstream project](https://github.com/ivall/xcl) · [Licence](LICENSE) · [Third-party notices](THIRD_PARTY.md)
