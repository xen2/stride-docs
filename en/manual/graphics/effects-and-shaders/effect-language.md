# Effects (SDFX)

An **effect** is the final shader used to draw: it puts together the [shaders and modules](shading-language/shaders-and-modules.md) written in SDSL. You build an effect in C#, or in an effect file (`.sdfx`).

## Create shaders in C&#35;

You can create a shader at runtime with @'Stride.Shaders.ShaderSource' objects. Shaders come in three types:

- @'Stride.Shaders.ShaderClassSource' correspond to a unique shader class
- @'Stride.Shaders.ShaderMixinSource' mix several @'Stride.Shaders.ShaderSource', set preprocessor values, define compositions
- @'Stride.Shaders.ShaderArraySource' are used for arrays of compositions

This method produces shaders at runtime. However, many platforms don't support HLSL and have no ability to compile shaders at runtime. Additionally, the approach doesn't benefit from the reusability of mixins.

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

>[!Note]
>Even if they're defined in the SDFX file, don't forget the `using` statement when you want to use them.

```cs
params MyParameters
{
	bool EnableSpecular = true; // true is the default value
}
```

### Compositions

To add a composition, assign the composition variable to your mixin with the syntax below.

```cs
// albedoSpecular is the name of the composition variable in the mixin
mixin compose albedoSpecular = ComputeColorTexture;
 
or
 
mixin compose albedoSpecular = MaterialParameters.AlbedoSpecular;
```

### Partial effects

You can also break the code into sub-mixins to reuse elsewhere with the syntax below.

```cs
partial effect MyPartialEffect
{
	mixin ComputeColorMultiply;
	mixin compose color1 = ComputeColorStream;
	mixin compose color2 = ComputeColorFixed;
}
 
// to use it
mixin MyPartialEffect;
mixin compose myComposition = MyPartialEffect;
```

You can use the `MyPartialEffect` mixin like any other mixin in the code.

### Add to an effect from another project

A `partial effect` with the name of an effect of another assembly adds to that effect instead of being an error. A plugin or a game can change an engine effect this way, without copying it:

```cs
namespace Stride.Rendering
{
    partial effect StrideForwardShadingEffect
    {
        mixin MyFogShader;
        order ShadingBase.Shading.Post: MyFogShader, EffectCompiling;
    }
}
```

The statements of the effect come first, then the ones of each layer. An assembly that references another layer comes after it, so a game comes after the plugins it uses, and layers that don't know each other are in the order of their assembly names. For one `order` target, the last layer wins.

### Wiring lines

Wiring lines say how an effect uses the [extension points](shading-language/extension-points.md) of the classes it mixes. They name the first definition of the method, `Owner.Method`, and a phase:

| Line | Meaning |
|---|---|
| `order Owner.Method.Phase: X, Y;` | `X` runs before `Y`. It wins over `[After]`, and it is what the error [SDSL0137](../../diagnostics/SDSL0137.md) asks for |
| `bind Owner.Method.Phase = X;` | puts the override of `X` in this phase, for a class that names no phase or another one |
| `unbind Owner.Method.Phase;` | nothing of this phase runs in this effect |
| `unbind Owner.Method = X;` | the override of `X` doesn't run in this effect |
| `phase Owner.Method.Wind after A before B;` | adds a phase. `after` or `before` alone is enough. A neighbour that doesn't exist makes it warning [SDSL0218](../../diagnostics/SDSL0218.md), and the line does nothing |

Use `unbind` when an effect owns its pipeline: its overrides aren't dropped by accident, the effect says they aren't part of it. The shadow caster of the engine has no pixel shading:

```cs
partial effect ShadowMapCaster
{
    mixin ShadowMapCasterNoPixelShader;
    unbind ShaderBase.PSMain.Shade;
    unbind ShaderBase.PSMain.Input;
}
```

A line about a method or a class that isn't in the effect does nothing, as the effect can be mixed in many ways. A phase that the method doesn't declare is error [SDSL0141](../../diagnostics/SDSL0141.md). The lines apply to `module` methods.

`order`, `bind`, `unbind` and `phase` are not reserved words: a variable can have these names.

### Features, optional slots and variants

Three statements say what an `if` means, so that tools can list the permutations of an effect without running it:

```cs
effect MyEffect
{
    using params MyKeys;

    // A bool parameter
    feature MyKeys.UseFog { mixin Fog; } else { mixin NoFog; }

    // A parameter that C# fills or not. Its value is data, whether it has one is a choice
    optional MyKeys.Extension { mixin compose extension = MyKeys.Extension; }

    // The effect with what the block adds
    variant DepthOnly { unbind ShaderBase.PSMain.Shade; }
}
```

`feature` and `optional` are the structural axes of the effect. `ShaderMixinManager.GetStructuralAxes` returns them, and the generated class carries them as `[EffectAxes]`.

A `variant` is compiled under the name `MyEffect.DepthOnly`. Unlike `mixin child`, which stops at the place it is written, a variant has all the statements of the effect, wherever it is written. It goes at the top of the effect, not inside an `if`.

## See also

* [Shading language (SDSL)](shading-language/index.md)
* [Shaders and modules](shading-language/shaders-and-modules.md)
* [Composition](shading-language/composition.md)
