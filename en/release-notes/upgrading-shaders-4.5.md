# Upgrading shaders from 4.4

Stride 4.5 changes the Stride shading language (SDSL) in three ways:

- The `stage` modifier is now written `module`. `stage` is an error.
- A class that holds only shared code or state (functions, constants, values, textures, streams) can be a **module**: declared `module X`, and used with `import X;`. A module exists once per effect.
- Using a member of a shader by name (`X.Member`) without inheriting `X` is an error. Before, the compiler silently mixed `X` in.

The project upgrade rewrites your shaders for you. The engine and the samples were upgraded by the same tool, and their compiled shaders are equivalent to the ones of 4.4.

## Upgrade your project

When you open a project made with an older Stride version in Game Studio, or run `stride upgrade`, the project upgrade now also rewrites its `.sdsl` files (the `bin` and `obj` folders are left out). For how to upgrade a project, see [Update Stride](../manual/install-and-update/update-stride.md).

- If the backup of modified files is on, each `.sdsl` file is copied to the backup before it's written. We still recommend committing your project to version control first.
- The upgrade log has a line for each upgraded file: `Upgraded shader [path] to the current shader language`.
- What the upgrade can't do on its own is logged as a warning that starts with `Shader upgrade:`. See [What needs a person](#what-needs-a-person).
- Running the upgrade again changes nothing: each step only rewrites the old syntax.

> [!NOTE]
> Projects that were already upgraded to a 4.5 beta before these changes (4.5.0-beta01) are not upgraded again automatically. Fix them by hand with the [error messages](#error-messages-and-their-fix) below.

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

The `clone` method modifier has had no effect since Stride 4.4. The upgrade removes it; code that still has it gets the warning SDSL0212.

In effects (`.sdfx`), `mixin clone` (no effect since Stride 4.4 either) and `mixin remove` (which couldn't be parsed since Stride 4.4) are removed too: the build stops with SDSL0046 or SDSL0047. The upgrade doesn't change `.sdfx` files: delete `mixin clone` lines, and instead of `mixin remove X`, mix `X` in only where it's needed (under an `if`).

### 5. `override` is added to implementations of abstract methods

Before Stride 4.4, a method could implement an abstract method of a parent without `override`; since Stride 4.5 it is the error SDSL0114. The upgrade adds `override` when the method has the name and parameter types of an abstract method of a parent: a shader of your project, of a library given to the upgrade, or of the engine.

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

`ShaderBase` and `ComputeShaderBase` are root-only (`[Root]`) in 4.5: a shader that imports them can't be used in a composition (SDSL0119). A shader used in a composition that inherited one of them only to read its streams now imports the module that holds the streams: `ShaderBaseStream` or `ComputeShaderStreams`. This applies to a shader that inherits a composition type (such as `ComputeColor`) and overrides no entry point (`VSMain`, `PSMain`, ...):

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

`base.X()` in instance code can't call a module method (SDSL0208). When `X` is a helper of a parent that becomes a module and nothing overrides `X`, `base.X()` meant the same as `X()`: the upgrade removes `base.`. If something overrides `X`, the upgrade logs a warning instead.

### 8. `[Root]` on shaders that rely on being mixed at the root

A module passes on only its own `[Root]`. When a class of your project becomes a module and was root-only through its parents (it inherited `ImageEffectShader`, for example), the shaders built on it are not root-only any more. A shader whose code relies on being mixed at the root (module code that reads its own members or calls its instance methods, or a plain `override` of a module method) gets `[Root]`, so it compiles without the warnings SDSL0204 to SDSL0206. Shaders used in compositions don't get it.

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

A composition or a stream isn't a member of a `cbuffer` or an `rgroup` (SDSL0050). Stride 4.3 compiled one declared inside such a block as declared after it, and Stride 4.4 stopped with an error. The upgrade moves it right after the block, so the shader works as in Stride 4.3:

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

A composition gets its shader from the effect. A value written after its name (`compose ComputeColor Tint = float4(1, 1, 1, 1);`) was never used, and since Stride 4.5 the compiler reports the warning SDSL0236 for it. The upgrade removes `= value` and keeps the rest of the line, attributes such as `[Default(...)]` included.

### 11. Names of shaders and effects

Shaders and effects now have namespaces, and a short name is only found through what is in scope ([Namespaces](../manual/graphics/effects-and-shaders/shading-language/namespaces.md)). The upgrade changes:

- **Shaders:** it adds the `using` that a short name needs, when the namespace isn't one of the implicit ones. It writes key names in full where they are text (`[Link]`, generic arguments). It doesn't add a namespace to your shaders, so the names and keys of a shader with no namespace don't change.
- **C#:** literal names of shaders and effects, of the engine and of your project, and the names of their keys are rewritten in full. This goes by symbol, so only real uses of these APIs change. You can then replace a literal by the generated constant (`ComputeColorMultiplyKeys.ShaderName`), so that a typo is a build error.
- **Assets:** effect logs, graphics compositors, materials, and the materials in scenes and prefabs are rewritten in full, in the project folder and in the asset folders of the package.

```cs
// Before
new ShaderClassSource("ComputeColorMultiply");
ParameterKeys.FindByName("Texturing.Texture0");

// After
new ShaderClassSource("Stride.Rendering.ComputeColorMultiply");
ParameterKeys.FindByName("Stride.Rendering.Texturing.Texture0");
```

## What needs a person

- **Warning `Shader upgrade: ...: module code X calls Y.Z, an instance method; this works only if the class of X is mixed at the root, not used in a composition`.** Module code calls a method of a composition type. If the class is mixed at the root of the effect (a post effect, for example), nothing to do. If it's used in a composition slot, the effect fails to build with the "calls ... an instance method" error below: compute the value in instance code, or pass it as a parameter.
- **Warning `Shader upgrade: ...: class X not found in the text, not upgraded`.** The upgrade couldn't find the declaration of `X` in its file (for example, it's built by a macro). Upgrade it by hand.
- **Shaders from other packages.** The upgrade rewrites the `.sdsl` files of your project only. Shaders that come from a NuGet package or another library need a version of that package made for Stride 4.5. Library authors can run the same upgrade on their own shaders: see [Upgrading shaders outside a Stride project](#upgrading-shaders-outside-a-stride-project).
- **Shaders loaded at run time.** `.sdsl` files that a tool reads from the disk at run time (vvvv's shader folders, for example) and code registered with `ShaderSourceManager.AddShaderSource` don't go through the project upgrade: the compiler reports the errors and warnings above for their old syntax. Upgrade the files with the same API (see below), and make a generator write the 4.5 syntax itself: `module` instead of `stage`, `import` of the streams modules in shaders used in compositions (see 6 above), `override` on implementations.
- **Names built at run time.** The upgrade only sees literals. A string that your code builds and that names a namespaced shader, effect or key must use the full name. A shader gives SDSL0307, but a key name built as text stops matching with no error, so check code that uses `FindByName`, or that compares the name of a key.
- **Compiled plugins that aren't rebuilt.** They name engine shaders by their short names, and get SDSL0307. Rebuild them against 4.5.
- **Warning `Shader upgrade: ...: X calls base.F() of the module M, and F is overridden: check the call by hand`.** `base.F()` in instance code can't call a module method (SDSL0208), and the upgrade can't remove `base.` because another class overrides `F`. Make the calling method `module`, or call `F()` if that is what you mean.
- **Warnings SDSL0204 to SDSL0208 on a shader that an effect mixes only at its root.** The compiler checks the module rules for every shader that isn't root-only. A shader that inherits or imports `ShaderBase`, `ShadingBase`, `ImageEffectShader`, `ComputeShaderBase`, `SpriteBase` or `ParticleBase` directly is root-only. An effect feature that reaches them only through another module (for example through `TransformationBase`) isn't: if the effect mixes it only at its root, mark it `[Root]` (see "Shaders and modules" in the manual of Stride 4.5). The upgrade adds `[Root]` itself when the shader's own code shows that it relies on the root (see 8 above); these warnings remain for the other cases.
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

Five engine shaders took the texture as a `Texture2D` generic parameter. Since Stride 4.4, no effect that used them could compile, so Stride 4.5 removes them, with their key classes (`ComputeColorTextureKeys`, ...). A shader that declares a texture or sampler generic parameter now gets the error SDSL0118.

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

## Error messages and their fix

| Message | Fix |
|---|---|
| ``SDSL0045: `stage` is now `module`: write `module` (the Stride project upgrade does it)`` | Replace `stage` by `module`. |
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
| ``SDSL0191: Y inherits from X, but shader X can't be found. ... Did you mean `using A;`?`` (`SDSL0196` for `import`, `SDSL0190` for a member type, `SDSL0110` for a use in code) | The name isn't in scope. Add the `using`, or write the full name (the project upgrade does it). |
| `SDSL0300: X is ambiguous: it can be A.X or B.X.` | Write the full name of the one to use. |
| `SDSL0301: X is declared twice: ...` | Rename one or put it in another namespace. To replace a shader, use `[Replaces]`. |
| `SDSL0302: effect X is declared twice: ...` | Rename one or put it in another namespace. |
| `SDSL0307: no shader or effect is named X. Did you mean A.X? ...` | Use `XKeys.ShaderName` or `XKeys.EffectName`, or the full name. |

The messages about module code and overrides above are errors when an effect is built (in the asset build or at run time) and uses the shader in a composition. The same mistakes are reported earlier, as the warnings below, when the `.sdsl` file compiles, unless the shader is root-only. Each warning message also says the fix.

| Warning | Fix |
|---|---|
| `SDSL0204`: module code reads an instance member | Mark the member `module`, or mark the shader `[Root]` if the effect mixes it only at its root. |
| `SDSL0205`: module code calls an instance method | Mark the called method `module`, or mark the shader `[Root]` if the effect mixes it only at its root. |
| `SDSL0206`: a plain `override` of a module method | Write `module override`, or mark the shader `[Root]` if the effect mixes it only at its root. |
| `SDSL0207`: an `override module` of an instance method | Remove `module` from the override, or mark the shader `[Root]` if the effect mixes it only at its root. |
| `SDSL0208`: instance code calls a module method through `base` | Make the calling method `module` (`module override` if it is an override), or mark the shader `[Root]`. |
| `SDSL0212`: `clone` has no effect | Remove `clone` (the project upgrade does it). |
| `SDSL0236`: the value of a composition is ignored | Remove `= value` (the project upgrade does it). |

## Learn more

The manual of Stride 4.5 describes shaders, modules, `import` and `module` members in the Shading language (SDSL) section.
