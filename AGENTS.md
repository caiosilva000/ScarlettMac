# ScarlettMac engineering guide

This file applies to the repository root and its contents. Read [CONTRIBUTING.md](CONTRIBUTING.md) for submission instructions, [docs/compatibility.md](docs/compatibility.md) for observed results, and [docs/runbook.md](docs/runbook.md) for benchmark commands. Follow instructions in the prepared upstream tree when editing that tree. Use the pins and patch manifests in this repository to reproduce a published ScarlettMac configuration.

## What ScarlettMac does

ScarlettMac is an experimental Xbox Series X|S compatibility project for Apple Silicon. The title's compiled code and assets supply the workload. The runtime implements the platform behavior the title expects. In this project, performance work usually means reducing the cost of reproducing that behavior on a different host.

| Boundary | Responsibility |
| --- | --- |
| Title loading and system services | The runtime loads the Xbox executable and supplies the Xbox interfaces it calls, including input, saves, storage, threading and memory. Wine supplies Windows behavior to the x86-64 runtime on macOS. Rosetta translates x86-64 CPU instructions for Apple Silicon. |
| Graphics API | The runtime implements Xbox graphics calls using desktop D3D12. It tracks resources and views, prepares command lists, and preserves the ordering of copies, barriers, queue waits, fences and presentation. D3DMetal implements the host graphics path on Metal. |
| Shader translation | The runtime translates the compiled Xbox GPU programs into host-consumable DXIL. The translator must preserve the meaning of sampling, resource descriptors, numeric formats, lane masks, cross-lane operations and memory ordering. D3DMetal consumes the translated shaders as part of the host pipeline. |

Keep these boundaries in mind when reading a trace. An Xbox pipeline bind says which pipeline the title requested. A successful host pipeline creation establishes that the host accepted a descriptor and shader set. A GPU readback can verify specific output values. A displayed frame also depends on which image was composited and presented. Logs from these stages answer different questions.

## Repository map

| Path | Where to work |
| --- | --- |
| [scripts/](scripts/) | The XCL build and launch entry points. `scripts/build.sh --prepare` fetches the pinned upstream and applies the local patch in `.build/xcl-mac`. `scripts/build.sh --test-title` and `scripts/run.sh --test-title` build and exercise the synthetic title. |
| [patches/](patches/) | The XCL patch set and optional Lanes experiment. Prepare the pinned source before editing or regenerating a patch. [SOURCE-NOTICES.md](SOURCE-NOTICES.md) documents this lineage. |
| [runtime-preview/](runtime-preview/) | The separate VIctory-macOS-based Runtime Preview. Its default, checkpoint and performance recipes have separate `UPSTREAM` pins and ordered `patches/series` manifests. [runtime-preview/BUILD.md](runtime-preview/BUILD.md) documents the build modes, launch profiles, lighting presets and known limitations. |
| [tools/fps-report.py](tools/fps-report.py) and [tests/](tests/) | Frame timing, run comparison and parser tests. Keep report semantics aligned with [docs/runbook.md](docs/runbook.md). |
| [docs/](docs/) | Compatibility observations, reproducible measurements and testing procedures. Identify the exact runtime and report date when updating a result. |

Start changes in the smallest relevant layer. Startup, bottle handling and argument processing belong in the launch scripts. A runtime change belongs in a patch against the relevant pinned upstream. GPU or shader investigation may cross both the prepared runtime and the host backend. Identify the first boundary where expected and observed behavior diverge before choosing the fix.

The XCL and Runtime Preview builds have different upstreams and license terms. Preserve the notices and authorship of imported code and link the upstream commit or pull request. Keep generated upstream checkouts, temporary probes, local caches and build outputs out of the published repository. See [runtime-preview/SOURCE-NOTICES.md](runtime-preview/SOURCE-NOTICES.md) for the Runtime Preview lineage.

## Prepare a change

1. Identify which recipe and exact revision reproduce the symptom. Record the patch series, build profile, title profile, graphics backend and effective runtime settings. Confirm from the launcher and runtime logs that the intended options took effect.
2. Reproduce the smallest case that preserves the failure. Use a synthetic title, a focused shader or queue fixture, or a repeatable scene. For compatibility failures, state the expected behavior and capture the first meaningful divergence.
3. Make one behavior change per comparison. Keep diagnostics and behavior changes in separate commits when practical. Use opt-in switches for unproven runtime changes and preserve documented defaults unless the task changes them. Check the fallback path and transitions between old and new state.
4. Test the prepared source and record the evidence in the pull request. Update the relevant build or runbook section when a public option or procedure changes. Follow [CONTRIBUTING.md](CONTRIBUTING.md) for the submission workflow.

Run the relevant shell syntax checks from the repository root. The reporting tools use Python's standard library test runner.

```sh
zsh -n scripts/build.sh
zsh -n scripts/run.sh
bash -n runtime-preview/build.sh
bash -n runtime-preview/run.sh
python3 -m unittest discover -s tests
```

For XCL changes, prepare the pinned source and run the affected tests there. On a configured Apple Silicon host, run the synthetic title as documented in [CONTRIBUTING.md](CONTRIBUTING.md). Its exit status counts failed checks. Record the names of any failed checks and the Wine and D3DMetal versions.

```sh
scripts/build.sh --prepare
scripts/build.sh --test-title
scripts/run.sh --test-title
```

For Runtime Preview changes, prepare a new directory using the intended recipe. These examples run from `runtime-preview/`. Choose one command and a fresh destination. For compilation, targeted crate tests and the Wine `config` check, use the pinned toolchain and procedures in [BUILD.md](runtime-preview/BUILD.md).

```sh
./build.sh --prepare /path/to/new-preview-source
./build.sh --checkpoint --prepare /path/to/new-checkpoint-source
./build.sh --perf --prepare /path/to/new-perf-source
```

When updating a patch stack, check for patch drift against its recorded `UPSTREAM` pin. Apply the full `patches/series` in a new destination and verify that the prepared tree contains the intended changes. If changing a shared patch or default, check each affected recipe. A pin update changes the baseline and should be explicit in the change description.

## Graphics correctness and crash diagnosis

- **Shader semantics.** Build a minimal input that distinguishes the expected behavior from the bug. Exercise relevant precision, texture formats, lane masks, cross-lane reads and writes, and barrier behavior. On a graphics host, compare known buffer or pixel values after execution. Record unsupported shader cases with their exact failure reasons. A translation or pipeline creation result and a GPU execution result are separate evidence.
- **Resource identity and lifetime.** Track the actual host resource and view used at submission. A descriptor slot can be reused. A matching guest address alone is weak evidence. Preserve view ordering, alias visibility, resource states, and the lifetime of resources referenced by recorded or in-flight GPU work. If a change eliminates a copy, verify both CPU-to-GPU visibility and GPU-to-CPU visibility where the title expects them.
- **Queues and timing.** Identify which queue owns each operation and which fence completes it. Check that memory waits observe the effects of preceding writes and that pooled objects stay valid until completion. CPU time summed across threads can overlap. A host timing bucket that includes submission or waiting must be described as such when comparing it with hardware GPU timing.
- **Black frames.** Compare input and output at the same stage of the same frame. Useful checkpoints include a scene render target, the upscaler input and output, post-processing, and the selected presentation image. Check draw coverage and dropped draws, resource bindings and pending pipelines. Prewarming a pipeline tests readiness. If the image remains dark with the pipeline ready, check what it sampled and wrote. A few readback points describe those points, so include a window capture for a scene-wide claim.
- **Crashes and hangs.** Find the first relevant error and the last verified milestone. Record the process exit, fault location when available, memory pressure, and whether a configured guard or time limit ended the run. Preserve a clean fallback and verify recovery after a forced stop. Keep Wine bottle recovery scoped to the dedicated bottle and follow the conditions in [CONTRIBUTING.md](CONTRIBUTING.md).

Shader compilation caches and host pipeline libraries contain different artifacts. Record them separately for cold and warm tests. A change to shader output or binding conventions must use an appropriate cache identity so an older entry cannot hide it. Verify both a fresh translation and a cache hit when the change affects caching or compilation.

## Performance and gameplay verification

Match the baseline and candidate on the same Mac, runtime build where possible, title build, console profile, scene, save, camera and input sequence. Match rendering and output resolution, lighting and upscaler settings, cache state and any profiling switches. Record memory pressure and the degree of background shader or pipeline work. Keep runs isolated from one another and serialize tests that use the same bottle, save or GPU. Use a copied profile and a per-run cache directory for game tests.

Measure steady gameplay separately from boot, menus, loading, shader compilation, cinematics and stationary holds. Prefer frame times and include sample counts and a slow-frame measure as well as the median. For example, 10 FPS requires 100 ms per frame. Compare draw counts and dropped or pending work. If the candidate draws less, changes the camera, or reduces image quality, describe the change before attributing a speedup to the translation layer. For expensive experiments, look for a repeatable wall-clock gain before changing defaults.

Use logs and captures together when reporting a milestone. A boot receipt establishes execution. A visible 3D scene establishes rendering. For movement and camera controls, capture the neutral scene, the input interval, and the visible response, ideally with the runtime's input record alongside it. State the precise endpoint of a stability or progression test and whether it finished normally. A successful test of one scene supports a claim about that scene and configuration.

```sh
python3 tools/fps-report.py runs/baseline.log
python3 tools/fps-report.py --compare runs/baseline.log runs/candidate.log
```

Keep automated editing and analysis subject to the same checks as hand-written changes. A suggested optimization needs a reproducible mechanism, a targeted correctness check, and evidence that the requested behavior improved. State which checks ran and which results are still unverified. Keep failed experiments in the measurement record when they explain a regression or rule out a hypothesis.

## What to include in a review

- The symptom, affected recipe, relevant upstream reference and intended mechanism of the change.
- The prepared revision, build profile and relevant toolchain versions. For live runs, include the Mac chip and memory, macOS, Wine or CrossOver, and D3DMetal version.
- The commands and results for targeted tests, along with the scene, effective flags and cache state for benchmarks.
- For performance work, comparable old and new frame times, draw and drop counts, and visual or input evidence. For stability work, include the exit reason and the observed duration or milestone.
- A small sanitized log excerpt or capture that supports the claim. Remove private paths, identifiers and account information. Use synthetic or authorized inputs. Keep game files, executables, keys, shader dumps and other nonredistributable data out of commits and public reports.
