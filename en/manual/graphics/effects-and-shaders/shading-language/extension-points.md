# Extension points

An **extension point** is a `module` method that says how its overrides combine. The first definition declares a **kind**, and it can declare **phases**, so that the order of the overrides no longer depends on the order of the mixins.

```cs
module TransformationBase
{
    [Pipeline(World, Offset)]
    void PreTransformPosition() {}
}

shader TransformationWAndVP
{
    import TransformationBase;

    [Phase(World)]
    module override void PreTransformPosition()
    {
        base.PreTransformPosition();
        streams.PositionWS = mul(streams.Position, Transformation.World);
    }
}
```

## Kinds

| Kind | Overrides | `base` |
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

`[After(X)]` means that this override runs after `X`, and it can name several classes. `X` has to exist, but it doesn't have to be in the effect: when it isn't, `[After]` does nothing. It orders overrides inside one phase, so that naming a class of a later phase is an error ([SDSL0138](../../../../diagnostics/SDSL0138.md)), like a circle of `[After]`.

Two overrides of one phase need an order only when it matters: one writes a stream that the other reads or writes, or both have a side effect (a write to a buffer or a texture, an atomic operation, a `discard`). The compiler looks at the override and at the methods it calls, not at what `base` reaches. When nothing orders them:

- in an effect built by the asset compiler, it is error [SDSL0137](../../../../diagnostics/SDSL0137.md);
- in a mix built at run time, the order of the mixins decides.

A class comes after its parents, so inheritance is an order too.

## An override without a phase

An override without `[Phase]` is in an implicit phase, `External`, after every phase the method declares. Overrides in `External` keep the order of the mixins and never conflict. They can call `base` anywhere in their body, also from another method. Shaders written for an earlier version of Stride keep compiling and don't need phases.

Being outermost changes two things for them:

- An override without a phase runs outside every phase, even when it is mixed before a class with a phase.
- If it doesn't call `base`, it drops the overrides with a phase, also the ones mixed after it. Give it a phase to keep them, for example `Surface` for a `Shading` that provides the surface color.

## An override of a method that isn't an extension point

Overriding a `module` method that declares no kind still works: the override takes its place in the chain from the mix order, and the compiler reports [SDSL0246](../../../../diagnostics/SDSL0246.md). Implementing an `abstract` method, and overriding an instance method, are not concerned.

## Dropping an override

An override that doesn't call `base` drops the overrides before it. Replacing what one of your own parents contributed is normal inheritance and isn't reported. Otherwise, when an effect is built by the asset compiler, the compiler reports [SDSL0216](../../../../diagnostics/SDSL0216.md). In a mix built at run time, it writes the same text in the log, as information: replacing engine code is often the point there.

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

`External` lists the overrides without a phase.

## Diagnostics

- [SDSL0135](../../../../diagnostics/SDSL0135.md): a kind or a phase written the wrong way
- [SDSL0137](../../../../diagnostics/SDSL0137.md): two overrides of one phase that nothing orders
- [SDSL0138](../../../../diagnostics/SDSL0138.md): an order that can't be followed
- [SDSL0139](../../../../diagnostics/SDSL0139.md): `[Collect]` is reserved
- [SDSL0140](../../../../diagnostics/SDSL0140.md): the place of `base` doesn't fit the phase
- [SDSL0141](../../../../diagnostics/SDSL0141.md): a phase the method doesn't declare
- [SDSL0142](../../../../diagnostics/SDSL0142.md): a kind on an override
- [SDSL0216](../../../../diagnostics/SDSL0216.md): an override drops others
- [SDSL0246](../../../../diagnostics/SDSL0246.md): an override of a method that is not an extension point

## See also

* [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)
* [Shaders and modules](shaders-and-modules.md)
* [Effects (SDFX)](../effect-language.md)
