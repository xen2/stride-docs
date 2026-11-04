# Extension points

An **extension point** is a method, most often a `module` method, that says how its overrides combine. The first definition declares a **kind**, and it can declare **phases**, so that the order of the overrides no longer depends on the order of the mixins.

```cs
module TransformationBase
{
    [Pipeline(World, Offset)]
    void PreTransformPosition() {}
}

shader TransformationWAndVP
{
    import TransformationBase;

    extend TransformationBase.PreTransformPosition in World
    {
        streams.PositionWS = mul(streams.Position, Transformation.World);
    }
}
```

## Extend a method

```cs
extend Owner.Method [(parameters)] [in Phase] [after X, Y] [replaces Z]
{
    ...
}
```

`extend` works in a class that inherits or imports `Owner`, and it is an override: it follows the rules of this page.

- **The parameters** are the ones of the first definition, with their names. You can write them again, and the compiler checks them. Write them when the method has several overloads, or when one is an array, a texture or a buffer.
- **Without `in`**, it goes in the phase of the first definition, which is the innermost one for `[Around]`.
- **A step of a `[Pipeline]`** doesn't call anything: the steps run in phase order. `next` is error [SDSL0144](../../../../diagnostics/SDSL0144.md).
- **In `[Around]` and `[Replace]`**, `next(...)` calls the next one inward. An `extend` that doesn't call it and drops others is error [SDSL0136](../../../../diagnostics/SDSL0136.md), unless it says `replaces`.

`next` only means this inside `extend` and `patch`. `extend`, `patch` and `next` are not reserved words.

To change a method that is not an extension point, write `patch`: a plain `override` of it is an [error](#an-override-of-a-method-that-isnt-an-extension-point). The word shows the cost: the owner of the method didn't plan for this. In Stride 4.5 that word is the only cost: a `patch` compiles like an `override`. Its own shader variant and its hash in the lock file come in a later version.

```cs
patch NormalUpdate.GetTangentMatrix
{
    float3x3 m = next();
    ...
}
```

Two patches of one method need an order, or the result would depend on the order of the mix: error [SDSL0148](../../../../diagnostics/SDSL0148.md). A patch says which one it runs after, and an effect can say it for two patches it can't change:

```cs
patch NormalUpdate.GetTangentMatrix after MyOtherPatch
{
    ...
}
```

```cs
order NormalUpdate.GetTangentMatrix.External: MyOtherPatch, MyPatch;
```

A patch has no phase, because the method declares none. `External` is the implicit phase of what has no phase. The rule is on patches: plain overrides of an instance method keep following the order of the mix.

`extend` of a method that is not an extension point, `patch` of one that is, and `patch` with `in` are error [SDSL0143](../../../../diagnostics/SDSL0143.md).

## The compat form

`override` and `base` keep working, with attributes for what `extend` writes in words:

```cs
[Phase(World)]
[After(TransformationSkinning)]
[Replaces(TransformationWAndVP)]
module override void PreTransformPosition()
{
    base.PreTransformPosition();
    ...
}
```

Some things can only be written this way, and they stay allowed:

- an override without a phase, which runs outside every phase, with `base` anywhere in it;
- `base` from another method of the class;
- a pipeline step that doesn't call `base`, or calls it in the middle of its body.

When a project is upgraded, an override with a phase, or of a `[Replace]` point, becomes an `extend` if it can. The rest of this page uses both forms.

## Kinds

| Kind | Overrides | `next` or `base` |
|---|---|---|
| `[Replace]` | one wins, the last in the chain | calls the method before it |
| `[Pipeline(A, B)]` | each one is a step, and every step runs, in phase order | first in the body: after the phases before it. Last: before them |
| `[Around(A, B)]` | each one wraps the rest, and takes the inner result | calls the next one inward, and the innermost reaches the first definition |

Most engine entry points and hooks are pipelines (`VSMain`, `PSMain`, `PreTransformPosition`). `Shading` is `around`. `ComputeShadingPosition` is `replace`.

A kind goes on the first definition of a `module` method. An override doesn't repeat it.

## Phases

Phases are written in the order the call reaches them: run order for a pipeline, outermost first for `around`. An override picks one with `[Phase]`.

The first definition sits in the phase named `Default` if there is one, else in the first phase. A phase before it holds steps that run before the first definition, so they call `base` last. A phase after it holds steps that run after it, so they call `base` first.

```cs
module ShaderBase
{
    [Pipeline(Input, Default, Deform, Transform, Attributes, Screen)]
    void VSMain() {}
}

module PositionStream2
{
    [Phase(Input)]
    override void VSMain()
    {
        streams.Position = float4(streams.Position2, 0.0f, 1.0f);
        base.VSMain();
    }
}
```

An override can add a phase of its own, for a phase that the first definition doesn't have: `[Phase(Wind, After = World, Before = Offset)]`. A neighbour that doesn't exist is error [SDSL0218](../../../../diagnostics/SDSL0218.md). The neighbour can be a phase that another class adds. The `base` call has to fit the place of the phase, as for any step ([SDSL0140](../../../../diagnostics/SDSL0140.md)), and an `extend` puts it there when the neighbour is a phase of the first definition.

A phase always wins, over inheritance and over the order of the mixins: a class with a phase runs in that phase wherever it is mixed.

## The order inside a phase

Overrides of one phase keep the order of the mixins, unless one of them says otherwise with `[After]`:

```cs
module NormalMeshSkinning
{
    import TransformationSkinning;

    [Phase(World)]
    [After(TransformationSkinning)]
    override void PreTransformPosition()
    {
        base.PreTransformPosition();
        streams.meshNormal = normalize(mul(streams.meshNormal, (float3x3)streams.skinningBlendMatrix));
    }
}
```

An effect can also state the order, with a rule that wins over `[After]`: see [linkage rules](../effect-language.md#linkage-rules). `[After(X)]` means that this override runs after `X`, and it can name several classes. `X` has to exist, but it doesn't have to be in the effect: when it isn't, `[After]` does nothing. It orders overrides inside one phase, so that naming a class of a later phase is an error ([SDSL0138](../../../../diagnostics/SDSL0138.md)), like a circle of `[After]`.

Two overrides of one phase need an order only when it matters: one writes a stream that the other reads or writes, or both have a side effect (a write to a buffer or a texture, an atomic operation, a `discard`). Streams with one semantic count as one stream, because they become one field. A store to the whole of `streams`, such as `streams = input[i]`, writes every stream. The compiler looks at the override and at the methods it calls, not at what `base` reaches. When nothing orders them:

- in an effect built by the asset compiler, it is error [SDSL0137](../../../../diagnostics/SDSL0137.md);
- in a mix built at run time, the order of the mixins decides, and the log says so (see [Effects built from C#](#effects-built-from-c)).

A class comes after its parents, so inheritance is an order too.

## An override without a phase

An override without `[Phase]` is in an implicit phase, `External`, after every phase the method declares. Overrides in `External` keep the order of the mixins and never conflict. They can call `base` anywhere in their body, also from another method. Shaders written for an earlier version of Stride keep compiling and don't need phases.

Being outermost changes two things for them:

- An override without a phase runs outside every phase, even when it is mixed before a class with a phase.
- If it doesn't call `base`, it drops the overrides with a phase, also the ones mixed after it. Give it a phase to keep them, for example `Surface` for a `Shading` that provides the surface color.

## An override of a method that isn't an extension point

An `override` of a `module` method whose first definition declares no kind doesn't compile: error [SDSL0246](../../../../diagnostics/SDSL0246.md). Add the kind to the first definition. Upgrading a project does it for you when the first definition is in the project: `[Pipeline]` if an override calls `base`, else `[Replace]`. If you don't own the first definition, because it is in a library or in the engine, write [`patch`](#extend-a-method) instead of `override`, or ask its author to add the kind. Implementing an `abstract` method, and overriding an instance method, are not concerned.

## Replacing an override

`[Replaces]` on an override says what doesn't run in this chain when this class is in the effect:

```cs
module NormalVSSkinningFromMesh
{
    import NormalFromMesh;

    [Phase(Generate)]
    [Replaces(NormalFromMesh)]
    override void GenerateNormal_VS()
    {
        streams.normalWS = streams.meshNormal;
    }
}

shader MaterialFrontBackBlendShader : ShadingBase
{
    // Whoever provides the surface color, this class provides it instead
    [Phase(Surface)]
    [Replaces(Surface)]
    module override float4 Shading() { ... }
}
```

It takes names of classes and of phases of this method, and a phase wins over a class of the same name. A class that isn't in the effect is ignored, but it has to exist. Only this chain changes: the class still contributes to other methods, and `base` reaches the next override inward. Two overrides can't replace each other ([SDSL0138](../../../../diagnostics/SDSL0138.md)).

> [!NOTE]
> On a class, `[Replaces("Full.Name")]` takes a string and [replaces the whole class](namespaces.md), in every effect. On an override, `[Replaces(Name)]` takes names without quotes, and replaces one contributor of one method.

## Dropping an override

An override that doesn't call `base` drops the overrides before it, unless it states it with `[Replaces]`. Replacing what one of your own parents contributed is normal inheritance and isn't reported. Otherwise, when an effect is built by the asset compiler, it is error [SDSL0216](../../../../diagnostics/SDSL0216.md). In a mix built at run time, the compiler writes the same text in the log, as information: replacing engine code is often the point there.

## Effects built from C\#

A mix built at run time (an `EffectSource` from C#, or a tool such as VL) follows the same rules, so its mixin order is only one source of order among others:

- The order you add mixins never overrides a phase or `[After]`. A class with a phase runs in that phase, even if you add it first.
- It orders the overrides of one phase, and all the overrides without a phase. So a tool that wants a class to be outermost adds it without a phase, last.
- When it decides between two overrides whose order matters, the effect compiles, the log gets one information line, and the explanation prints `order: builder`.

## Instance methods

A kind can also go on an instance method. The effect of the rules is then the [composition](composition.md) node, not the whole effect: phases, `[After]` and `[Replaces]` order the classes mixed into that node. The engine declares `IStreamInitializer.ResetStream` as a pipeline, and `ShadowGroup.ComputeShadow`, `ShadowMapFilterBase.FilterShadow` and the two methods of `TextureProjectionGroup` as `[Replace]`.

An instance method without a kind keeps the order of the mixins, and nothing is reported.

## See what an effect does

The mixer's `Explanation` (and the corpus's `.explain.txt` files) gives the chain of each method and, for an extension point, its running order per phase:

```
module method PSMain() -> void = ShaderBase > ShadingBase > NormalBase > ShadowMapCasterNoPixelShader
  pipeline
  Input: NormalBase
  Default: ShaderBase
  Shade: ShadingBase
  Output: (none)
  External: ShadowMapCasterNoPixelShader
  dropped by ShadowMapCasterNoPixelShader: ShadingBase, NormalBase
```

`External` lists the overrides without a phase. A line `replaced by X: Y` gives what `[Replaces]` took out of the chain, and `not bound by the effect: Y` what an `unbind` rule took out.

A phase with several overrides also says what ordered them, for example `Input: A > B   order: after`:

| Order | Meaning |
|---|---|
| `order rule` | a [linkage rule](../effect-language.md#linkage-rules) of the effect states it |
| `after` | `[After]` states it |
| `builder` | the order of the mixins decided between overrides whose order matters, in a mix built at run time |
| `tie-break` | the order of the mixins, and their order doesn't matter |

## Diagnostics

- [SDSL0135](../../../../diagnostics/SDSL0135.md): a kind or a phase written the wrong way
- [SDSL0136](../../../../diagnostics/SDSL0136.md): an `extend` drops others
- [SDSL0137](../../../../diagnostics/SDSL0137.md): two overrides of one phase that nothing orders
- [SDSL0138](../../../../diagnostics/SDSL0138.md): an order that can't be followed
- [SDSL0139](../../../../diagnostics/SDSL0139.md): `[Collect]` is reserved
- [SDSL0140](../../../../diagnostics/SDSL0140.md): the place of `base` doesn't fit the phase
- [SDSL0141](../../../../diagnostics/SDSL0141.md): a phase the method doesn't declare
- [SDSL0142](../../../../diagnostics/SDSL0142.md): a kind on an override
- [SDSL0143](../../../../diagnostics/SDSL0143.md): `extend` or `patch` of the wrong method
- [SDSL0144](../../../../diagnostics/SDSL0144.md): `next` in a pipeline step
- [SDSL0148](../../../../diagnostics/SDSL0148.md): two patches of one method that nothing orders
- [SDSL0216](../../../../diagnostics/SDSL0216.md): an override drops others
- [SDSL0218](../../../../diagnostics/SDSL0218.md): a phase added next to one that doesn't exist
- [SDSL0246](../../../../diagnostics/SDSL0246.md): an override of a method that is not an extension point

## See also

* [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)
* [Shaders and modules](shaders-and-modules.md)
* [Effects (SDFX)](../effect-language.md)
