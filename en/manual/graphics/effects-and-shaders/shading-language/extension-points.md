# Extension points

An **extension point** is a method, most often a `module` method, that says how its overrides combine. The first definition declares a **kind**, and it can declare **phases**, so that the order of the overrides no longer depends on the order of the mixins.

```cs
module TransformationBase
{
    [Phases(World, Offset)]
    pipeline void PreTransformPosition() {}
}

shader TransformationWAndVP
{
    import TransformationBase;

    [Phase(World)]
    extend TransformationBase.PreTransformPosition()
    {
        streams.PositionWS = mul(streams.Position, Transformation.World);
    }
}
```

## Extend a method

```cs
[Phase(A)]
[MixinOrder(After = { X, Y })]
[Drops(Mixins = { Z })]
extend Owner.Method(parameters)
{
    ...
}
```

`extend` works in a class that inherits or imports `Owner`, and it is an override: it follows the rules of this page. A keyword says how a method takes part (`extend`, `patch`, `pipeline`, `replace`, `around`), and an attribute adds information about it (`[Phases]`, `[Phase]`, `[MixinOrder]`, `[Drops]`, `[Unbind]`).

- **The parameters** are written like in an `override`, also `()`, and the compiler checks them against the first definition. The return type is not written: it comes from the first definition.
- **Without `[Phase]`**, it goes in the phase of the first definition, which is the innermost one for `around`.
- **A step of a `pipeline`** doesn't call anything: the steps run in phase order. `next` is error [SDSL0144](../../../../diagnostics/SDSL0144.md).
- **In `around` and `replace`**, `next(...)` calls the next one inward. An `extend` that doesn't call it and drops others, without saying it with `[Drops]`, is error [SDSL0136](../../../../diagnostics/SDSL0136.md) when an effect is built by the asset compiler. In a mix built at run time, the compiler writes the same text in the log, as information.

`next` only means this inside `extend` and `patch`. `extend`, `patch` and `next` are not reserved words.

To change a method that is not an extension point, write `patch`: a plain `override` of it is an [error](#an-override-of-a-method-that-isnt-an-extension-point). The word shows the cost: the owner of the method didn't plan for this. In Stride 4.5 that word is the only cost: a `patch` compiles like an `override`. Its own shader variant and its hash in the lock file come in a later version.

```cs
patch NormalUpdate.GetTangentMatrix()
{
    float3x3 m = next();
    ...
}
```

Two patches of one method need an order, or the result would depend on the order of the mix: error [SDSL0148](../../../../diagnostics/SDSL0148.md). A patch says which one it runs after, and an effect can say it for two patches it can't change, with a [block](../effect-language.md#blocks):

```cs
[MixinOrder(After = MyOtherPatch)]
patch NormalUpdate.GetTangentMatrix()
{
    ...
}
```

```cs
extend shader NormalUpdate { [MixinOrder(MyOtherPatch, MyPatch)] GetTangentMatrix; }
```

A patch has no phase, because the method declares none. `External` is the implicit phase of what has no phase. The rule is on patches: plain overrides of an instance method keep following the order of the mix.

`extend` of a method that is not an extension point, and `patch` of one that is, are error [SDSL0143](../../../../diagnostics/SDSL0143.md).

## The compat form

`override` and `base` keep working, with the same attributes as `extend`:

```cs
[Phase(World)]
[MixinOrder(After = TransformationSkinning)]
[Drops(Mixins = { TransformationWAndVP })]
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

When a project is upgraded, an override with a phase, or of a `replace` point, becomes an `extend` if it can. The rest of this page uses both forms.

## Kinds

| Kind | Overrides | `next` or `base` |
|---|---|---|
| `replace` | one wins, the last in the chain | calls the method before it |
| `[Phases(A, B)] pipeline` | each one is a step, and every step runs, in phase order | first in the body: after the phases before it. Last: before them |
| `[Phases(A, B)] around` | each one wraps the rest, and takes the inner result | calls the next one inward, and the innermost reaches the first definition |

Most engine entry points and hooks are pipelines (`VSMain`, `PSMain`, `PreTransformPosition`). `Shading` is `around`. `ComputeShadingPosition` is `replace`.

A kind is a keyword, right before the return type. It goes on the first definition of a `module` method, and an override doesn't repeat it. `[Phases]` is optional, and without it the method has one phase, `Default`. The words `pipeline`, `replace` and `around` are not reserved: they stay usable as the names of variables, functions, members and types. `collect` is reserved for a later kind: error [SDSL0139](../../../../diagnostics/SDSL0139.md).

```cs
[Phases(Input, Default, Shade, Output)]
pipeline void PSMain() { }

replace float4 Filter(float4 color) { return color; }

[Phases(Post, Surface)]
around float4 Shading() { return float4(0, 0, 0, 0); }
```

An `abstract` method has no kind: its overrides implement it, they don't take part in a chain.

## Phases

`[Phases]` lists them in the order the call reaches them: run order for a pipeline, outermost first for `around`. An override picks one with `[Phase]`.

The first definition sits in the phase named `Default` if there is one, else in the first phase. A phase before it holds steps that run before the first definition, so they call `base` last. A phase after it holds steps that run after it, so they call `base` first.

```cs
module ShaderBase
{
    [Phases(Input, Default, Deform, Transform, Attributes, Screen)]
    pipeline void VSMain() {}
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

### Add a phase

An override can add a phase that the first definition doesn't have, and run in it:

```cs
[Phase(MyLibrary.Wind, After = World, Before = Offset)]
extend TransformationBase.PreTransformPosition() { ... }
```

- **Its name has a prefix**, such as the name of your library: only the owner of the method declares plain names, in `[Phases]`. So a phase the owner adds later never meets yours. A plain name is error [SDSL0252](../../../../diagnostics/SDSL0252.md). Names that start with `Stride.` are kept for the engine's classes: error [SDSL0253](../../../../diagnostics/SDSL0253.md). The compiler can't see which assembly a class comes from, so it checks the namespace of the class that adds the phase.
- **`After` and `Before` name phases**, one each. `After` alone puts the phase right after its neighbour, `Before` alone right before it. A neighbour can be a phase that another class adds. A neighbour that doesn't exist is error [SDSL0218](../../../../diagnostics/SDSL0218.md).
- **Several classes can add the same phase**, on purpose: it is one phase, and the neighbours of each of them count. If they contradict each other, it is error [SDSL0138](../../../../diagnostics/SDSL0138.md), which names both.
- **Added phases that nothing orders** between them keep the order of the mixins. The explanation shows it.

Another class joins the phase by its full name, `[Phase(MyLibrary.Wind)]`. A name with a dot is checked when the effect is mixed: if no class of the effect adds it, it is error [SDSL0141](../../../../diagnostics/SDSL0141.md) then.

The `base` call has to fit the place of the phase, as for any step ([SDSL0140](../../../../diagnostics/SDSL0140.md)), and the compiler checks every step once the phases are placed. An `extend` puts the call there for you. When its class compiles, it may not know where a phase that another class adds will be: then it compiles twice, with `base` first and with `base` last, and the effect keeps the one that fits.

### Phases of an around method

On an `around` method, phases are nesting levels: the first one is the outermost. An added phase is a level between its neighbours. With `[Phases(Outer, Inner)]`, a phase added `After = Outer` wraps everything in `Inner`, and is wrapped by everything in `Outer`.

## The order inside a phase

Overrides of one phase keep the order of the mixins, unless something says otherwise. An override says it with `[MixinOrder]`:

```cs
module NormalMeshSkinning
{
    import TransformationSkinning;

    [Phase(World)]
    [MixinOrder(After = { TransformationSkinning, TransformationSkinningInstanced })]
    override void PreTransformPosition()
    {
        base.PreTransformPosition();
        streams.meshNormal = normalize(mul(streams.meshNormal, (float3x3)streams.skinningBlendMatrix));
    }
}
```

- `After = X` means that this override runs after `X`, and `Before = Y` before `Y`. Each takes one name, or several in braces, and the attribute can be repeated.
- On the method itself, `[MixinOrder(A, B, C)]` lists classes in their running order. The owner can write it on its first definition, but it is meant for a [block](../effect-language.md#blocks) of an effect.
- A class has to exist, but it doesn't have to be in the effect: when it isn't, its name does nothing. A name in quotes is kept for a later version, which will allow naming a class of a package you don't reference: error [SDSL0255](../../../../diagnostics/SDSL0255.md).
- `[MixinOrder]` orders classes of one phase. The phases already order the others, so naming a class of another phase is error [SDSL0138](../../../../diagnostics/SDSL0138.md).

On an `around` method, "after" means "inside": the override that runs after `X` is entered after it, so `X` wraps it. `ImageScalerShader` runs after `SpriteSuperSampler`, which calls it once per sample.

### How the order is decided

The compiler turns every order into pairs, "`A` runs before `B`", each with the place it is written:

1. Each `[MixinOrder(After = A, Before = B)]` of a class gives one pair per name, and each list `[MixinOrder(A, B, C)]` gives `A` before `B` and `B` before `C`, once the classes that aren't in the effect are left out.
2. A pair has a level: a class is the innermost, then the effect, then each layer of the effect, in the order the layers run. A [block](../effect-language.md#blocks) replaces what its class says with the same attribute: the class's own pairs don't count then.
3. A pair replaces the opposite pair of an inner level. Pairs that don't contradict each other all apply. Two opposite pairs of one level are error [SDSL0138](../../../../diagnostics/SDSL0138.md), which names where both are written.
4. A circle that is left, through several pairs, is error [SDSL0138](../../../../diagnostics/SDSL0138.md) too, with each pair and where it is written.
5. What no pair orders keeps the order of the mixins.

Two overrides of one phase need an order only when it matters:

- One writes a stream that the other reads or writes. Streams with one semantic count as one stream, because they become one field. A store to the whole of `streams`, such as `streams = input[i]`, writes every stream.
- One writes a field of an `inout` or `out` parameter of the point, or of a module `static`, that the other reads or writes. Other fields of the struct don't count. A read or a write of the whole struct counts for every field. A parameter by value is the override's own copy, so it never counts.
- Both have a side effect: a write to a buffer or a texture, an atomic operation, a `discard`.
- The point is `around`: each one wraps the other, so two of one phase always need an order.

The compiler looks at the override and at the methods it calls, not at what `base` reaches. When nothing orders them:

- in an effect built by the asset compiler, it is error [SDSL0137](../../../../diagnostics/SDSL0137.md);
- in a mix built at run time, the order of the mixins decides, and the log says so (see [Effects built from C#](#effects-built-from-c)).

A class comes after its parents, so inheritance is an order too.

## An override without a phase

An override without `[Phase]` is in an implicit phase, `External`, after every phase the method declares. Overrides in `External` keep the order of the mixins and never conflict. They can call `base` anywhere in their body, also from another method. Shaders written for an earlier version of Stride keep compiling and don't need phases.

Being outermost changes two things for them:

- An override without a phase runs outside every phase, even when it is mixed before a class with a phase.
- If it doesn't call `base`, it drops the overrides with a phase, also the ones mixed after it. Give it a phase to keep them, for example `Surface` for a `Shading` that provides the surface color.

## An override of a method that isn't an extension point

An `override` of a `module` method whose first definition declares no kind doesn't compile: error [SDSL0246](../../../../diagnostics/SDSL0246.md). Add the kind to the first definition. Upgrading a project does it for you when the first definition is in the project: `pipeline` if an override calls `base`, else `replace`. If you don't own the first definition, because it is in a library or in the engine, write [`patch`](#extend-a-method) instead of `override`, or ask its author to add the kind. Implementing an `abstract` method, and overriding an instance method, are not concerned.

## Drop other overrides

`[Drops]` on an override says what doesn't run in this chain when this class is in the effect:

```cs
module NormalVSSkinningFromMesh
{
    import NormalFromMesh;

    [Phase(Generate)]
    [Drops(Mixins = { NormalFromMesh, NormalFromMeshInstanced })]
    override void GenerateNormal_VS()
    {
        streams.normalWS = streams.meshNormal;
    }
}

shader MaterialFrontBackBlendShader : ShadingBase
{
    // Whoever provides the surface color, this class provides it instead
    [Phase(Surface)]
    [Drops(Phases = { Surface })]
    module override float4 Shading() { ... }
}
```

`Mixins` takes classes and `Phases` takes phases of this method, so a name always says what it is. A name of the wrong kind is error [SDSL0254](../../../../diagnostics/SDSL0254.md). A class that isn't in the effect is ignored, but it has to exist. Only this chain changes: the class still contributes to other methods, and `base` reaches the next override inward. Two overrides can't drop each other ([SDSL0138](../../../../diagnostics/SDSL0138.md)).

To take overrides out of one effect, the effect writes [`[Unbind]`](../effect-language.md#blocks) in a block.

> [!NOTE]
> On a class, `[Replaces(Full.Name)]` [replaces the whole class](namespaces.md), in every effect. `[Drops]` on an override drops contributors of one method.

## Dropping an override

An override that doesn't call `base` drops the overrides before it, unless it states it with `[Drops]`. Replacing what one of your own parents contributed is normal inheritance and isn't reported. Otherwise, when an effect is built by the asset compiler, it is error [SDSL0216](../../../../diagnostics/SDSL0216.md). In a mix built at run time, the compiler writes the same text in the log, as information: replacing engine code is often the point there.

## Effects built from C\#

A mix built at run time (an `EffectSource` from C#, or a tool such as VL) follows the same rules, so its mixin order is only one source of order among others:

- The order you add mixins never overrides a phase or `[MixinOrder]`. A class with a phase runs in that phase, even if you add it first.
- It orders the overrides of one phase, and all the overrides without a phase. So a tool that wants a class to be outermost adds it without a phase, last.
- When it decides between two overrides whose order matters, the effect compiles, the log gets one information line, and the explanation prints `order: builder`.

It can also write the [blocks](../effect-language.md#blocks) of an effect, with calls shaped like their attributes. Each key of an attribute is its own call: `MixinOrderAfter` and `MixinOrderBefore`, `UnbindPhases` and `UnbindMixins`.

```cs
var effect = new EffectSource(mixin);
effect.UnbindPhases("Stride.Rendering.ShaderBase", "PSMain", "Shade", "Input");
effect.MixinOrder("Stride.Rendering.ShadingBase", "Shading", "MyFogShader", "MyGlowShader");
effect.ExtendShader("MyFogShader").Method("Stride.Rendering.ShadingBase.Shading").Phase("Post").MixinOrderAfter("MyTintShader");
```

## Instance methods

A kind can also go on an instance method. The effect of the rules is then the [composition](composition.md) node, not the whole effect: phases, `[MixinOrder]` and `[Drops]` order the classes mixed into that node. The engine declares `IStreamInitializer.ResetStream` as a pipeline, and `ShadowGroup.ComputeShadow` and the two methods of `TextureProjectionGroup` as `replace`.

An instance method without a kind keeps the order of the mixins, and nothing is reported.

The blocks of an effect are about its root, so they don't reach a composition. To settle two overrides in a node, use `[MixinOrder]`, or give one of them a phase of its own: `[Phase(MyLibrary.Later, After = Detail)]`.

## See what an effect does

The mixer's `Explanation` (and the corpus's `.explain.txt` files) gives the chain of each method and, for an extension point, its overrides per phase, in the order they run:

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

The first line is the chain, from the first definition outwards. In a phase before the default body, and in every phase of an `around` method, the outermost override runs first, so the phase line reads the chain backwards. `External` lists the overrides without a phase, in the order of the chain. A line `replaced by X: Y` gives what `[Drops]` took out of the chain, and `not bound by the effect: Y` what `[Unbind]` took out.

A phase with several overrides also says what ordered them, for example `Input: MeshVelocity > ScreenPositionBase > NormalBase   order: mixin order`:

| Order | Meaning |
|---|---|
| `mixin order` | `[MixinOrder]` of a class states it |
| `block` | a [block](../effect-language.md#blocks) of the effect states it, or a block written in C# |
| `builder` | the order of the mixins decided between overrides whose order matters, in a mix built at run time |
| `tie-break` | the order of the mixins, and the compiler found nothing above that depends on it |

`tie-break` only knows the list above. One override that writes a buffer or a texture while the other reads it, or two steps of a `pipeline` that change what it returns, are not on the list. If the order matters there, state it.

## Diagnostics

- [SDSL0135](../../../../diagnostics/SDSL0135.md): an attribute of an extension point written the wrong way
- [SDSL0136](../../../../diagnostics/SDSL0136.md): an `extend` drops others
- [SDSL0137](../../../../diagnostics/SDSL0137.md): two overrides of one phase that nothing orders
- [SDSL0138](../../../../diagnostics/SDSL0138.md): an order that can't be followed
- [SDSL0139](../../../../diagnostics/SDSL0139.md): `collect` is reserved
- [SDSL0140](../../../../diagnostics/SDSL0140.md): the place of `base` doesn't fit the phase
- [SDSL0141](../../../../diagnostics/SDSL0141.md): a phase the method doesn't declare
- [SDSL0142](../../../../diagnostics/SDSL0142.md): a kind on an override
- [SDSL0143](../../../../diagnostics/SDSL0143.md): `extend` or `patch` of the wrong method, or written the wrong way
- [SDSL0144](../../../../diagnostics/SDSL0144.md): `next` in a pipeline step
- [SDSL0148](../../../../diagnostics/SDSL0148.md): two patches of one method that nothing orders
- [SDSL0216](../../../../diagnostics/SDSL0216.md): an override drops others
- [SDSL0218](../../../../diagnostics/SDSL0218.md): a phase added next to one that doesn't exist
- [SDSL0246](../../../../diagnostics/SDSL0246.md): an override of a method that is not an extension point
- [SDSL0249](../../../../diagnostics/SDSL0249.md): `[Pipeline]`, `[Replace]` or `[Around]`, which are keywords
- [SDSL0250](../../../../diagnostics/SDSL0250.md): a kind written the wrong way
- [SDSL0251](../../../../diagnostics/SDSL0251.md): `[After]`, `[Replaces]` on a method, or a linkage rule of an earlier version
- [SDSL0252](../../../../diagnostics/SDSL0252.md): an added phase without a prefix
- [SDSL0253](../../../../diagnostics/SDSL0253.md): an added phase that starts with `Stride.`
- [SDSL0254](../../../../diagnostics/SDSL0254.md): `[Drops]` or `[Unbind]` without `Mixins` and `Phases`, or with a name of the wrong kind
- [SDSL0255](../../../../diagnostics/SDSL0255.md): a name in quotes

## See also

* [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)
* [Shaders and modules](shaders-and-modules.md)
* [Effects (SDFX)](../effect-language.md)
