# Port status: upstream 0.0.9 on RDNA3

As of 2026-11-25. What of the upstream project
([mochizuki0323/DLSSNR-AMD](https://gitlab.com/mochizuki0323/DLSSNR-AMD)) is in this branch, what is not,
and what has and has not been checked on hardware.

## Base

- This branch is upstream 0.0.8 (`960038f`) plus the RDNA3 port (the 81 commits of `rdna3-rebased`).
- Upstream 0.0.9 (`9503f6b`, 0.0.8.1 to 0.0.9) was merged into it with a normal three-way merge. The 11
  files both sides had changed were resolved by hand; everything else is one side's as it was.
- The RDNA3 network (`linux/shaders/rdna3/`) is the branch's own work and is not upstream's.

## In place

| Upstream feature | On RDNA3 |
| --- | --- |
| New NR Control (Add-ons, Hotkeys, languages, Layers), Unicode paths, 0.0.8.1 | yes, shared host code |
| Memory Guard, lower video memory, History Precision, dynamic resolution, 0.0.8.2 | yes, shared host code |
| Interleaved presets of Passes (fast), 0.0.9 | yes; the two markers the RDNA3 network lacked (`noise_buffer`, `post_no_out1`) are ported |
| Style grade (Natural / Cinematic), 0.0.7 | ported to the RDNA3 tree: extra `*_grade.spv` pipelines, the Standard path unchanged |
| CC_SMALL_FP8 and CC_SILVER_AARDWOLD networks | unchanged from 0.0.8; see the verification below |
| int4 mixed | unchanged from 0.0.8, one tier; tables in `linux/data/int4/rdna3/`, weights `dlssnr-amd/dlssnr-int4.bin` |
| The small model (`CC_SMALL_FP8.bin`) | unchanged from 0.0.8 |
| Temporal reuse (the network on every Nth frame) | kept; it is this branch's own feature |

Model files use upstream's names since this merge: `CC_SILVER_AARDWOLD.bin` and `CC_SMALL_FP8.bin`. An
installation of an earlier package needs a reinstall (`install.sh --dll`), as for every new package.

Upstream replaced int4 mixed with the Optimized layers in 0.0.8.3. On RDNA3, network 3 is this branch's
int4 mixed network (`CC_SILVER_AARDWOLD_INT4_MIXED`) instead: Quality runs the plain network, Balanced runs
it with int4, and the Performance levels read as Balanced (there is one int4 network). `install.sh`,
`build_package.sh`, `build_int4.sh` and `assemble_product_data.sh` keep the `-rdna3` package to the three
networks it has.

## Not in place

- **CC_SMALL_FP8_EXPERIMENTAL** on RDNA3. Needs `NR_SACT`, `NR_FIXP` and `NR_VOMERGE` in the RDNA3 kernels,
  `pipelines-experimental.json` for `rdna3/`, the `fswin32nhd` pipeline, and a new golden (its picture is not
  the base network's by design).
- **CC_SILVER_AARDWOLD_OPTIMIZED** and its four levels on RDNA3. Needs `pipelines-optimized.json`,
  `-performance.json`, `-ultra.json` and an int4 overlay for `rdna3/`, the same shader work as above plus
  `NR_CONSTH`, and new goldens.
- **The Windows RDNA3 support** (upstream's ACO records, #26). `windows/` is upstream's tree, carried by the
  merge as upstream ships it; this branch's RDNA3 port is Linux-only and does not build it.

Both networks above are RDNA4-only for now, which is what the port-gap list recommends doing last: they
need new goldens, and a golden needs the card.

## Verification

Checked here (Linux x86_64, no GPU in the build environment):

- The shaders build for both architectures with the pinned glslang 16.5.0, default and `--int4`.
- Every base-network and int4 SPIR-V is byte-identical to 0.0.8's except `g_fswinimagepreds32`,
  `g_fswinimagepreds32nh` and the three temporal pre variants: those read the noise table from binding 5
  instead of the weight buffer (the `noise_buffer` the 0.0.9 host requires). The glslang `-H` dump of the
  old and new kernel differs only in declarations and decorations, not in instructions.
- The new `*_grade.spv` files build for the base and int4 networks, and their temporal twins.
- `shader-constants.txt` is accepted by the host's manifest reader with `noise_buffer`, `post_no_out1` and
  `kswap` set.
- The host compiles for rdna3 and rdna4, with and without `NR_INT4`, and the native test binaries
  (`run_frame`, `nr_graph`, the e4m3 tests, `wmma_half_probe`, `int4_pipelines`) compile.

Not checked here (needs the card and a model extracted from `nvngx_dlssnr.dll`):

- `python3 linux/test/check_rdna3.py --model CC_SILVER_AARDWOLD.bin` must be **EXACT**. The network's
  arithmetic did not change, so any difference is a bug in the merge, not a new golden.
- `python3 linux/test/check_rdna3_int4.py --model CC_SILVER_AARDWOLD.bin` must be **EXACT** for the same
  reason; it also rebuilds `dlssnr-int4.bin` and checks its SHA-256, which is how the install-time
  generator is verified.
- `python3 linux/test/check_rdna3_small.py --model CC_SMALL_FP8.bin` must be **EXACT**.
- `install.sh` with a real DLL: the int4 weights are made and match `dlssnr-int4.bin.sha256`, and the
  networks the installer lists match what the package has.
- The Style grade against NVIDIA's Natural and Cinematic frames (the RDNA4 port scored 49 dB; the RDNA3
  path is the same code at the same place in the post block).

## Notes for the next merge

- The Style grade's include (`linux/shaders/rdna3/include/nr_style_grade.glsl`) is a copy of RDNA4's, and
  the shared Passes (fast) shaders include RDNA4's copy: change both together.
- A new kernel that reads weights, a new marker the host checks, or a new pipeline name the host dispatches
  needs the same port in `linux/shaders/rdna3/` and `linux/src/core/nr_graph.cpp`; the manifest check
  refuses a folder whose constants disagree with the host.
- RDNA3's raw 32-bit views of an arena (counters, sync words) live at alias binding 16 plus the arena's
  binding, and the host has to bind them in every descriptor set it makes for a kernel, including the ones
  `Kernel::retarget`/`copy_set` rebuild.
