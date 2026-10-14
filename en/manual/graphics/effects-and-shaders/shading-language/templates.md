# Generics

<span class="badge text-bg-primary">Advanced</span>
<span class="badge text-bg-success">Programmer</span>

A shader can have **generic parameters** (also called template parameters). Their values are fixed when the effect is compiled: each set of values gives its own compiled shader. In Game Studio, a material shows them under **Generics**.

Unlike many template systems, SDSL generic parameters are strongly typed. The available kinds are:

- HLSL value types (`float`, `int`, `bool`, `float2`, `float3`, `float4`)
- semantics (`Semantic`), used to replace the semantic of a stream
- link types (`LinkType`), used to replace the key name of a `[Link]` annotation
- member names (`MemberName`), used where the code expects a name, such as a swizzle (`.rgb`) or a resource group name

A generic shader behaves the same way as any other shader. The value parameters are used like any other variable, but you can't change their value: doing so is a compilation error.

**Code:** Generic parameters

```cs
shader GenericShader<float speed, Semantic mySemantic, LinkType myLink>
{
	[Color]
	[Link("myLink")]
	float4 myColor;

	stream float2 texcoord : mySemantic;

	float4 GetValue()
	{
		return speed * myColor * float4(streams.texcoord, 0, 1);
	}
};

// To instantiate the shader, use:
GenericShader<1.0f, TEXCOORD0, MyColorLink>
```

> [!Note]
> Some engine shaders declare `Texture2D` and `SamplerState` generic parameters, but the shader compiler can't instantiate them at the moment. To choose a texture or a sampler by name, use a `LinkType` parameter, like the engine's `DynamicTexture` and `DynamicSampler` modules (see the example below).

A shader can pass its generic parameters to its parents and to the modules it imports:

```cs
shader ComputeColorTextureSampler<LinkType TTextureName, Semantic TStream, LinkType TSampler, MemberName TRgba> : ComputeColor, DynamicTextureStream<TStream>
{
    import DynamicTexture<TTextureName, PerMaterial>;
    import DynamicSampler<TSampler, PerMaterial>;
    ...
}
```

A module used with different generic values gives one module per set of values that it really uses.

## See also

* [Shading language (SDSL)](index.md)
    - [Shaders and modules](shaders-and-modules.md)
    - [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)
    - [Composition](composition.md)
    - [Streams and stage input/output](automatic-shader-stage-input-output.md)
* [Effects (SDFX)](../effect-language.md)
