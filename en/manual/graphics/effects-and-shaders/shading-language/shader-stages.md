# Shader stages

The function for each stage has a predefined name, so we recommend you don't change it.

- `HSMain` for hull shader
- `HSConstantMain` for patch constant function
- `DSMain` for domain shader
- `VSMain` for vertex shader (takes no arguments)
- `GSMain` for geometry shader
- `PSMain` for pixel shader (takes no arguments)
- `CSMain` for compute shader (takes no arguments)

These are all void methods.

The entry points exist once per effect. `VSMain` and `PSMain` are declared in the engine module `ShaderBase`, and `CSMain` in `ComputeShaderBase`: import the module, and override them with `module override` (see [Shaders and modules](shaders-and-modules.md)).

The geometry and tessellation shaders need some kind of predefined structure as input and output. However, since Stride shaders are generic, it's impossible to know beforehand what the structure will be. As a result, these shaders use `Input` and `Output` structures that are automatically generated to fit the final shader.

## Vertex shader

A vertex shader has to set the variable with the semantic `SV_Position`. In `ShaderBase`, it's `ShadingPosition`.

For example:

```cs
module override void VSMain()
{
	...
	streams.ShadingPosition = ...;
	...
}
```

## Pixel shader

A pixel shader has to set the variable with the semantic `SV_Target`. In `ShaderBase`, it is `ColorTarget`.

For example:

```cs
module override void PSMain()
{
	...
	streams.ColorTarget = ...;
	...
}
```

### Primitive ID

A pixel shader can read the index of the primitive it draws, with the semantic `SV_PrimitiveID`:

```cs
stream uint PrimitiveId : SV_PrimitiveID;

module override void PSMain()
{
	streams.ColorTarget = ColorOf(streams.PrimitiveId);
}
```

Without a geometry shader, the GPU gives the value. With a geometry shader, the geometry shader must write it (`streams.PrimitiveId = ...;`), or the effect doesn't compile.

On Vulkan, an effect with no tessellation shader needs geometry shader support on the device for this. macOS and iOS (MoltenVK) don't have it.

## Geometry shader

An example of geometry shader:

```cs
[maxvertexcount(1)]
void GSMain(triangle Input input[3], inout PointStream<Output> pointStream)
{
	...
	// fill the streams object
	streams = input[0];
 	...
 
	// always append streams
	pointStream.Append(streams);
	...
}
```

`Input` can be used in the method body. It behaves like the streams object and contains the same members.

`Output` is only used in the declaration of the method. You should append the streams object to your geometry shader output stream.

## Tessellation shader

An example of a tessellation shader:

```cs
[domain("tri")]
[partitioning("fractional_odd")]
[outputtopology("triangle_cw")]
[outputcontrolpoints(3)]
[patchconstantfunc("HSConstantMain")]
[maxtessfactor(48.0)]
void HSMain(InputPatch<Input, 3> input, out Output output, uint uCPID : SV_OutputControlPointID)
{
	...
	output = streams;
}
 
void HSConstantMain(InputPatch<Input, 3> input, const OutputPatch<Input2, 3> output, out Constants constants)
{
	...
	output = streams;
	...
}
 
[domain("tri")]
void DSMain(const OutputPatch<Input, 3> input, out Output output, in Constants constants, float3 f3BarycentricCoords : SV_DomainLocation)
{
	...
	output = streams;
	...
}
```

`Input` and `Input2` both behave like streams.

>[!Note]
>Don't forget to assign `output` to `streams` at the end of your stage.

## Compute shader

An example of a compute shader:

```cs
[numthreads(2, 3, 5)]
void CSMain()
{
	...
}
```

You can also import the engine module `ComputeShaderBase`, which declares `CSMain` and the thread streams, and override its `Compute` method:

```cs
shader MyComputeShader
{
	import ComputeShaderBase;

	RWTexture2D<float4> Output;

	module override void Compute()
	{
		Output[streams.DispatchThreadId.xy] = float4(1, 0, 0, 1);
	}
};
```

## See also

* [Effects (SDFX)](../effect-language.md)
* [Shading language (SDSL)](index.md)
    - [Shaders and modules](shaders-and-modules.md)
    - [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)
    - [Composition](composition.md)
    - [Streams and stage input/output](automatic-shader-stage-input-output.md)
    - [Generics](templates.md)