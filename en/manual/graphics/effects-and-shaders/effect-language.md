# Effects (SDFX)

An **effect** is the final shader used to draw: it puts together the [shaders and modules](shading-language/shaders-and-modules.md) written in SDSL. You build an effect in C#, or in an effect file (`.sdfx`).

## Create shaders in C&#35;

You can create a shader at runtime with @'Stride.Shaders.ShaderSource' objects. Shaders come in three types:

- @'Stride.Shaders.ShaderClassSource' correspond to a unique shader class
- @'Stride.Shaders.ShaderMixinSource' mix several @'Stride.Shaders.ShaderSource', define compositions
- @'Stride.Shaders.ShaderArraySource' are used for arrays of compositions

To compile a shader, give an @'Stride.Shaders.EffectSource' to the effect compiler. It holds the root of the tree and the preprocessor values of the whole effect:

```cs
var effect = new EffectSource(root)
{
    Macros = { new ShaderMacro("USE_DETAIL", 1) },
};
```

This method produces shaders at runtime: a shader that isn't compiled yet stalls the frame that first needs it (see [Compile shaders](compile-shaders.md)). Additionally, the approach doesn't benefit from the reusability of mixins.

## Stride Effects (SDFX)

Many shaders are variations or combinations of pre-existing shaders. For example, some meshes cast shadows, others receive them, and others need skinning. To reuse code, it's a good idea to select which parts to use through conditions (eg "Skinning required"). This is often solved by "uber shaders": monolithic shaders configured by a set of preprocessor parameters.

Stride offers the same kind of control, keeping extensibility and reusability in mind. The simple code blocks defined by shader classes can be mixed together by a shader mixer. This process can use more complex logic, described in Stride Effect (*.sdfx) files.

### General syntax

An .sdfx file is a small program used to generate shader permutations. It takes a set of parameters (key and value in a collection) and produces a `ShaderMixinSource` ready to be compiled.

An example .sdfx file:

```cs
using Stride.Effects.Data;

namespace StrideEffects
{
    params MyParameters
    {
        bool EnableSpecular = true;
    };
    
    effect BasicEffect
    {
        using params MaterialParameters;
        using params MyParameters;

        mixin ShaderBase;
        mixin TransformationWAndVP;
        mixin NormalVSStream;
        mixin PositionVSStream;
        mixin BRDFDiffuseBase;
        mixin BRDFSpecularBase;
        mixin LightMultiDirectionalShadingPerPixel<2>;
        mixin TransparentShading;
        mixin DiscardTransparent;

        if (MaterialParameters.AlbedoDiffuse != null)
        {
            mixin compose DiffuseColor = ComputeBRDFDiffuseLambert;
            mixin compose albedoDiffuse = MaterialParameters.AlbedoDiffuse;
        }

        if (MaterialParameters.AlbedoSpecular != null)
        {
            mixin compose SpecularColor = ComputeBRDFColorSpecularBlinnPhong;
            mixin compose albedoSpecular = MaterialParameters.AlbedoSpecular;
        }
    };
}
```

### Add a mixin

To add a mixin, use `mixin <mixin_name>`.

The order of the mixins matters when they override the same method: see [Inheritance and overrides](shading-language/shader-classes-mixins-and-inheritance.md).

When the mixin is a [module](shading-language/shaders-and-modules.md) (such as `ShaderBase`), `mixin` imports it at the root of the effect: there is still one of it for the whole effect.

`mixin clone` and `mixin remove` were removed in Stride 4.5: see [SDSL0046](../../../diagnostics/SDSL0046.md) and [SDSL0047](../../../diagnostics/SDSL0047.md).

### Use parameters

The syntax is similar to C#. The following rules are added:

- When you use parameter keys, add them using `params <shader_name>`. If you don't, keys are treated as variables.

- You don't need to tell the program where to check the values behind the keys. Just use the key.

```cs
using params MaterialParameters;
 
if (MaterialParameters.AlbedoDiffuse != null)
{
    mixin MaterialParameters.AlbedoDiffuse;
}
```

The parameters behave like any variable. You can read and write their value, compare their values, and set template parameters. Since some parameters store mixins, they can be used for composition and inheritance, too.

### Custom parameters

You can create your own set of parameters using a structure definition syntax. 

> [!NOTE]
> Even if they're defined in the SDFX file, don't forget the `using` statement when you want to use them.

```cs
params MyParameters
{
    bool EnableSpecular = true; // true is the default value
}
```

### Macros

To set a preprocessor value for every class of the effect, use `mixin macro <name> = <value>`.

```cs
mixin macro USE_DETAIL = 1;
```

A macro belongs to the whole effect. It works at the top level of an effect and in an effect you mix with `mixin`. Inside a `mixin compose` it is error [SDSL0149](../../../diagnostics/SDSL0149.md). In a composition, use a generic for a value, or a separate class for a different shape.

### Compositions

To add a composition, assign the composition variable to your mixin with the syntax below.

```cs
// albedoSpecular is the name of the composition variable in the mixin
mixin compose albedoSpecular = ComputeColorFixed;
 
or
 
mixin compose albedoSpecular = MaterialParameters.AlbedoSpecular;
```

### Sub-mixins

You can also break the code into sub-mixins to reuse elsewhere with the syntax below.

```cs
effect MySubMixin
{
    mixin ComputeColorMultiply;
    mixin compose color1 = ComputeColorStream;
    mixin compose color2 = ComputeColorFixed;
}
 
// to use it
mixin MySubMixin;
mixin compose myComposition = MySubMixin;
```

You can use the `MySubMixin` mixin like any other mixin in the code. Before Stride 4.5, this could also be written `partial effect`, which did nothing more. It is now error [SDSL0311](../../../diagnostics/SDSL0311.md).

### Add to an effect from another project

`extend effect` adds to an effect of another project. A plugin or a game can change an engine effect this way, without copying it:

```cs
namespace Stride.Rendering
{
    extend effect StrideForwardShadingEffect
    {
        mixin MyFogShader;
        extend shader ShadingBase { [MixinOrder(MyFogShader, EffectCompiling)] Shading; }
    }
}
```

Write it in the namespace of the effect, in a project that references the project of the effect. If the build can't find the effect, for example because its name is misspelled, it is error [SDSL0310](../../../diagnostics/SDSL0310.md). A project can extend an effect once, and it can't extend its own effects: that is error [SDSL0312](../../../diagnostics/SDSL0312.md).

The statements of the effect come first, then the ones of each layer. An assembly that references another layer comes after it, also when its code uses no type of that assembly, so a game comes after the plugins it references, and layers that don't know each other are in the order of their assembly names. The [blocks](#blocks) of each layer are one level further out than the ones before it.

A layer also runs when a `mixin child` of the effect is asked for, after the child.

### Blocks

A **block** gives attributes to the methods of a class, for this effect only. It says how the effect uses the [extension points](shading-language/extension-points.md) of the classes it mixes:

```cs
extend shader Owner
{
    [MixinOrder(A, B)] Method;
    [Unbind(Phases = { Shade }, Mixins = { X })] Method;
}

extend shader X
{
    [Phase(P)] Owner.Method;
    [MixinOrder(After = A)] Owner.Method;
    [Phase(MyLibrary.Wind, After = P)] Owner.Method;
}
```

A method of the class is written by its name, `Method`. A method the class contributes to is written with the class of its first definition, `Owner.Method`. Parentheses are only needed when the method has several overloads, with the parameter types as in an `extend`: `Method(float3, int)`, names allowed. A block about `A` also covers every generic version of `A`.

| Attribute | Written on | Meaning |
|---|---|---|
| `[MixinOrder(A, B, C)]` | `Method` | these classes run in this order, in their phase. It is what the error [SDSL0137](../../../diagnostics/SDSL0137.md) asks for |
| `[Unbind(Phases = { P }, Mixins = { X })]` | `Method` | these phases and classes don't run in this effect |
| `[Phase(P)]` | `Owner.Method` | the class runs in this phase |
| `[Phase(MyLibrary.Wind, After = P, Before = Q)]` | `Owner.Method` | the class adds this phase, and runs in it. The name has a prefix, as for an [added phase](shading-language/extension-points.md#add-a-phase) |
| `[MixinOrder(After = A, Before = B)]` | `Owner.Method` | the class runs after `A` and before `B`, in its phase |

A block replaces what the class says with the same attribute, for this effect. So `[Phase(P)]` moves the class to `P`, and `[MixinOrder]` on the method replaces the list the owner wrote. An `extend` placed its call to `base` for the phase its class names: moving it to the other side of the default body is error [SDSL0140](../../../diagnostics/SDSL0140.md). Orders from the effect and from its layers combine as pairs, as [the order inside a phase](shading-language/extension-points.md#how-the-order-is-decided) explains: a layer replaces the opposite order of the effect, and its other orders add up with it.

Use `[Unbind]` when an effect owns its pipeline: its overrides aren't dropped by accident, the effect says they aren't part of it. The shadow caster of the engine has no pixel shading:

```cs
effect ShadowMapCaster
{
    mixin ShadowMapCasterNoPixelShader;
    extend shader ShaderBase { [Unbind(Phases = { Shade, Input })] PSMain; }
}
```

A block about a class that exists and isn't in the effect does nothing, as the effect can be mixed in many ways. A class that exists nowhere is error [SDSL0158](../../../diagnostics/SDSL0158.md), also a class of a package the project doesn't reference. If the class is in the effect, a method it doesn't have, or doesn't contribute to, is error [SDSL0154](../../../diagnostics/SDSL0154.md), and a phase that the method doesn't have is error [SDSL0141](../../../diagnostics/SDSL0141.md) or [SDSL0254](../../../diagnostics/SDSL0254.md). A block can name a phase that an override adds. A block applies to `module` methods.

An attribute that a block can't take, or a block outside an effect, is error [SDSL0256](../../../diagnostics/SDSL0256.md). A name in quotes is kept for a later version: error [SDSL0255](../../../diagnostics/SDSL0255.md).

A block belongs to the whole effect. It works at the top level of an effect, in a condition, and in an effect you mix with `mixin`. In an effect you mix with `mixin compose` it is error [SDSL0151](../../../diagnostics/SDSL0151.md).

The statements `order`, `bind`, `unbind` and `phase` of earlier versions are error [SDSL0251](../../../diagnostics/SDSL0251.md), which gives the block that replaces them. These words, and `extend`, are not reserved: a variable can have these names.

`[Link]` is a different thing: it ties a member to a parameter key, and has nothing to do with blocks.

A C# effect writes the same blocks with `ExtendShader`, `MixinOrder`, `UnbindPhases` and `UnbindMixins` on its `EffectSource`, or on the `ShaderMixinContext` of a builder: see [effects built from C#](shading-language/extension-points.md#effects-built-from-c).

## See also

* [Shading language (SDSL)](shading-language/index.md)
* [Shaders and modules](shading-language/shaders-and-modules.md)
* [Composition](shading-language/composition.md)
