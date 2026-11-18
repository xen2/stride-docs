# Quick tour

<span class="badge text-bg-primary">Beginner</span>
<span class="badge text-bg-success">Programmer</span>

This page shows the main parts of the Stride shading language (SDSL) in a few small steps. Each step links to the page with the details.

SDSL is HLSL with a few additions: you write shaders that build on engine shaders, and Stride puts them together into the final shader for you.

To create the shader file and use it in Game Studio, follow [Custom shaders](../custom-shaders.md).

## 1. Write a shader

This shader is a material node: it computes a color for a material property, such as the diffuse color.

```cs
shader ComputeColorPulse : ComputeColor
{
    override float4 Compute()
    {
        return float4(1, 0, 0, 1);
    }
};
```

- `shader ComputeColorPulse` declares the shader. The name must be the same as the file name (`ComputeColorPulse.sdsl`).
- `: ComputeColor` inherits the engine shader `ComputeColor`. A material property accepts any shader that inherits `ComputeColor`.
- `override float4 Compute()` replaces the method `ComputeColor.Compute()`. Every override needs the `override` keyword.

See [Inheritance and overrides](shader-classes-mixins-and-inheritance.md).

## 2. Add a parameter

A member of the shader is a parameter: you can set it in Game Studio or from C#.

```cs
shader ComputeColorPulse : ComputeColor
{
    [Color] float4 Color;
    float Speed = 1.0;

    override float4 Compute()
    {
        return Color * (0.5 + 0.5 * sin(Global.Time * Speed));
    }
};
```

`[Color]` tells Game Studio and the generated C# key that `Color` is a color.

`Global.Time` is the time in seconds. `Global` is an engine **module**: it exists once per effect, and you can use its members by name.

## 3. Use engine modules

`Texturing` is the engine module with the texture coordinates, the default textures and the samplers. Import it to use its members:

```cs
shader ComputeColorPulse : ComputeColor
{
    import Texturing;

    [Color] float4 Color;
    float Speed = 1.0;

    override float4 Compute()
    {
        float wave = sin((streams.TexCoord.x + Global.Time) * Speed);
        return Color * (0.5 + 0.5 * wave);
    }
};
```

`streams.TexCoord` is a **stream**: a value that the engine passes from the vertex shader to the pixel shader for you. See [Streams and stage input/output](automatic-shader-stage-input-output.md).

## 4. Call the inherited method

`base.Method()` calls the method that your override replaces. This shader wraps another color and makes it brighter:

```cs
shader ComputeColorBrighter : ComputeColorPulse
{
    float Boost = 2.0;

    override float4 Compute()
    {
        return base.Compute() * Boost;
    }
};
```

## 5. Plug in another shader

A `compose` member is a slot: whoever builds the effect (for example the material in Game Studio) chooses which shader goes in it.

```cs
shader ComputeColorMix : ComputeColor
{
    compose ComputeColor A;
    compose ComputeColor B;
    float Amount = 0.5;

    override float4 Compute()
    {
        return lerp(A.Compute(), B.Compute(), Amount);
    }
};
```

Each slot has its own instance of the shader it holds, with its own parameters. See [Composition](composition.md).

## 6. Share code with a module

Functions and values that several shaders use go into a **module**. A module exists once per effect:

```cs
module MyNoise
{
    float Hash(float2 p)
    {
        return frac(sin(dot(p, float2(12.9898, 78.233))) * 43758.5453);
    }
};
```

```cs
shader ComputeColorNoise : ComputeColor
{
    import Texturing;
    import MyNoise;

    override float4 Compute()
    {
        float n = Hash(floor(streams.TexCoord * 64));
        return float4(n, n, n, 1);
    }
};
```

You can also call `MyNoise.Hash(...)` without the import. See [Shaders and modules](shaders-and-modules.md).

## 7. Change what the whole effect does

Some engine methods run once per effect: `VSMain` and `PSMain` of `ShaderBase`, `Shading` of the post effect shaders, `TransformPosition` of the transformation shaders. To change them, write a `module override`:

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

`ImageEffectShader` draws a texture on the whole screen; this post effect turns its color to gray.

A member of a shader that must be the same for the whole effect is a `module` member, for example `module float Strength;`. See [Shaders and modules](shaders-and-modules.md).

## Next steps

* [Custom shaders](../custom-shaders.md): create a shader file and use it in a material.
* [Shaders and modules](shaders-and-modules.md): instances, modules, `import` and `module` members.
* [Composition](composition.md): slots and arrays of slots.
* [Effects (SDFX)](../effect-language.md): how shaders are put together into effects.
