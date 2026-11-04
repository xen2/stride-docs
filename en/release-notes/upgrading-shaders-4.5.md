# Upgrading shaders from 4.4

Opening a project in Stride 4.5 upgrades its shaders, after a backup. This page says what changes in extension points, and what can change in what your shaders do.

## Extension points

Engine methods that are meant to be extended now say how: [Extension points](../manual/graphics/effects-and-shaders/shading-language/extension-points.md).

`override` and `base` stay valid, and they don't need a phase.

What changes:

- **A phase always wins.** Engine classes now run in their phase, wherever they are mixed. An effect that mixes `NormalBase` before `TransformationBase` now runs `TransformationBase` first in `VSMain`.
- **An override without a phase runs outside every phase.** If it doesn't call `base`, it drops the overrides with a phase, also the ones mixed after it. What it drops is error [SDSL0216](../diagnostics/SDSL0216.md) in an effect built by the asset compiler, and information in a mix built at run time. Give it a phase and call `base` to keep them, or write `[Replaces]` to drop them. The upgrade writes `[Replaces]` when the class inherited what it drops: that parent is a module now, so it is imported, and the attribute keeps what 4.4 did.
- **Overriding a `module` method that is not an extension point** is error [SDSL0246](../diagnostics/SDSL0246.md). The upgrade adds the kind when the first definition is in your project: `[Pipeline]` if an override calls `base`, else `[Replace]`. If it is in a library, the upgrade notes say so: write `patch` instead of `override`, or ask its author to add the kind.
- **Two overrides of one phase whose order matters** need an order in an effect built by the asset compiler: [SDSL0137](../diagnostics/SDSL0137.md).
- **Two patches of one method** need an order in every effect: `after` on the patch, or an `order` rule in the effect. [SDSL0148](../diagnostics/SDSL0148.md).

The upgrade writes `extend` for overrides that have a phase, or that override a `[Replace]` method, when `extend` can say the same thing. Shaders from 4.4 have no phases, so most of their other overrides stay as they are, and that is fine.

## Mistakes are errors

What Stride 4.4 reported as a warning, or let through silently, is now an error when it is a mistake in the shader. The upgrade fixes the ones a tool can fix:

| Code | Mistake | The upgrade |
|---|---|---|
| [SDSL0212](../diagnostics/SDSL0212.md) | `clone` on a method | removes it |
| [SDSL0214](../diagnostics/SDSL0214.md) | `partial` on a class | removes it |
| [SDSL0236](../diagnostics/SDSL0236.md) | a value on a `compose` | removes it |
| [SDSL0203](../diagnostics/SDSL0203.md) | `[Color]` on a member that is not `float3` or `float4` | removes it |
| [SDSL0204](../diagnostics/SDSL0204.md) to [SDSL0208](../diagnostics/SDSL0208.md) | `module` code that uses instance members, and overrides of the wrong kind | marks what it can as `module`, and adds `[Root]` to a class nothing uses in a composition |
| [SDSL0200](../diagnostics/SDSL0200.md), [SDSL0201](../diagnostics/SDSL0201.md), [SDSL0202](../diagnostics/SDSL0202.md) | an `override` or a `base` call with no inherited method, too many generic values | you fix it |
| [SDSL0209](../diagnostics/SDSL0209.md), [SDSL0210](../diagnostics/SDSL0210.md), [SDSL0211](../diagnostics/SDSL0211.md), [SDSL0213](../diagnostics/SDSL0213.md) | `[Pure]` rules | you fix it |
| [SDSL0152](../diagnostics/SDSL0152.md), [SDSL0153](../diagnostics/SDSL0153.md) | a stream name that a class reaches twice, and a class in front of the name of a stream | you fix it |
| [SDSL0215](../diagnostics/SDSL0215.md) | two structs of one name in a class | you fix it |
| [SDSL0136](../diagnostics/SDSL0136.md) | an `extend` that drops others without saying so, in an effect built by the asset compiler. Two overrides of one `[Replace]` method become this after the upgrade, because one drops the other | you fix it: `replaces` or `next` |
| [SDSL0216](../diagnostics/SDSL0216.md) | an override that drops others without saying so, in an effect built by the asset compiler | writes `[Replaces]` when the class inherited what it drops; otherwise you fix it: `[Replaces]` or `base` |
| [SDSL0217](../diagnostics/SDSL0217.md) | an attribute misspelled on a class or a `cbuffer` | you fix it |

A class that doesn't have to work in a composition can say so with `[Root]`: the rules of SDSL0204 to SDSL0208 don't apply to it. The upgrade adds it to a class that needs it, when nothing in the project uses that class in a composition and the upgrade knows all its parents. The upgrade can't see what C# or a tool does: if one of them mixes that class into a composition, the compiler reports [SDSL0119](../diagnostics/SDSL0119.md), and you have to change the module code.

[SDSL0219](../diagnostics/SDSL0219.md), [SDSL0220](../diagnostics/SDSL0220.md), [SDSL0232](../diagnostics/SDSL0232.md) and SDSL0242 to SDSL0244 stay warnings: they are about valid HLSL. Other unknown attributes on a class stay allowed, because libraries read attributes of their own.

## Macros belong to the effect

A macro now applies to the whole effect, so only the top of a shader tree can hold one. `ShaderMixinSource` no longer has `Macros` or `AddMacro`.

| In Stride 4.4 | In Stride 4.5 |
|---|---|
| `mixin macro` at the top level of an effect | The same |
| `mixin macro` inside a `mixin compose` | Error [SDSL0149](../diagnostics/SDSL0149.md) |
| `root.Macros.Add(...)` on the root of a tree in C# | `new EffectSource(root) { Macros = { ... } }` |
| `compiler.Compile(mixin, parameters)` | `compiler.Compile(new EffectSource(mixin), parameters)` |
| `ShaderMixinManager.Generate(...)` gives a `ShaderMixinSource` | It gives an `EffectSource`, whose `Root` is the tree |

A macro on a node inside the tree has three replacements:

- For a value that changes per node, use a generic.
- For a different shape, write a separate class.
- For a value of the whole effect, set the macro at the top level.

The upgrade changes C# where it can be certain:

- `mixin.AddMacro(a, b)` becomes `context.AddMacro(a, b)`, where `mixin` is the parameter of `IShaderMixinBuilder.Generate`.
- An override of `Compile` on an effect compiler takes an `EffectSource`, and `mixinTree.Member` becomes `mixinTree.Root.Member`.
- `compiler.Compile(x, parameters)` wraps `x` when its type says it is a mixin or a class.
- `ShaderMixinManager.Generate(...).Member` becomes `ShaderMixinManager.Generate(...).Root.Member`.

Other uses of `Macros` and `AddMacro` on a mixin don't compile, because only you know if that mixin is the root.

The number of render targets and of samples now come from `StrideEffectBaseKeys.RenderTargetCount` and `StrideEffectBaseKeys.MultisampleCount`, and the tessellation macro from `MaterialKeys.TessellationInputControlPoints`. `StrideEffectBase` turns them into the same macros as before. An effect of yours that mixes `StrideEffectBaseKeys.RenderTargetExtensions` without `StrideEffectBase` has to set these macros itself.

The voxel classes changed:

- `VoxelStorageClipmapShader` is abstract now. Mix `VoxelStorageClipmapSingleShader` or `VoxelStorageClipmapAllShader`, or you get [SDSL0166](../diagnostics/SDSL0166.md).
- `VoxelMarchCone` has a fifth generic, `bool fast`. With `Fast` on, an effect has one more sampler binding that no code uses.
- `IVoxelMarchMethod` and `IVoxelMarchSet` lose `attrID`. The upgrade removes the argument at call sites when it is a plain value. A class that implements them doesn't compile until you remove the parameter.

**Effect logs.** Effect logs from 4.4 no longer match, and the upgrade doesn't convert them. Most entries still build, but the game doesn't use them and compiles those effects when it runs. A platform that can't compile at run time, such as iOS or Android, stops with `Unable to find compiled shaders`. An entry that names a class that changed stops the build with an error that names the log.

**What to do:** delete each `.sdeffectlog` in your project, run the game with **Record used effects** on, or use **Import effects**, and build again. Do this before you ship to a platform that can't compile at run time.
