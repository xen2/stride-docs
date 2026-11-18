# Upgrading shaders from 4.4

Stride 4.5 changes the Stride shading language (SDSL) in these ways:

- The `stage` modifier is now written `module`. `stage` is an error.
- A class that holds only shared code or state (functions, constants, values, textures, streams) can be a **module**: declared `module X`, and used with `import X;`. A module exists once per effect.
- Using a member of a shader by name (`X.Member`) without inheriting `X` is an error. Before, the compiler silently mixed `X` in.
- Shaders and effects have namespaces, as C# types do. C# code and stored data name them by their full name, such as `Stride.Rendering.ComputeColor`.
- Engine methods that are meant to be extended say how their overrides combine and in which phase, so the order of the mixins no longer decides what runs first.
- Mistakes in a shader that Stride 4.4 let through, or reported as warnings, are errors.
- A macro applies to the whole effect, so only the top of a shader tree can hold one.

The project upgrade rewrites your shaders for you. The engine and the samples were upgraded by the same tool, and their compiled shaders are equivalent to the ones of 4.4.

## Upgrade your project

When you open a project made with an older Stride version in Game Studio, or run `stride upgrade`, the project upgrade now also rewrites its `.sdsl` and `.sdfx` files (the `bin` and `obj` folders are left out) and the names of shaders and effects in its C# code and its assets. It also deletes its effect logs. For how to upgrade a project, see [Update Stride](../manual/install-and-update/update-stride.md).

- If the backup of modified files is on, each file is copied to the backup before it's written. We still recommend committing your project to version control first.
- The upgrade log has a line for each upgraded file: `Upgraded shader [path] to the current shader language`, or `Upgraded [path] to the full names of shaders and effects` for an asset.
- What the upgrade can't do on its own is logged as a warning that starts with `Shader upgrade:`. See [What needs a person](#what-needs-a-person).
- Running the upgrade again changes nothing: each step only rewrites the old syntax.

> [!NOTE]
> The upgrade runs for a project made with Stride 4.5.4 or an older version, betas included. A project made with Stride 4.5.5 gets only the steps that came with Stride 4.5.6: steps 11 and up, and the deletion of effect logs.

## What the upgrade changes

### 1. `stage` becomes `module`

`stage` is replaced by `module` where it is a modifier. Comments, strings and names that contain the word are not changed.

```cs
// Before
shader MyWaterNode : ComputeColor, Texturing
{
    stage Texture2D WaterNormalMap;
    stage float WaveSpeed;
    ...
};

// After
shader MyWaterNode : ComputeColor
{
    import Texturing;

    module Texture2D WaterNormalMap;
    module float WaveSpeed;
    ...
};
```

A `module` member (before: `stage`) exists once per effect, also when the shader is used in several composition slots (for example several material properties). Its parameter key doesn't change.

### 2. Helpers of module code become `module`

Module code exists once per effect, so it can't call a method that exists once per slot. In a shader used in a composition, a plain method called from module code becomes `module`, and so do its overrides:

```cs
// Before
shader MyWindDisplacement : IMaterialSurface, TransformationBase, PositionStream4, Global
{
    stage float WindStrength;

    stage override void TransformPosition()
    {
        base.TransformPosition();
        streams.PositionWS.xyz += WindOffset(streams.PositionWS.xyz);
    }

    float3 WindOffset(float3 position) { ... }
};

// After
shader MyWindDisplacement : IMaterialSurface
{
    import TransformationBase;
    import PositionStream4;
    import Global;

    module float WindStrength;

    module override void TransformPosition()
    {
        base.TransformPosition();
        streams.PositionWS.xyz += WindOffset(streams.PositionWS.xyz);
    }

    module float3 WindOffset(float3 position) { ... }
};
```

A method of a composition type (such as `ComputeColor.Compute`) can't become `module`: each slot has its own. The upgrade logs a warning instead (see below).

### 3. Modules and `import`

A class becomes `module X` when all of these are true:

- it has no instance part: no member, `compose` or plain `override` / `abstract` method without `module`;
- it isn't the type of a `compose` slot;
- another class inherits it or uses it by name;
- no shader that may be used in a composition overrides one of its plain methods.

The `module` modifiers inside it are removed (every member of a module is a module member). Classes that nothing imports, such as post effects, stay shaders.

In every class, a parent that is a module (an engine module such as `ShaderBase`, `Texturing`, `Transformation`, `Global`, or one of your own) becomes an `import` line, first in the body.

A shader that a class uses by name without inheriting it is added to the class's parents. This is what the compiler did silently before.

```cs
// Before
shader MyBlurSettings
{
    stage float BlurRadius;
    stage float2 Direction;
};

shader MyBlur : ImageEffectShader, Texturing, MyBlurSettings
{
    stage override float4 Shading()
    {
        float4 color = Texture0.Sample(LinearSampler, streams.TexCoord + Direction * BlurRadius / MyMath.Pi);
        return color * (1.0 - MyVignette.Apply(streams.TexCoord));
    }
};

// After
module MyBlurSettings
{
    float BlurRadius;
    float2 Direction;
};

shader MyBlur : MyVignette
{
    import ImageEffectShader;
    import Texturing;
    import MyBlurSettings;

    module override float4 Shading()
    {
        float4 color = Texture0.Sample(LinearSampler, streams.TexCoord + Direction * BlurRadius / MyMath.Pi);
        return color * (1.0 - MyVignette.Apply(streams.TexCoord));
    }
};
```

Here `MyMath` (functions and constants only) became a module, so `MyMath.Pi` still works without an import. `MyVignette` has its own value (`Strength`), so it stays a shader and becomes a parent of `MyBlur`.

Writing a module in the parent list (`: Texturing`) still means import, so code that the upgrade didn't touch keeps working. Effects (`.sdfx`) and C# code that mix a module (`mixin ShaderBase`) also keep working. Parameter keys don't change.

### 4. `clone` is removed

The `clone` method modifier has had no effect since Stride 4.4. The upgrade removes it; code that still has it gets the error [SDSL0212](../diagnostics/SDSL0212.md).

In effects (`.sdfx`), `mixin clone` (no effect since Stride 4.4 either) and `mixin remove` (which couldn't be parsed since Stride 4.4) are removed too: the build stops with [SDSL0046](../diagnostics/SDSL0046.md) or [SDSL0047](../diagnostics/SDSL0047.md). The upgrade doesn't remove them: delete `mixin clone` lines, and instead of `mixin remove X`, mix `X` in only where it's needed (under an `if`).

### 5. `override` is added to implementations of abstract methods

Before Stride 4.4, a method could implement an abstract method of a parent without `override`; since Stride 4.5 it is the error [SDSL0114](../diagnostics/SDSL0114.md). The upgrade adds `override` when the method has the name and parameter types of an abstract method of a parent: a shader of your project, of a library given to the upgrade, or of the engine.

```cs
// Before
shader MyInvert : MyFilterBase
{
    float4 Filter(float4 color) { return float4(1 - color.rgb, color.a); }
};

// After
shader MyInvert : MyFilterBase
{
    override float4 Filter(float4 color) { return float4(1 - color.rgb, color.a); }
};
```


### 6. Shaders used in compositions import the streams of `ShaderBase` and `ComputeShaderBase`

`ShaderBase` and `ComputeShaderBase` are root-only (`[Root]`) in 4.5: a shader that imports them can't be used in a composition ([SDSL0119](../diagnostics/SDSL0119.md)). A shader used in a composition that inherited one of them only to read its streams now imports the module that holds the streams: `ShaderBaseStream` or `ComputeShaderStreams`. This applies to a shader that inherits a composition type (such as `ComputeColor`) and overrides no entry point (`VSMain`, `PSMain`, ...):

```cs
// Before
shader MyThreadColor : ComputeColor, ComputeShaderBase
{
    override float4 Compute() { return float4(streams.DispatchThreadId, 1); }
};

// After
shader MyThreadColor : ComputeColor
{
    import ComputeShaderStreams;

    override float4 Compute() { return float4(streams.DispatchThreadId, 1); }
};
```

### 7. `base.` before a helper of a module parent is removed

`base.X()` in instance code can't call a module method ([SDSL0208](../diagnostics/SDSL0208.md)). When `X` is a helper of a parent that becomes a module and nothing overrides `X`, `base.X()` meant the same as `X()`: the upgrade removes `base.`. If something overrides `X`, the upgrade logs a warning instead.

### 8. `[Root]` on shaders that rely on being mixed at the root

A module passes on only its own `[Root]`. When a class of your project becomes a module and was root-only through its parents (it inherited `ImageEffectShader`, for example), the shaders built on it are not root-only any more. A shader whose code relies on being mixed at the root (module code that reads its own members or calls its instance methods, or a plain `override` of a module method) gets `[Root]`, so it compiles without the errors [SDSL0204](../diagnostics/SDSL0204.md) to [SDSL0206](../diagnostics/SDSL0206.md). Shaders used in compositions don't get it.

```cs
// Before
shader MyFxBase : ImageEffectShader, Texturing
{
    stage float4 SampleInput(float2 uv) { return Texture0.Sample(LinearSampler, uv); }
};

shader MyTint : MyFxBase
{
    float4 Tint;
    stage override float4 Shading() { return SampleInput(streams.TexCoord) * Tint; }
};

// After
module MyFxBase
{
    import ImageEffectShader;
    import Texturing;

    float4 SampleInput(float2 uv) { return Texture0.Sample(LinearSampler, uv); }
};

[Root]
shader MyTint
{
    import MyFxBase;

    float4 Tint;
    module override float4 Shading() { return SampleInput(streams.TexCoord) * Tint; }
};
```

### 9. `compose` and `stream` move out of `cbuffer` and `rgroup` blocks

A composition or a stream isn't a member of a `cbuffer` or an `rgroup` ([SDSL0050](../diagnostics/SDSL0050.md)). Stride 4.3 compiled one declared inside such a block as declared after it, and Stride 4.4 stopped with an error. The upgrade moves it right after the block, so the shader works as in Stride 4.3:

```cs
// Before
shader MyShrink : ComputeColor
{
    rgroup PerMaterial
    {
        stage Texture2D ShrinkTexture;
        stage compose ComputeColor ShrinkStrength;
    }
    ...
};

// After
shader MyShrink : ComputeColor
{
    rgroup PerMaterial
    {
        module Texture2D ShrinkTexture;
    }
    module compose ComputeColor ShrinkStrength;
    ...
};
```

### 10. The value of a composition is removed

A composition gets its shader from the effect. A value written after its name (`compose ComputeColor Tint = float4(1, 1, 1, 1);`) was never used, and since Stride 4.5 it is the error [SDSL0236](../diagnostics/SDSL0236.md). The upgrade removes `= value` and keeps the rest of the line, attributes such as `[Default(...)]` included.

### 11. Names of shaders and effects

Shaders and effects now have namespaces, and a short name is only found through what is in scope (see [Namespaces](../manual/graphics/effects-and-shaders/shading-language/namespaces.md)). The upgrade changes:

- **Shaders:** it adds the `using` that a short name needs, when the namespace isn't one of the implicit ones. It writes key names in full where they are text (`[Link]`, generic arguments). It doesn't add a namespace to your shaders, so the names and keys of a shader with no namespace don't change.
- **C#:** literal names of shaders and effects, of the engine and of your project, and the names of their keys are rewritten in full. This goes by symbol, so only real uses of these APIs change. You can then replace a literal by the generated constant (`ComputeColorMultiplyKeys.ShaderName`), so that a typo is a build error.
- **Assets:** graphics compositors, materials, and the materials in scenes and prefabs are rewritten in full, in the project folder and in the asset folders of the package. Effect logs are deleted, with a copy in the backup: see [Effect logs](#effect-logs).

```cs
// Before
new ShaderClassSource("ComputeColorMultiply");
ParameterKeys.FindByName("Texturing.Texture0");

// After
new ShaderClassSource("Stride.Rendering.ComputeColorMultiply");
ParameterKeys.FindByName("Stride.Rendering.Texturing.Texture0");
```

See [Names in your C#](#names-in-your-c) for what the upgrade can't see.

### 12. `partial effect` becomes `effect`

`partial` on an effect did nothing. Since Stride 4.5 it is the error [SDSL0311](../diagnostics/SDSL0311.md), and the upgrade removes the word. If you wanted to add to an effect of another project, which didn't work before, use the new `extend effect` (see [Effects (SDFX)](../manual/graphics/effects-and-shaders/effect-language.md)).

### 13. Overrides of `module` methods get what extension points need

The upgrade adds a kind to the first definition of a `module` method that your project overrides, `[Drops]` to an override that drops what its class inherited, and `extend` where it says the same thing as `override`: see [Extension points](#extension-points).

### 14. `partial` on a class and `[Color]` on a member of another type are removed

`partial` before `shader` or `module` never meant anything ([SDSL0214](../diagnostics/SDSL0214.md)), and `[Color]` on a member that is not `float3` or `float4` never did anything ([SDSL0203](../diagnostics/SDSL0203.md)). Both are errors in Stride 4.5, and the upgrade removes them.

## Extension points

Engine methods that are meant to be extended now say how: [Extension points](../manual/graphics/effects-and-shaders/shading-language/extension-points.md).

`override` and `base` stay valid, and they don't need a phase.

What changes:

- **A phase always wins.** Engine classes now run in their phase, wherever they are mixed. An effect that mixes `NormalBase` before `TransformationBase` now runs `TransformationBase` first in `VSMain`. A class with no phase that you mix before the engine ran first in 4.4. It now runs after the phases, with no message, so writing `streams.Position` in `PreTransformPosition` does nothing, because the `World` phase has already read it. Say where it belongs: `[Phase(Morph)] extend TransformationBase.PreTransformPosition()`. A class that lists `NormalBase` first now generates its normal after `PreTransformPosition`, so what it writes to `meshNormal` there now counts.
- **An override without a phase runs outside every phase.** If it doesn't call `base`, it drops the overrides with a phase, also the ones mixed after it. What it drops is error [SDSL0216](../diagnostics/SDSL0216.md) in an effect built by the asset compiler, and information in a mix built at run time. Give it a phase and call `base` to keep them, or write `[Drops]` to drop them. The upgrade writes `[Drops(Mixins = { ... })]` when the class inherited what it drops: that parent is a module now, so it is imported, and the attribute keeps what 4.4 did.
- **Overriding a `module` method that is not an extension point** is error [SDSL0246](../diagnostics/SDSL0246.md). The upgrade adds the kind when the first definition is in your project: `pipeline` if an override calls `base`, else `replace`. The upgrade looks at one project at a time. A method whose first definition is in another project, in a package or in the engine gets no kind and no note, also when that project is in your solution. You get [SDSL0246](../diagnostics/SDSL0246.md) when the game compiles the effect, or in the asset build if the effect is in an effect log. If you own the method, add `pipeline` if an override calls `base`, else `replace`. If you don't, write `patch` instead of `override`, or ask its author to add the kind.
- **Two overrides of one phase whose order matters** need an order in an effect built by the asset compiler: [SDSL0137](../diagnostics/SDSL0137.md).
- **Two patches of one method** need an order in every effect: `[MixinOrder(After = ...)]` on the patch, or a block in the effect. [SDSL0148](../diagnostics/SDSL0148.md).
- **An effect orders and removes with blocks**, `extend shader Owner { [Unbind(Phases = { Shade })] PSMain; }`: see [Effects (SDFX)](../manual/graphics/effects-and-shaders/effect-language.md). They replace the `order`, `bind`, `unbind` and `phase` statements of the 4.5 previews, which are error [SDSL0251](../diagnostics/SDSL0251.md). Shaders from 4.4 have none of these.

The upgrade writes `extend` for overrides that have a phase, or that override a `replace` method, when `extend` can say the same thing. Shaders from 4.4 have no phases, so most of their other overrides stay as they are, and that is fine.

## Effect logs

The upgrade deletes each `.sdeffectlog` of your project, and writes which ones in its log. A copy is in the backup. Their entries were recorded on an older version, so they don't match the effects of 4.5, and building them reports errors in the shaders of that version.

With no log, a build compiles no shader, so it passes even when a shader has an error. Run your game once after the upgrade: errors in your shaders appear when each effect is compiled.

If your game uses precompiled effects, record them again: run the game with **Record used effects** on, then click **Import effects**, and build. A game with no run-time compiler (**Effect compiler: None**) doesn't find an effect that isn't in the log: the log has `Unable to find compiled shaders`, and the meshes are drawn without their material.

A log you record on 4.5 stays when you upgrade later.
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

## Names in your C#

Strings that name an effect or a shader take the full name. The upgrade changes them where it sees the API: `new DynamicEffectInstance("X")`, `new ShaderClassSource("X")`, `EffectName = "X"`. It can't see a string kept in a variable or passed through your own method, and `context.Mixin(mixin, "X")` in a builder. Those give [SDSL0307](../diagnostics/SDSL0307.md) when the effect is asked for, with the name to write.

The key of a member of an engine shader now has the full name of its class: `Transformation.World` is `Stride.Rendering.Transformation.World`. A key name that you build or compare as text gets no error when this happens. `$"Transformation.{name}"` and `key.Name.StartsWith("Texturing.")` just stop matching, so a value or a texture is no longer set, and the image changes. On Direct3D 12 a texture that isn't set can remove the device.

The upgrade writes a note in its log for a string that starts like such a key, with the full name. It changes no code, and a name that comes from data gets no note. Use the key where you can, as in `TransformationKeys.World.Name`, or write the full name. Keys of a class with no namespace, of `params` in an effect and of C# classes keep their names. A shader name built at run time does give an error, [SDSL0307](../diagnostics/SDSL0307.md), with the name to write.

A custom render feature that looks up `TransformationKeys.World` gets -1 in a skinned effect, which no longer reads it. Check for -1 before you write.

## What needs a person

- **Warning `Shader upgrade: ...: module code X calls Y.Z, an instance method; this works only if the class of X is mixed at the root, not used in a composition`.** Module code calls a method of a composition type. If the class is mixed at the root of the effect (a post effect, for example), nothing to do. If it's used in a composition slot, the effect fails to build with the "calls ... an instance method" error below: compute the value in instance code, or pass it as a parameter.
- **Warning `Shader upgrade: ...: class X not found in the text, not upgraded`.** The upgrade couldn't find the declaration of `X` in its file (for example, it's built by a macro). Upgrade it by hand.
- **Warning `Shader upgrade: ...: not upgraded, parsing failed: ...`.** The file has a syntax error that the upgrade can't read past. Fix the error the message gives, then make the changes of this page by hand: the upgrade runs once.
- **Warning `Shader upgrade: ...: X.F implements an abstract method but has overloads; add 'override' by hand`.** The upgrade adds `override` only when the name alone says which method is implemented. Add it to the overload that implements the abstract method ([SDSL0114](../diagnostics/SDSL0114.md) says which one).
- **Shaders from other packages.** The upgrade rewrites the `.sdsl` files of your project only. A package with shaders that was built for 4.4 doesn't work in 4.5, because its `.sdsl` files use `stage`: you get [SDSL0045](../diagnostics/SDSL0045.md) with a path in your NuGet folder. Ask the author for a version made for Stride 4.5, or copy the shader into your project. Library authors can run the same upgrade on their own shaders: see [Upgrading shaders outside a Stride project](#upgrading-shaders-outside-a-stride-project). Give a kind to each `module` method that others may override, because the upgrade only adds a kind when it sees an override.
- **Shaders loaded at run time.** `.sdsl` files that a tool reads from the disk at run time (vvvv's shader folders, for example) and code registered with `ShaderSourceManager.AddShaderSource` don't go through the project upgrade: the compiler reports the errors and warnings above for their old syntax. Upgrade the files with the same API (see below), and make a generator write the 4.5 syntax itself: `module` instead of `stage`, `import` of the streams modules in shaders used in compositions (see 6 above), `override` on implementations.
- **Warning `Shader upgrade: ...: X calls base.F() of the module M, and F is overridden: check the call by hand`.** `base.F()` in instance code can't call a module method ([SDSL0208](../diagnostics/SDSL0208.md)), and the upgrade can't remove `base.` because another class overrides `F`. Make the calling method `module`, or call `F()` if that is what you mean.
- **Errors [SDSL0204](../diagnostics/SDSL0204.md) to [SDSL0208](../diagnostics/SDSL0208.md) on a shader that an effect mixes only at its root.** The compiler checks the module rules for every shader that isn't root-only. A shader that inherits or imports `ShaderBase`, `ShadingBase`, `ImageEffectShader`, `ComputeShaderBase`, `SpriteBase` or `ParticleBase` directly is root-only. An effect feature that reaches them only through another module (for example through `TransformationBase`) isn't: if the effect mixes it only at its root, mark it `[Root]` (see [Shaders and modules](../manual/graphics/effects-and-shaders/shading-language/shaders-and-modules.md)). The upgrade adds `[Root]` itself when the shader's own code shows that it relies on the root (see 8 above); these errors remain for the other cases.
- **Names built at run time.** The upgrade only sees literals. A string that your code builds and that names a namespaced shader, effect or key must use the full name. A shader gives [SDSL0307](../diagnostics/SDSL0307.md), but a key name built as text stops matching with no error, so check code that uses `FindByName`, or that compares the name of a key. The upgrade writes a note in its log for a string that starts like such a key, with the full name.
- **Compiled plugins that aren't rebuilt.** They name engine shaders by their short names, and get [SDSL0307](../diagnostics/SDSL0307.md). Rebuild them against 4.5.
- **An `EffectCompiler` that your code creates.** Set `ImplicitUsings = ShaderImplicitUsings.List`, or engine classes named without a namespace aren't found ([SDSL0196](../diagnostics/SDSL0196.md)).
- **Classes added to the parents.** When a class used by name becomes a parent, it may come earlier in the order than the hidden mixin did before. This matters only if it overrides a method that another parent overrides too: check that the result is the one you want (see [Inheritance and overrides](../manual/graphics/effects-and-shaders/shading-language/shader-classes-mixins-and-inheritance.md)).

## Upgrading shaders outside a Stride project

The steps of the project upgrade are a public API in `Stride.Shaders.Parsers`, for shader libraries and tools that keep their `.sdsl` files outside a Stride project:

```cs
using Stride.Shaders.Parsing.Upgrade;

// path → text: the shaders to upgrade (one library)
var files = Directory.GetFiles("Shaders", "*.sdsl", SearchOption.AllDirectories)
    .ToDictionary(path => path, File.ReadAllText);
// path → text: the shaders of the libraries it builds on, already made for Stride 4.5
var dependencies = Directory.GetFiles("OtherLibrary/Shaders", "*.sdsl", SearchOption.AllDirectories)
    .ToDictionary(path => path, File.ReadAllText);

var result = ShaderUpgrader.Upgrade(files, EngineShaderClasses.Current, dependencies: dependencies);
foreach (var (path, text) in result.Files)
{
    if (text != files[path])
        File.WriteAllText(path, text);
}
foreach (var note in result.Notes)
    Console.WriteLine(note);
```

- `files` is upgraded as one project: a class becomes a module only when the files show how it's used, so give the whole library at once.
- `dependencies` are read to know their classes (parents, modules, abstract methods) and are never changed. Upgrade a library after the libraries it builds on: a dependency still written for an older version gets the note `dependency written for an older Stride version ...; upgrade it first`.
- `EngineShaderClasses.Current` describes the engine shaders of the Stride version that runs the upgrade.
- `Notes` has the same warnings as the project upgrade (see [What needs a person](#what-needs-a-person)). Running the upgrade on its result changes nothing.

## Removed engine shaders

Five engine shaders took the texture as a `Texture2D` generic parameter. Since Stride 4.4, no effect that used them could compile, so Stride 4.5 removes them, with their key classes (`ComputeColorTextureKeys`, ...). A shader that declares a texture or sampler generic parameter now gets the error [SDSL0118](../diagnostics/SDSL0118.md).

The engine materials didn't use these shaders: the **Texture** node of a material uses the `...Sampler` shaders below. If your code or your shaders used one of the removed shaders, use its replacement:

| Removed shader | Replacement |
|---|---|
| `ComputeColorTexture<TTexture, TStream>` | `ComputeColorTextureSampler<TTextureName, TStream, TSampler, TRgba>` |
| `ComputeColorTextureRepeat<TTexture, TStream, TFactor>` | `ComputeColorTextureScaledSampler<TTextureName, TStream, TSampler, TRgba, TScale>`, with `TScale = float2(TFactor, TFactor)` and a repeat sampler such as `Texturing.LinearRepeatSampler` |
| `ComputeColorTextureScaled<TTexture, TStream, TScale>` | `ComputeColorTextureScaledSampler<TTextureName, TStream, TSampler, TRgba, TScale>` |
| `ComputeColorTextureScaledOffset<TTexture, TStream, TScale, TOffset>` | `ComputeColorTextureScaledOffsetSampler<TTextureName, TStream, TSampler, TRgba, TScale, TOffset>` |
| `ComputeColorTextureDynamicScaledOffset<TTexture, TStream>` | `ComputeColorTextureScaledOffsetDynamicSampler<TTextureName, TStream, TSampler, TRgba, TScale, TOffset>`: `TScale` and `TOffset` are the names of your `float2` scale and offset keys (before: `ComputeColorTextureDynamicScaledOffsetKeys.Scale` and `.Offset`) |

The new generic parameters:

- `TTextureName`: the name of the texture key, for example `Material.DiffuseMap`. The texture is set with this key.
- `TSampler`: the name of the sampler key. The removed shaders used `Texturing.Sampler`.
- `TRgba`: the channels to read, `rgba` for all of them.

Stride 4.5 also removes these shaders, with their key classes, because nothing uses them: `BackgroundVelocity`, `ComputeColorScaler`, `FlattenLayers`, `GBuffer`, `SelectedSpriteShader`, `ShadingColor`, `SwapUV`, `TransformationMatrix`, `TransformationWVP`, `TransformationZero`, and the effects `BackgroundVelocityEffect` and `SelectedSprite`.

They have no replacement, and the project upgrade can't write one. To keep using one of them, copy it from the sources of Stride 4.4 into your project. A shader or a string that still names one gets a message that says it was removed.

The render stage named `GBuffer` in graphics compositors stays: it draws the `ZPrepass` child effect, not this shader.

## Error messages and their fix

| Message | Fix |
|---|---|
| ``SDSL0045: `stage` is now `module`: write `module`. The Stride project upgrade does it for the shaders of your project. A shader in a package needs a new version of that package.`` | Replace `stage` by `module`. |
| `SDSL0050: X is declared inside cbuffer B, but it is not a member of a cbuffer` | Move the `compose` or `stream` line out of the block (the project upgrade does it). |
| `SDSL0114: X.F() implements abstract method Y.F but is not marked 'override'` | Add `override` (the project upgrade does it). |
| ``SDSL0046: `mixin clone` was removed: it has had no effect since Stride 4.4. Delete the line.`` | Delete the `mixin clone` line in the `.sdfx` file. |
| ``SDSL0047: `mixin remove` was removed: ...`` | In the `.sdfx` file, mix the shader in only where it's needed (for example under an `if`) instead of removing it. |
| `X is a shader, not a module: inherit it to use its members, or make it a module` | You use a member of the shader `X` by name (`X.Member`) without inheriting it. Add `X` to the parents, or declare `X` as `module X` if it has no instance part. |
| ``X is a shader, not a module: inherit it (`: X`) instead of importing it`` | `import` takes a module. Move `X` to the parent list. |
| `module X can't inherit the shader Y: a module only imports other modules` | Make `X` a shader, or make `Y` a module. |
| ``Module code `A.F` reads `M`, an instance member of `A`. `A` is used in a composition (`slot`), where each instance has its own `M`. Mark `M` `module`, or mix `A` at the root.`` | Mark `M` with `module` if one value for the whole effect is right. Otherwise move the code that needs `M` out of module code. |
| ``Module code `A.F` calls `B.G`, an instance method. `A` is used in a composition (`slot`), which has an instance per slot. Mark `B.G` `module`, or mix `A` at the root.`` | Mark `B.G` with `module` (and its overrides). |
| ``Module code `A.F` calls `G`, an instance method of `A`. ...`` | Mark `G` with `module`. |
| ``` `A.F` overrides the module method `B.F` but is not `module`. ... Write `override module`. ``` | Write `module override` (or `override module`, the order of the keywords doesn't matter). |
| ``` `A.F` is `module` but overrides the instance method `B.F`. ... Remove `module`, or mark `B.F` `module`. ``` | Remove `module` from the override, or mark `B.F` with `module`. |
| ``` `base.F()` in `A.G`: `F` is a module method, called from instance code, and `A` is used in a composition. ...``` | Make `G` `module` (`override module` if it is an override). |
| ``'S' is a `module compose` declared by 'X', so it is one slot for the whole effect and must be supplied at the root.`` | Fill the slot `S` at the root of the effect (in the `.sdfx` file or the C# mixin tree), not in a composition. |
| `SDSL0118: X declares generic parameter 'T' of type Texture2D ...` | Declare the texture as a member, or use a `LinkType` generic parameter (see [Removed engine shaders](#removed-engine-shaders)). |
| `SDSL0119: X can only be mixed at the root of an effect ...` | `X` is root-only (a post effect, for example) but an effect uses it in a composition: mix it at the root, or compose a shader that doesn't inherit or import a root-only class. If `X` only reads the streams of `ShaderBase` or `ComputeShaderBase`, import `ShaderBaseStream` or `ComputeShaderStreams` instead. |
| ``SDSL0191: Y inherits from X, but shader X can't be found. ... Did you mean `using A;`?`` ([`SDSL0196`](../diagnostics/SDSL0196.md) for `import`, [`SDSL0190`](../diagnostics/SDSL0190.md) for a member type, `SDSL0110` for a use in code) | The name isn't in scope. Add the `using`, or write the full name (the project upgrade does it). |
| `SDSL0300: X is ambiguous: it can be A.X or B.X.` | Write the full name of the one to use. |
| `SDSL0301: X is declared twice: ...` | Rename one or put it in another namespace. To replace a shader, use `[Replaces]`. |
| `SDSL0302: effect X is declared twice: ...` | Rename one or put it in another namespace. |
| `SDSL0307: no shader or effect is named X. Did you mean A.X? ...` | Use `XKeys.ShaderName` or `XKeys.EffectName`, or the full name. |
| ``SDSL0308: F.sdsl doesn't declare X with the macros of this compile: ...`` | Define the macro in the effect that asks for `X`, or move `X` out of the `#if`. |
| `SDSL0311: 'partial effect' doesn't exist. ...` | Remove `partial` (the project upgrade does it). |
| `... X was removed from the engine in Stride 4.5. To keep using it, copy it from the sources of Stride 4.4 into your project.` (at the end of [`SDSL0190`](../diagnostics/SDSL0190.md), [`SDSL0191`](../diagnostics/SDSL0191.md), [`SDSL0196`](../diagnostics/SDSL0196.md), `SDSL0110`, or of `Shader X could not be found.` for a string) | See [Removed engine shaders](#removed-engine-shaders). |

## Mistakes are errors

What Stride 4.4 reported as a warning, or let through silently, is now an error when it is a mistake in the shader. Each message says the fix. [SDSL0219](../diagnostics/SDSL0219.md), [SDSL0220](../diagnostics/SDSL0220.md), [SDSL0232](../diagnostics/SDSL0232.md) and [SDSL0242](../diagnostics/SDSL0242.md) to [SDSL0244](../diagnostics/SDSL0244.md) stay warnings: they are about valid HLSL. To make them fail the build too, set the MSBuild property `StrideShaderWarningsAsErrors` to `true` in your project.

The messages about module code and overrides in the table above are errors when an effect is built (in the asset build or at run time) and uses the shader in a composition. The same mistakes are reported earlier, as the errors [SDSL0204](../diagnostics/SDSL0204.md) to [SDSL0208](../diagnostics/SDSL0208.md) below, when the `.sdsl` file compiles, unless the shader is root-only.

| Error | Fix |
|---|---|
| [`SDSL0200`](../diagnostics/SDSL0200.md): an `override` of a method that no parent declares | Inherit the class that declares the method, or remove `override`. |
| [`SDSL0201`](../diagnostics/SDSL0201.md): a `base` call with no inherited method | Inherit the class that declares the method, or call it without `base.`. |
| [`SDSL0202`](../diagnostics/SDSL0202.md): more generic values than the class has parameters | Remove the extra values. |
| [`SDSL0203`](../diagnostics/SDSL0203.md): `[Color]` on a member that is not `float3` or `float4` | Remove `[Color]`, or change the type (the project upgrade removes it). |
| [`SDSL0204`](../diagnostics/SDSL0204.md): module code reads an instance member | Mark the member `module`, or mark the shader `[Root]` if the effect mixes it only at its root. |
| [`SDSL0205`](../diagnostics/SDSL0205.md): module code calls an instance method | Mark the called method `module`, or mark the shader `[Root]` if the effect mixes it only at its root. |
| [`SDSL0206`](../diagnostics/SDSL0206.md): a plain `override` of a module method | Write `module override`, or mark the shader `[Root]` if the effect mixes it only at its root. |
| [`SDSL0207`](../diagnostics/SDSL0207.md): an `override module` of an instance method | Remove `module` from the override, or mark the shader `[Root]` if the effect mixes it only at its root. |
| [`SDSL0208`](../diagnostics/SDSL0208.md): instance code calls a module method through `base` | Make the calling method `module` (`module override` if it is an override), or mark the shader `[Root]`. |
| [`SDSL0209`](../diagnostics/SDSL0209.md): state in a `[Pure]` module | Remove `[Pure]`, or move the state to another module. |
| [`SDSL0210`](../diagnostics/SDSL0210.md): a `[Pure]` module uses a module that isn't pure | Mark that module `[Pure]` if it can be, or remove `[Pure]`. |
| [`SDSL0211`](../diagnostics/SDSL0211.md): `[Pure]` on a shader | Remove `[Pure]`, or make the shader a module. |
| [`SDSL0212`](../diagnostics/SDSL0212.md): `clone` has no effect | Remove `clone` (the project upgrade does it). |
| [`SDSL0213`](../diagnostics/SDSL0213.md): an operation that works only in a pixel shader (`ddx`, `clip`, `Sample`...) in a `[Pure]` module | Compute the value in the caller and pass it as a parameter, use `SampleLevel` or `SampleGrad`, or remove `[Pure]`. |
| [`SDSL0214`](../diagnostics/SDSL0214.md): `partial` on a shader or a module | Remove `partial` (the project upgrade does it). |
| [`SDSL0215`](../diagnostics/SDSL0215.md): two structs of one name in a class | Rename one of them. |
| [`SDSL0217`](../diagnostics/SDSL0217.md): a misspelled attribute on a shader, a module or a `cbuffer` | Fix its spelling. Other unknown attributes on a class stay allowed, because libraries read attributes of their own. |
| [`SDSL0236`](../diagnostics/SDSL0236.md): the value of a composition is ignored | Remove `= value` (the project upgrade does it). |
| [`SDSL0136`](../diagnostics/SDSL0136.md): an `extend` that drops others without saying so, in an effect built by the asset compiler. Two overrides of one `replace` method become this after the upgrade, because one drops the other | Call `next`, or write `[Drops]`. |
| [`SDSL0146`](../diagnostics/SDSL0146.md): classes that inherit, import or use each other in a cycle | Remove one of the links. Code that two classes share goes to a class or a module they both use. |
| [`SDSL0147`](../diagnostics/SDSL0147.md): a preprocessor condition that reads a generic parameter | Use a plain `if`, or a separate class for each value. |
| [`SDSL0152`](../diagnostics/SDSL0152.md): a stream name that a class reaches twice | Rename one of the streams, or declare it once in a class that both inherit. |
| [`SDSL0153`](../diagnostics/SDSL0153.md): a class in front of the name of a stream | Write `streams.name`. |
| [`SDSL0159`](../diagnostics/SDSL0159.md): `streams.name` with no stream of that name | Declare the stream, inherit or import its class, or remove `streams.`. |
| [`SDSL0216`](../diagnostics/SDSL0216.md): an override that drops others without saying so, in an effect built by the asset compiler | Call `base`, or write `[Drops]` (the project upgrade writes it when the class inherited what it drops). |
| [`SDSL0248`](../diagnostics/SDSL0248.md): a call that two unrelated classes could answer | Write the class in front of the call, such as `MyColors.Tint(color)`. Their order decided which one ran before. |

## Learn more

The manual of Stride 4.5 describes shaders, modules, `import` and `module` members in the [Shading language (SDSL)](../manual/graphics/effects-and-shaders/shading-language/index.md) section.
