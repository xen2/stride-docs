# Shaders and modules

<span class="badge text-bg-primary">Intermediate</span>
<span class="badge text-bg-success">Programmer</span>

SDSL code is written in two kinds of units: **shaders** and **modules**. The difference is how many copies of them an effect has.

- A **shader** can have several instances in one effect. It is what you mix into an effect, or plug into a [composition](composition.md) slot. Each instance has its own values.
- A **module** exists **once per effect**. It holds functions, constants and shared state: values, textures, samplers and [streams](automatic-shader-stage-input-output.md). Shaders use it with `import`.

Most of the time you write shaders: material nodes, color transforms, post effects. You write a module when you want to share functions or state between several shaders.

## Shaders

A shader is declared with `shader`. It can inherit other shaders, override their methods, and declare its own members:

```cs
shader ComputeColorTint : ComputeColor
{
    [Color] float4 Tint;

    override float4 Compute()
    {
        return Tint;
    }
};
```

When a material uses `ComputeColorTint` in two slots (for example the diffuse color and the emissive color), the effect has two instances of it, and each instance has its own `Tint`.

A shader mixed at the root of an effect (a post effect, for example) has only one instance: the effect itself.

## Modules

A module is declared with `module`. All its members belong to the module, so there is one of each for the whole effect:

```cs
module MyNoise
{
    static const float Scale = 43758.5453;

    float Hash(float2 p)
    {
        return frac(sin(dot(p, float2(12.9898, 78.233))) * Scale);
    }
};
```

```cs
module MyWeather
{
    float Wetness;
    float3 WindDirection;
};
```

A module can't have instance state, and it can't inherit a shader. It can import other modules.

Many engine classes are modules: `ShaderBase`, `Texturing`, `Transformation`, `Global`, `Camera`, `ImageEffectShader`, `ComputeShaderBase`, `Math` and the stream classes such as `PositionStream4` or `NormalStream`.

## Import a module

A shader uses a module with `import`. Write the imports first in the body:

```cs
shader ComputeColorRain : ComputeColor
{
    import Texturing;
    import MyNoise;
    import MyWeather;

    [Color] float4 DryColor;

    override float4 Compute()
    {
        float drop = step(1 - Wetness, Hash(floor(streams.TexCoord * 64)));
        return lerp(DryColor, DryColor * 0.5, drop);
    }
};
```

The names of an imported module are visible without a prefix (`Wetness`, `Hash`, `streams.TexCoord`), also in shaders that inherit `ComputeColorRain`.

An import doesn't copy the module. If two shaders of the same effect import `MyWeather`, they read the same `Wetness`.

Writing a module in the parent list (`shader ComputeColorRain : ComputeColor, Texturing`) also means import. Code written before Stride 4.5 does this, and it still works. `import` says more clearly what happens.

The order: the compiler walks a shader's parents first, then its imports in the order they are written. A module is placed where the walk meets it first. The order matters only when two classes override the same method (see [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)).

## Static references

You can also use a member of a module by its name, without importing it:

```cs
shader ComputeColorPulse : ComputeColor
{
    [Color] float4 Color;

    override float4 Compute()
    {
        return Color * (0.5 + 0.5 * sin(Global.Time * 2 * Math.PI));
    }
};
```

`Global.Time` and `Math.PI` come from the modules `Global` and `Math`. Like an import, a static reference never makes a copy: it reads the module of the effect.

A static reference to a **shader** works only if your shader inherits it. Otherwise the compiler reports an error, because a shader can have several instances and the compiler can't know which one you mean:

```
MyVignette is a shader, not a module: inherit it to use its members, or make it a module
```

Before Stride 4.5, the compiler mixed the shader in silently in this case.

## Module members of a shader

A shader can also have members that exist once per effect. Mark them `module`:

```cs
shader ComputeColorWater : ComputeColor
{
    import Texturing;

    module Texture2D WaterNormalMap;
    module float WaveSpeed;

    [Color] float4 DeepColor;

    override float4 Compute()
    {
        float2 uv = streams.TexCoord + WaveSpeed * Global.Time;
        return DeepColor * WaterNormalMap.Sample(LinearSampler, uv).z;
    }
};
```

These members form the shader's **companion module**. If a material uses `ComputeColorWater` in two slots, both slots read the same `WaterNormalMap` and `WaveSpeed`, but each slot has its own `DeepColor`.

A `module` member also keeps its parameter key simple: `ComputeColorWaterKeys.WaveSpeed` is the value for the whole effect. An instance member in a composition gets one key per slot, with the slot's path added to its name. That's why material parameters that you set from C# are usually `module` members (see [Custom shaders](../custom-shaders.md)).

Methods can be `module` too. A `module` method is module code: it can use `module` members, modules, and streams.

> [!Note]
> Before Stride 4.5, the `module` modifier was written `stage`. `stage` is now an error. The project upgrade rewrites it (see [Update Stride](../../../install-and-update/update-stride.md)).

## Change engine behavior once per effect

Engine modules have methods that other classes override to change what the effect does, for example `VSMain` and `PSMain` of `ShaderBase`, `Shading` of the sprite and post effect shaders, or `TransformPosition` of `TransformationBase`. An override of a module method is a **module override**: it runs once per effect.

Write it `module override`:

```cs
shader MyGrayscale
{
    import ImageEffectShader;

    module override float4 Shading()
    {
        float4 color = base.Shading();
        float gray = dot(color.rgb, float3(0.299, 0.587, 0.114));
        return float4(gray, gray, gray, color.a);
    }
};
```

In a shader used in a composition (a material node, a light, a material feature), `module override` is required: a plain `override` of a module method gives a warning that says to write `module override`, and an error when an effect uses the shader in a composition. In a shader mixed at the root, it's the same method either way, but `module override` says what it does.

## Rules

Module code (a module, or the `module` members and methods of a shader) exists once per effect. So it can't use the state of one instance:

- Module code can't read an instance member, or call an instance method, of a shader used in a composition.
- An override must be the same kind as the method it overrides: `module override` for a module method, a plain `override` for an instance method. Instance code can't call a module method through `base`.
- A `module compose` slot is one slot for the whole effect. It must be filled at the root of the effect.

The compiler checks these rules when it compiles the shader, and reports warnings ([SDSL0204](../../../../diagnostics/SDSL0204.md) to [SDSL0208](../../../../diagnostics/SDSL0208.md)) that say the fix. When an effect uses the shader in a composition anyway, building the effect fails:

```
Module code `MyBlur.Shading` reads `Radius`, an instance member of `MyBlur`. `MyBlur` is used in a composition (`ShadingColor0`), where each instance has its own `Radius`. Mark `Radius` `module`, or mix `MyBlur` at the root.
```

A shader mixed at the root of an effect may use its own instance members from module code, because it is the only instance. Post effects do this. Such a shader is **root-only** (see below), and the compiler doesn't check these rules for it.

## Root-only shaders

Some shaders are the effect itself: post effects, sprite and particle shaders, compute shaders, effect features that an effect mixes at its root. They are never used in a composition. Mark such a class `[Root]`:

```cs
[Root]
shader MyWindFeature
{
    import TransformationBase;
    import PositionStream4;

    float Strength;

    module override void PreTransformPosition()
    {
        base.PreTransformPosition();
        streams.Position.x += sin(Global.Time) * Strength;
    }
};
```

Without `[Root]`, the compiler warns that the module code reads `Strength`, an instance member ([SDSL0204](../../../../diagnostics/SDSL0204.md)). With `[Root]`, it knows that the shader is the effect's only instance.

A shader that inherits or imports a `[Root]` class directly is root-only; importing a module that imports it doesn't count.

The engine marks `ShaderBase`, `ShadingBase`, `ImageEffectShader`, `ComputeShaderBase`, `SpriteBase` and `ParticleBase` with `[Root]`. So a post effect (which imports `ImageEffectShader`), a sprite shader or a compute shader is root-only without writing `[Root]`.

An effect that uses a root-only shader in a composition fails to build ([SDSL0119](../../../../diagnostics/SDSL0119.md)).

## Pure modules

A module marked `[Pure]` has functions, constants and struct types only:

```cs
[Pure]
module MyNoise
{
    static const float Scale = 43758.5453;

    float Hash(float2 p)
    {
        return frac(sin(dot(p, float2(12.9898, 78.233))) * Scale);
    }
};
```

The compiler checks that a `[Pure]` module:

- declares no variables, constant buffers, textures, samplers, compositions or streams, and doesn't read `streams` ([SDSL0209](../../../../diagnostics/SDSL0209.md)),
- imports, or uses by name, only other `[Pure]` modules ([SDSL0210](../../../../diagnostics/SDSL0210.md)),
- doesn't use operations that exist only in a pixel shader: `ddx`, `ddy`, `fwidth`, `clip`, `discard` ([SDSL0213](../../../../diagnostics/SDSL0213.md)). Texture sampling with an automatic level of detail (`Sample`) isn't checked.

So its functions depend only on their parameters, and you can call them from any shader. `[Pure]` is optional: a module without it isn't checked.

The engine's helper modules are pure: `Math`, `BlendUtils`, `BRDFMicrofacet`, `ColorUtility`, `HSVUtils`, `LuminanceUtils`, `NormalUtil`, `CubemapUtils` and others.

## Shader, module, or module member?

| You want to write | Use |
|---|---|
| a material node, a color transform, a post effect | a `shader` |
| a value that each instance of your shader sets on its own | an instance member (no modifier) |
| a value or texture of your shader that is the same for the whole effect, or a parameter you set from C# | a `module` member |
| a change to an engine method (`VSMain`, `Shading`, `TransformPosition`...) | a `module override` |
| functions, constants or state that several shaders share | a `module`, used with `import` |
| functions and constants only, usable from any shader | a `[Pure] module` |
| an effect feature that an effect mixes only at its root | a `[Root] shader` (not needed if it imports `ImageEffectShader`, `ComputeShaderBase`...) |

## Effects and C#

In an [effect](../effect-language.md) (`.sdfx`) or a C# @'Stride.Shaders.ShaderMixinSource', `mixin X` of a module imports it at the root of the effect. Effects and C# code written before Stride 4.5 keep working.

## See also

* [Quick tour](quick-tour.md)
* [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)
* [Composition](composition.md)
* [Streams and stage input/output](automatic-shader-stage-input-output.md)
* [Custom shaders](../custom-shaders.md)
