# Composition

<span class="badge text-bg-primary">Beginner</span>
<span class="badge text-bg-success">Programmer</span>

In addition to inheritance, SDSL has **composition**. A composition is a slot in a shader, declared with `compose` and the type of shader it accepts. Whoever builds the effect (a material, an [effect file](../effect-language.md), C# code) chooses which shader goes in the slot.

The slot accepts the shader of its type, or any shader that inherits it. Materials are built this way: each material property is a slot of type `ComputeColor`.

## Example code

```cs
shader CompositionBase
{
    float4 Compute()
    {
        return float4(0, 0, 0, 0);
    }
};

shader CompositionShaderA : CompositionBase
{
    float4 myColor;

    override float4 Compute()
    {
        return myColor;
    }
};

shader CompositionShaderB : CompositionBase
{
    float4 myColor;

    override float4 Compute()
    {
        return 0.5 * myColor;
    }
};

shader BaseShader
{
    compose CompositionBase Comp0;
    compose CompositionBase Comp1;

    float4 GetColor()
    {
        return Comp0.Compute() + Comp1.Compute();
    }
};
```

Each slot gets its own **instance** of the shader in it. Its instance members (`myColor` above) are only visible inside that instance, and each instance has its own values. A composition can have compositions too.

## Example code: array of compositions

You can also declare an array of slots. As the number of shaders isn't known in advance, iterate with a `foreach` statement:

```cs
shader BaseShaderArray
{
    compose CompositionBase Comps[];

    float4 GetColor()
    {
        float4 resultColor = float4(0, 0, 0, 0);

        foreach (var comp in Comps)
        {
            resultColor += comp.Compute();
        }

        return resultColor;
    }
};
```

## Module members in compositions

A member or method marked `module` belongs to the shader's companion module: there is only one for the whole effect, however many slots hold the shader. Instance members get one copy per slot.

```cs
shader MyNode
{
    module float SharedValue;
    float InstanceValue;
};

shader TestShader
{
    compose MyNode comp0;
    compose MyNode comp1;
};

// resulting shader (representation)
shader TestShader
{
    float SharedValue;
    float comp0_InstanceValue;
    float comp1_InstanceValue;
};
```

Methods work the same way: a `module` method is produced once, an instance method once per slot.

This is useful for a value that several slots need, but that you want to compute or set only once (for example a texture shared by several material nodes).

Module code runs once per effect, so it can't read the instance members of a shader in a slot. If you need data from outside a slot, put it in a [module](shaders-and-modules.md) or in a `module` member. See [Shaders and modules](shaders-and-modules.md) for the rules.

## Overrides in compositions

A shader in a slot can override methods, and call `base`, like any shader.

An override of a module method (for example `TransformPosition`, which a material feature changes to move vertices) is written `module override`. It runs once per effect, also when the shader is in several slots. A plain `override` of a module method in a composition is an error.

## One slot for the whole effect

A `module compose` slot is part of the companion module: it is one slot for the whole effect, shared by all the instances of the shader. It must be filled at the root of the effect.

```cs
shader MyLayer
{
    module compose ComputeColor SharedMask;
    compose ComputeColor Color;
};
```

## See also

* [Shaders and modules](shaders-and-modules.md)
* [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)
* [Streams and stage input/output](automatic-shader-stage-input-output.md)
* [Generics](templates.md)
* [Effects (SDFX)](../effect-language.md)
