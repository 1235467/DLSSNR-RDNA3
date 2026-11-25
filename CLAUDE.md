# Working on DLSSNR-AMD

Notes for contributors and coding assistants. README.md covers what the project is and how to build it.

## Layout rules

- `linux/` is the reference implementation. `windows/` is a separate copy for the AMD Windows driver
  (its shader compiler, LLPC) and is changed on its own. The two trees share no files on purpose.
- A workaround for the Windows driver goes into `windows/` only. A change goes into `linux/` only if
  it helps Linux. When a fix applies to both, make it in both, deliberately.
- Build scripts `cd` to the repository root themselves; run them from anywhere. Everything under
  `toolchain/`, `artifacts/` and `build/` is downloaded or generated and never committed.

## Things that must stay true

- **No NVIDIA files.** Never commit `nvngx_dlssnr.dll`, the weights, `CC_SILVER_AARDWOLD.bin`, `CC_SMALL_FP8.bin` (before 0.0.8.3 `dlssnr.bin`, `dlssnr-small.bin`), tensors extracted
  from the DLL, or packages that contain them. The model reaches users only through
  `linux/package/model-tools/extract_model.sh` run on their own DLL. CI checks that packages carry no model.
- **Host and shaders agree.** The C++ host is compiled with the defines in `*/build/arch/rdna4.sh`,
  the shaders with those in `*/shaders/rdna4/pipelines.json`; `shader-constants.txt` ties them together
  and the runtime refuses a shader folder that disagrees. Change both sides together.
- **Precision floor.** Keep at least the original network's precision at every operation: FP8 only
  where the original uses FP8, FP16/FP32 elsewhere. No new lower-precision boundary, no truncated
  inputs or lookup tables. Changing the order of a reduction changes the picture too - it is not a free
  reordering.
  The int4 layers of `CC_SILVER_AARDWOLD_OPTIMIZED` (`NR_INT4=1`, `linux/shaders/rdna4/pipelines-int4.json`,
  `[Network] OptimizedLayers`) are the one exception: they trade precision for speed by design, and switch off.
- **RenoDX's shaders stay RenoDX's.** `linux/shaders/renodx/` is RenoDX's code under its MIT licence; changes
  there are speed changes that leave every output bit as it was, each listed in the README in that folder.
- **Windows ACO Mode has a record for every network SPIR-V.** ACO Mode (the Windows default) runs the Linux
  network shaders (`linux/build/build_network.py rdna4`, also `--experimental`, `--optimized`, `--int4`, `--performance` and `--ultra`) as machine code from
  `windows/data/aco/records` and `windows/data/aco/int4/records`, found by the SPIR-V's hash. A new or changed
  Linux network shader needs new records in both (a missing one turns ACO Mode off for the run), and a new
  pipeline whose name has no native Windows shader needs a template alias in `shell-aliases.txt`.
  `CC_SMALL_FP8_EXPERIMENTAL` and `CC_SILVER_AARDWOLD_OPTIMIZED` (with or without its int4 layers) run with ACO
  Mode only.
- **RDNA3 is ACO machine code only.** RDNA3's network (CC_SMALL_FP8, CC_SMALL_FP8_EXPERIMENTAL, CC_SILVER_AARDWOLD and
  CC_SILVER_AARDWOLD_OPTIMIZED) is in this repository as `windows/data/aco/rdna3/records` (one record per pipeline) and,
  in place of its SPIR-V, `windows/data/aco/rdna3/{shaders,experimental/shaders,optimized/shaders}`: each pipeline's
  interface shell naming its record (`OpModuleProcessed "nr-aco-record <hash>"`, `nr_pipeline_binary.hpp`
  `record_reference`) and the network's marker files. Its shader sources are not part of this repository; records,
  stand-ins and markers change together.
- **No D3D device or swap chain inside the game's process** on Windows (NR Control): OptiScaler's hooks
  take them for the game's. `nr_present_probe.exe` finds DXGI's methods in a process of its own.
- **Weight packing.** A new kernel that reads weights must be added to the host's weight packing
  (`nr_graph.cpp`); otherwise it runs on unpacked data and its results are meaningless.
- **Reproducible shaders.** `fetch_deps.sh` pins glslang 16.5.0; with it `build_network.py` produces the
  same SPIR-V bytes every time. RenoDX's shaders (`linux/shaders/renodx`, the RenoDX style) are built
  with the pinned Slang 2026.19 by `linux/build/build_renodx.sh`. After a change that should not alter
  the arithmetic (refactor, comments, scheduling of independent work), compare the `.spv` files before
  and after.

## RDNA3

- `linux/shaders/rdna3/` is the RDNA3 network (e4m3 values held as FP16, see README-RDNA3.md); `rdna4/` is not
  touched by it. Build scripts take `NR_GPU=rdna3`; the host is gated by `NR_ARCH_RDNA3`.
- Run `python3 linux/test/check_rdna3.py --model CC_SILVER_AARDWOLD.bin` before and after any change to it. A change
  that leaves the arithmetic alone must be EXACT; a change that does not is a new golden and needs a reason in the
  commit. Time kernels with the GPU idle.
- A raw 32-bit view of an arena (counters, sync words, scale tables) goes at alias binding 16 plus the arena's
  binding, never next to an e4m3 view: the e4m3 bindings read the FP16 twin buffer. The int4 data's views are such.
- RDNA3 has no Optimized layers and no CC_SMALL_FP8_EXPERIMENTAL: network 3 in the in-game list is
  CC_SILVER_AARDWOLD's int4 mixed network (`pipelines-int4.json`, `NR_INT4=1` on the host, tables in
  `linux/data/int4/rdna3/`, weights `dlssnr-amd/dlssnr-int4.bin`), one tier. Run `python3
  linux/test/check_rdna3_int4.py --model CC_SILVER_AARDWOLD.bin` before and after any change to it, as above; its
  picture is not the default network's and has its own golden. Every int8 fragment in those kernels is multiplied
  twice with `NR_I4_MMA2` (the second marked saturating): the host rewrite turns the pair into the two halves of the
  fragment and refuses a pipeline whose WMMAs are not in marked pairs, so a WMMA added outside `NR_I4_MMA2` fails the
  build of that pipeline.
- The small model (`CC_SMALL_FP8.bin`, `[Network] Model`): run `python3 linux/test/check_rdna3_small.py --model
  CC_SMALL_FP8.bin` as above for a change that reaches it; a change to `nr_graph.cpp` or the planner also needs
  `check_rdna3.py` to stay EXACT.
- The Style grade (`fswinimagepost32_grade.spv`, `style_grade` in `pipelines.json`) and the `noise_buffer` /
  `post_no_out1` markers are in the RDNA3 tree as they are in RDNA4's: the grade pipelines are extra SPVs (the
  Standard path is untouched), the noise table is the pre block's binding 5, and the post block does not write its
  second output. `include/nr_style_grade.glsl` is the same file in both trees (the shared Passes (fast) shaders
  include RDNA4's copy): change one, change the other.

## Performance work on RDNA4 (RADV / ACO)

Measured facts worth knowing before changing kernels:

- FP8 WMMA and VALU instructions do not overlap on RDNA4, not even across waves. Frame time is roughly
  WMMA time + VALU time + load time + idle time; saving VALU instructions saves time directly.
- ACO turns per-tile guards and uniform-looking branches it cannot prove uniform into phi copies and
  extra VGPRs; a "cheap" skip can cost occupancy. Count VGPRs and waves in the ISA
  (`RADV_DEBUG=shaders`) before and after.
- Wave arbitration is oldest-first: in persistent kernels, the second workgroup on a CU runs much slower
  than the first.
- Keep every LDS `coopMatLoad` stride and offset 16-byte aligned: the Windows compiler reads fragments
  with 8-byte-aligned paired loads and otherwise reads unwritten padding.
- Measure frame time over many frames and several runs at the same clocks (the log's per-frame GPU
  time); a single fastest run is not a result.

## Versions and releases

- The version comes from git tags only (`*/build/version.sh`): tag `v0.0.1` builds as `0.0.1`, later
  commits as `0.0.1-3-g1a2b3c4`. Package names and the DLLs' log lines carry it.
- Releasing: `git tag v0.0.2 && git push origin v0.0.2`. CI (`.gitlab-ci.yml`) builds the Linux packages
  and the Windows package and publishes a GitLab release (the packages in the project's generic package
  registry). For a Linux-only release, set the CI/CD variable `WINDOWS_PACKAGE` to `off` first
  (`glab variable set WINDOWS_PACKAGE off`).
  A tag with `-win-` in it (`v0.0.4-win-preview`) is a Windows-only release: CI builds no Linux packages.
  A tag with `-linux-` in it (`v0.0.6-linux-preview`) is a Linux-only release: CI builds no Windows package.

## Style

- Comments and user-facing text in English. Comments explain why (the measurement, the trap), not
  what the next line does.
- No internal version tags or references to files that are not in this repository.
- Package text (`*/package/README.txt`, installer messages) stays short and plain.
