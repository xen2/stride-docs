# Inheritance and overrides

<span class="badge text-bg-primary">Intermediate</span>
<span class="badge text-bg-success">Programmer</span>

A shader can inherit other shaders and override their methods. This is how you change engine shaders without copying them.

## Inherit and override

A shader lists its parents after `:`. It sees their members and methods, and it can override their methods:

```cs
shader BaseShader
{
    float Compute()
    {
        return 1.0;
    }
};

shader ShaderA : BaseShader
{
    override float Compute()
    {
        return 2.0;
    }
};

shader ShaderB : BaseShader
{
    override float Compute()
    {
        float previous = base.Compute();
        return 5.0 + previous;
    }
};
```

- An override needs the `override` keyword. A method with the same name and signature as an inherited one, but without `override`, is an error.
- `base.Compute()` calls the method that this override replaces.

Parents that are modules are imported, not inherited: see [Shaders and modules](shaders-and-modules.md). Overrides of module methods are written `module override`.

## Abstract methods

An `abstract` method has no body. A shader that inherits it can leave it abstract, but the final effect must contain a shader that overrides it.

```cs
shader ColorSource
{
    abstract float4 GetColor();
};

shader RedColorSource : ColorSource
{
    override float4 GetColor()
    {
        return float4(1, 0, 0, 1);
    }
};
```

## Several parents: the order matters

A shader can have several parents. SDSL puts all the classes of a shader in one list, in a fixed order: each parent with its own parents first, in the order they are written. Each class is in the list once, at its first place. These classes are often called **mixins**.

For each method, the last override in the list wins, and `base` calls the one just before it.

```cs
shader MixAB : ShaderA, ShaderB
{
};

shader MixBA : ShaderB, ShaderA
{
};

// Resulting code (representation)

// Order: BaseShader, ShaderA, ShaderB
shader MixAB
{
    float Compute()
    {
        // code from BaseShader
        float v0 = 1.0;

        // code from ShaderA
        float v1 = 2.0;

        // code from ShaderB
        float previous = v1;
        float v2 = 5.0 + previous;

        return v2; // = 7.0
    }
};

// Order: BaseShader, ShaderB, ShaderA
shader MixBA
{
    float Compute()
    {
        // code from BaseShader
        float v0 = 1.0;

        // code from ShaderB
        float previous = v0;
        float v1 = 5.0 + previous;

        // code from ShaderA
        float v2 = 2.0;

        return v2; // = 2.0
    }
};
```

`ShaderA` doesn't call `base`, so in `MixBA` it hides the result of `ShaderB`.

The same order applies when an [effect](../effect-language.md) mixes several shaders: `mixin ShaderA; mixin ShaderB;` gives the order of `MixAB`.

## Keywords

SDSL uses the keywords of HLSL, and adds these:

- `shader`: declares a shader. See [Shaders and modules](shaders-and-modules.md).
- `module`: declares a module, or marks a member or method of a shader as part of its companion module (once per effect). See [Shaders and modules](shaders-and-modules.md).
- `import`: uses a module in a shader or module. See [Shaders and modules](shaders-and-modules.md).
- `override`: a method that replaces an inherited method. Required.
- `abstract`: a method declared without a body.
- `base`: calls the method that an override replaces.
- `compose`: a slot for another shader. See [Composition](composition.md).
- `stream`: a member that is passed between shader stages. See [Streams and stage input/output](automatic-shader-stage-input-output.md).
- `streams`: the object that holds the streams. See [Streams and stage input/output](automatic-shader-stage-input-output.md).
- `Input`, `Output`, `Input2`, `Constants`: for geometry and tessellation shaders. See [Shader stages](shader-stages.md).

> [!Note]
> `stage` was the name of the `module` modifier before Stride 4.5. It is now an error.

## Annotations

Like in HLSL, members can have annotations. The most useful ones are:

- `[Color]` on a `float3` or `float4` member: the parameter key has the type `Color3` or `Color4` instead of `Vector3` or `Vector4`, and Game Studio shows a color picker.
- `[Link("...")]` sets the name of the parameter key used to set this value.
- `[Map("...")]` uses an existing parameter key for this value. No new key is created.

```cs
shader MyAnnotatedShader
{
    [Color] float4 MyColor;

    [Link("ProjectKeys.MyTextureKey")]
    Texture2D MyTexture;

    [Map("Texturing.Texture0")]
    Texture2D DefaultTexture;
};
```

## See also

* [Shaders and modules](shaders-and-modules.md)
* [Composition](composition.md)
* [Streams and stage input/output](automatic-shader-stage-input-output.md)
* [Shader stages](shader-stages.md)
* [Effects (SDFX)](../effect-language.md)
