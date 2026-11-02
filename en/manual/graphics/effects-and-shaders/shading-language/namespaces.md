# Namespaces

<span class="badge text-bg-primary">Intermediate</span>
<span class="badge text-bg-success">Programmer</span>

Shaders and effects live in namespaces, as C# types do. Two packages can each have a `Noise` shader if they use different namespaces.

## Declare a namespace

Write the namespace on its own line for the whole file, or as a block:

```cs
namespace MyGame.Rendering;

shader ComputeColorTint : ComputeColor
{
    [Color] float4 Tint;

    override float4 Compute()
    {
        return Tint;
    }
};
```

```cs
namespace MyGame.Rendering
{
    shader ComputeColorTint : ComputeColor
    {
        [Color] float4 Tint;

        override float4 Compute()
        {
            return Tint;
        }
    };
}
```

The **full name** of this shader is `MyGame.Rendering.ComputeColorTint`. A shader with no namespace is in the global namespace: its full name is its short name, and every shader can see it.

An effect (`.sdfx`) follows the same rules.

> [!Note]
> We recommend that a namespace starts with the name of your package. The project upgrade doesn't add a namespace to an existing shader, because that would change the names your C# code and your assets use.

## Find a class by its name

In shader code you can use a short name: after `:`, in `compose`, `import` and `mixin`, in a static reference such as `Global.Time`, and in a generic argument. The compiler looks for the name in this order, and stops at the first level that has it:

1. The namespace of the class.
2. Its parent namespaces, closest first. A `using` written inside a `namespace X { }` block counts at the level of that block.
3. The global namespace.
4. The `using` lines of the file and the implicit usings, all at one level.

```cs
using MyPlugin.Noise;

namespace MyGame.Rendering;

shader ComputeColorRain : ComputeColor  // ComputeColor: an implicit using
{
    import Texturing;
    import MyNoise;                     // MyPlugin.Noise.MyNoise: the using
    import Weather;                     // MyGame.Rendering.Weather: its own namespace comes first

    override float4 Compute()
    {
        return Weather.Wetness * Hash(streams.TexCoord);
    }
};
```

If the name isn't in any of these, it's an error, and the message says which `using` to add when a class of that name exists elsewhere. There is no search across all namespaces.

If two classes match at the last level, it's an error too ([SDSL0300](../../../../diagnostics/SDSL0300.md)). Write the full name of the one you want. A `using` doesn't win over an implicit using, so a plugin update that adds `MyPlugin.Foo` can't silently take over a name that your shader already uses.

You can always write the full name: `shader MyTint : Stride.Rendering.ComputeColor`.

### Implicit usings

These namespaces of the engine are in scope in every file, so you don't write `using` for them:

- `Stride.Rendering`
- `Stride.Rendering.Materials`
- `Stride.Rendering.Lights`
- `Stride.Rendering.Images`
- `Stride.Rendering.Shadows`

Other engine shaders need a `using`, for example `using Stride.Rendering.Voxels;`.

## Name a shader or an effect from C#

C# names a shader or an effect by its full name. The generated `XxxKeys` class has a constant for it, even when the shader has no parameter:

```cs
var color = new ShaderClassSource(ComputeColorTintKeys.ShaderName);

var effect = EffectSystem.LoadEffect(MyEffectKeys.EffectName).WaitForResult();
```

This way a typo or a renamed shader is a build error. `XxxKeys` is generated in the namespace of the shader, so a C# `using` brings it in.

A string works too, but it must be the full name. A short name for a class in a namespace is an error when the shader or the effect loads, and the message says what you can mean ([SDSL0307](../../../../diagnostics/SDSL0307.md)):

```cs
new ShaderClassSource("MyGame.Rendering.ComputeColorTint"); // Works
new ShaderClassSource("ComputeColorTint");                  // SDSL0307
```

The same applies wherever a shader or an effect is named by text: render stage selectors, the graphics compositor, material assets and effect logs. Names of child effects are written after the full name: `Stride.Rendering.StrideForwardShadingEffect.ShadowMapCaster`.

The project upgrade rewrites the names in your assets and the literal strings in your C# code. Strings that your code builds at run time need a person (see [Upgrading shaders from 4.4](../../../../release-notes/upgrading-shaders-4.5.md)).

## Keys

The key of a shader member is named after the full name of its class: the member `Tint` above has the key `MyGame.Rendering.ComputeColorTint.Tint`. So two shaders with one short name in two namespaces don't share a key.

A shader in the global namespace keeps the short form, `ComputeColorTint.Tint`.

Keys declared in C# (`[EffectKeys]`) or in the `params` of an effect keep their names, such as `Material.DiffuseMap`. They reach a shader through `[Link]`, which is text. `[Link]` is also how two classes share a key on purpose.

If your C# code uses `ComputeColorTintKeys.Tint`, nothing changes. If it builds the name of a key as text, it must use the full name.

## Shaders and files

A shader is found by its full name, whichever file declares it. A file can have any name, and it can declare several shaders, in one or several namespaces:

```cs
namespace MyGame.Rendering;

shader ComputeColorTint : ComputeColor
{
    [Color] float4 Tint;

    override float4 Compute() { return Tint; }
};

shader ComputeColorFade : ComputeColorTint
{
    override float4 Compute() { return base.Compute() * 0.5f; }
};
```

Asking for a shader compiles that shader only, so a mistake in `ComputeColorFade` doesn't stop `ComputeColorTint`. A shader can inherit a shader that comes later in its file.

A shader that a file only declares inside an `#if` needs the macro to be defined when something asks for it. Otherwise the error is [SDSL0308](../../../../diagnostics/SDSL0308.md).

## Duplicate names

A full name can only be declared once in a game. Two shaders with one full name are a build error that names both ([SDSL0301](../../../../diagnostics/SDSL0301.md)), even if one of them is in a package you depend on. Two effects with one full name are an error too ([SDSL0302](../../../../diagnostics/SDSL0302.md)).

A game shader that has the short name of an engine shader, in another namespace or in the global namespace, is simply a different shader. It doesn't replace the engine one. To replace a shader, use `[Replaces]`.

## Replace a shader

`[Replaces]` on a class takes the full name of a shader and changes it for the whole game, also where engine shaders use it. The new shader inherits the original and overrides what it changes: To replace one contributor of one method, put it on an override: see [Extension points](extension-points.md).

```cs
namespace MyGame.Rendering;

[Replaces("Stride.Rendering.ComputeColorMultiply")]
shader MyMultiply : Stride.Rendering.ComputeColorMultiply
{
    override float4 Compute()
    {
        return base.Compute() * 2;
    }
};
```

In every effect that uses `ComputeColorMultiply`, `MyMultiply` joins right after it. So its overrides are called, also by shaders that derive from the original. `base` still means the original.

Rules:

- The new shader inherits the original directly, and it's the same kind: a shader for a shader, a module for a module ([SDSL0305](../../../../diagnostics/SDSL0305.md)). It can only be `[Root]` if the original is.
- The original must exist, and be in your package or in a package it depends on ([SDSL0303](../../../../diagnostics/SDSL0303.md)).
- A class has one replacer at most ([SDSL0304](../../../../diagnostics/SDSL0304.md)). To overrule a plugin that already replaces a shader, replace the plugin's shader.
- A generic shader is replaced by one with the same generic parameters, passed on unchanged ([SDSL0306](../../../../diagnostics/SDSL0306.md)): `shader MyBlur<int Radius> : Blur<Radius>`.

Limits:

- The original's variables, streams and parents stay. What nothing uses is removed when the effect compiles.
- The original's keys stay, so C# code that sets a key of `ComputeColorMultiplyKeys` keeps working. New members get keys named after the new shader.
- Effects can't be replaced. Name your own effect where the game selects one, for example in the graphics compositor.
- Effects that the engine embeds as bytecode (`SpriteBatch`, `SpriteEffect`, `UIEffect`, `PhysicsDebugEffect`) keep the engine's code.

Adding or removing a replacer changes the id of every effect, so all effects compile again once.

## See also

* [Shaders and modules](shaders-and-modules.md)
* [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)
* [Effects (SDFX)](../effect-language.md)
* [Upgrading shaders from 4.4](../../../../release-notes/upgrading-shaders-4.5.md)
