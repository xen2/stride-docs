# Shader compiler tests

This page is for contributors who change the SDSL shader compiler (`sources/shaders/`), or engine shaders and rendering code that change which shaders are compiled.

The tests are in `sources/shaders/Stride.Shaders.Tests`. They need only the compiler and the `.sdsl` files, so they run on Windows, Linux and macOS.

## Point tests and RenderTests

Most tests check one feature of the compiler: parsing, compile errors, the mixer. The shaders in `sources/shaders/assets/SDSL/RenderTests` are small effects that `RenderingTests` compiles and draws with Direct3D 11 and Vulkan (a backend that isn't available is skipped). Add one when you fix a bug or add a language feature: it also becomes an entry of the corpus snapshot below.

```bash
dotnet test sources/shaders/Stride.Shaders.Tests
```

## The shader corpus snapshot

A compiler change can change engine shaders that no point test compiles. The **corpus snapshot** catches this: `ShaderCorpusTests` compiles every known shader permutation with the current compiler, and checks that the output didn't change.

So a compiler PR either says "snapshot unchanged", or lists which permutations changed, and why.

The files are in `sources/shaders/Stride.Shaders.Tests/Corpus`:

- `corpus.json`: the permutations. Each entry is the mixin tree given to the shader mixer (after `.sdfx` evaluation, before the platform macros), its effect name, graphics profile, and where it was seen. The RenderTests shaders are added as extra entries.
- `snapshot.txt`: one line per permutation and platform, with a hash of each part of its output:
  - `strict`: the SPIR-V of the mixer, normalized (debug info removed, IDs renumbered by first use, types and constants named after their content);
  - `legal`: the same after legalization (inlining, dead code removal). When only `strict` changed, the code is the same and only its order moved;
  - `reflection`: constant buffers, resource bindings, input attributes;
  - `explain`: how the mixer linked the effect: the mixins of each node in order, the members placed in each node, and the override chain of each method;
  - `error`: instead of the four above, for a permutation that doesn't compile.

Each permutation is compiled for Direct3D 11 and for Vulkan. Vulkan uses one unified binding scheme, which Direct3D 12, Android and iOS also use, so the two cover every platform's binding layout.

Only hashes are in the repository. Every run writes the readable files (`.spvasm`, `.legal.spvasm`, `.reflection.txt`, `.explain.txt`, `.error.txt`) to `ShaderCorpus/<platform>/` next to the test assembly.

### Tests

| Test | Runs | Does |
|---|---|---|
| `CompilerOutputUnchanged` | always, also on CI | compares the output with `snapshot.txt` |
| `CorpusEntriesRoundTrip` | always | checks that the corpus entries read back with the same hashes |
| `SaveBaseline` | explicit | saves a full run to `ShaderCorpus.baseline/` next to the test assembly |
| `MatchesBaseline` | explicit | compares a run with that baseline, writes `ShaderCorpus/report.txt` and `ShaderCorpus/diff.txt` |
| `UpdateSnapshot` | explicit | rewrites `snapshot.txt` |

Explicit tests run when you select them in the IDE, or from the command line with `-- xUnit.Explicit=on`.

### Workflow for a compiler change

1. Before you change anything, save a baseline:

    ```bash
    dotnet test sources/shaders/Stride.Shaders.Tests --filter "FullyQualifiedName~ShaderCorpusTests.SaveBaseline" -- xUnit.Explicit=on
    ```

2. Change the compiler and rebuild.

3. Compare with the baseline:

    ```bash
    dotnet test sources/shaders/Stride.Shaders.Tests --filter "FullyQualifiedName~ShaderCorpusTests.MatchesBaseline" -- xUnit.Explicit=on
    ```

4. Read `ShaderCorpus/report.txt` and `ShaderCorpus/diff.txt` (a git diff of every changed file). The explain dump makes most diffs easy to read: "this shader moved before that one" instead of hundreds of lines of SPIR-V.

5. If the changes are expected, run `UpdateSnapshot` and commit `snapshot.txt`:

    ```bash
    dotnet test sources/shaders/Stride.Shaders.Tests --filter "FullyQualifiedName~ShaderCorpusTests.UpdateSnapshot" -- xUnit.Explicit=on
    ```

    In the PR description, say which permutations changed and why.

Legalization keeps functions and the order of declarations, so `legal` also changes when a function only moved to another node, or when two copies of a function became one. For this case, baseline runs also compare the code after `spirv-opt -O` with canonical IDs: the report says "same optimized code" when only the layout changed, or "optimized code differs too". These optimized files are not in the snapshot.

A permutation that already failed in the snapshot and still fails is **stale** (old data, for example a shader that gained a generic parameter since). It is reported, not a failure. A permutation that compiled before and fails now is a failure.

### Faster runs while you work

- `STRIDE_SHADER_CORPUS_PLATFORMS=Vulkan` (a comma-separated list) checks only some platforms.
- `STRIDE_SHADER_CORPUS_FILTER=<text>` checks only the permutations whose id contains the text.

`UpdateSnapshot` refuses to run with either of them, so the snapshot always covers everything.

### On CI

`CompilerOutputUnchanged` runs in the existing Simple test lanes, on every PR that touches `sources/shaders/**`: the Windows lane checks Direct3D 11, the Linux lane checks Vulkan. It takes about 1 to 2 minutes per lane.

## Refreshing the corpus

The corpus is rebuilt from empty, out of what the current code really compiles. A refresh is expected after rendering or material changes, which change the permutations the engine builds. A compiler PR shouldn't need one.

The **Refresh Shader Corpus** workflow (`.github/workflows/refresh-shader-corpus.yml`, started by hand) does it:

1. It runs the samples, the editor and the game test suites with capture on (their `capture-shader-corpus` input). `RenderFeatureTests` in `Stride.Engine.Tests` covers the features that no sample uses, such as MSAA, ambient occlusion, depth of field, local reflections, temporal anti-aliasing and cubemap backgrounds.
2. It adds the two `Stride.Graphics` effect logs. They describe the bytecode embedded in `SpriteBatch`, `UIBatch`, `PrimitiveQuad` and the text renderer, which the runtime never compiles.
3. It writes `corpus.json` with the `Stride.Shaders.CorpusCapture` tool, and updates `snapshot.txt`.
4. It uploads both files as the `shader-corpus-refresh` artifact, or opens a PR (`open-pr` input). The run summary lists the permutations per source.

The other effect logs (samples, templates, editor) are not a source: they record what games compiled at some point, not what the current code builds. To cover a feature that no capture run reaches, add a scenario (a test or a sample), not a log.

### Capture locally

With `STRIDE_SHADER_CORPUS_CAPTURE=<folder>`, every effect compile writes its mixin tree to that folder. `STRIDE_SHADER_CORPUS_TAG=<name>` sets the source name recorded with it (by default, the entry assembly name, such as the game or `Stride.GameStudio`).

Then build `corpus.json` from the captures and the two effect logs, and run `UpdateSnapshot`:

```bash
dotnet run --project sources/tools/Stride.Shaders.CorpusCapture -- --logs sources/engine/Stride.Graphics --capture <folder> [--capture <folder>...]
```

`--logs` takes effect log files or folders, `--capture` capture folders (searched recursively).

## See also

* [Shading language (SDSL)](../../manual/graphics/effects-and-shaders/shading-language/index.md)
* [Hot reloading editor shaders](hot-reloading-shaders.md)
