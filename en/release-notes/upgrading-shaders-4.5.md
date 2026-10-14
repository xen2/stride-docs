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

## What needs a person

- **Warning `Shader upgrade: ...: module code X calls Y.Z, an instance method; this works only if the class of X is mixed at the root, not used in a composition`.** Module code calls a method of a composition type. If the class is mixed at the root of the effect (a post effect, for example), nothing to do. If it's used in a composition slot, the effect fails to build with the "calls ... an instance method" error below: compute the value in instance code, or pass it as a parameter.
- **Warning `Shader upgrade: ...: class X not found in the text, not upgraded`.** The upgrade couldn't find the declaration of `X` in its file (for example, it's built by a macro). Upgrade it by hand.
- **Shaders from other packages.** The upgrade rewrites the `.sdsl` files of your project only. Shaders that come from a NuGet package or another library need a version of that package made for Stride 4.5.
- **Classes added to the parents.** When a class used by name becomes a parent, it may come earlier in the order than the hidden mixin did before. This matters only if it overrides a method that another parent overrides too: check that the result is the one you want (see [Inheritance and overrides](../manual/graphics/effects-and-shaders/shading-language/shader-classes-mixins-and-inheritance.md)).

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

## Error messages and their fix

| Message | Fix |
|---|---|
| ``SDSL0045: `stage` is now `module`: write `module` (the Stride project upgrade does it)`` | Replace `stage` by `module`. |
| `X is a shader, not a module: inherit it to use its members, or make it a module` | You use a member of the shader `X` by name (`X.Member`) without inheriting it. Add `X` to the parents, or declare `X` as `module X` if it has no instance part. |
| ``X is a shader, not a module: inherit it (`: X`) instead of importing it`` | `import` takes a module. Move `X` to the parent list. |
| `module X can't inherit the shader Y: a module only imports other modules` | Make `X` a shader, or make `Y` a module. |
| ``Module code `A.F` reads `M`, an instance member of `A`. `A` is used in a composition (`slot`), where each instance has its own `M`. Mark `M` `module`, or mix `A` at the root.`` | Mark `M` with `module` if one value for the whole effect is right. Otherwise move the code that needs `M` out of module code. |
| ``Module code `A.F` calls `B.G`, an instance method. `A` is used in a composition (`slot`), which has an instance per slot. Mark `B.G` `module`, or mix `A` at the root.`` | Mark `B.G` with `module` (and its overrides). |
| ``Module code `A.F` calls `G`, an instance method of `A`. ...`` | Mark `G` with `module`. |
| ``` `A.F` overrides the module method `B.F` but is not `module`. ... Write `override module`. ``` | Write `module override` (or `override module`, the order of the keywords doesn't matter). |
| ``` `A.F` is `module` but overrides the instance method `B.F`. ... Remove `module`, or mark `B.F` `module`. ``` | Remove `module` from the override, or mark `B.F` with `module`. |
| ``'S' is a `module compose` declared by 'X', so it is one slot for the whole effect and must be supplied at the root.`` | Fill the slot `S` at the root of the effect (in the `.sdfx` file or the C# mixin tree), not in a composition. |
| `SDSL0118: X declares generic parameter 'T' of type Texture2D ...` | Declare the texture as a member, or use a `LinkType` generic parameter. See [SDSL0118](../diagnostics/SDSL0118.md) and [Removed engine shaders](#removed-engine-shaders). |

The messages about module code and overrides appear when an effect is built (in the asset build or at run time), not when the `.sdsl` file is compiled alone: they depend on whether the shader is used in a composition.

## Learn more

The manual of Stride 4.5 describes shaders, modules, `import` and `module` members in the Shading language (SDSL) section.
