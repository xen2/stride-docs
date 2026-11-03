# Streams and stage input/output

<span class="badge text-bg-primary">Advanced</span>
<span class="badge text-bg-success">Programmer</span>

When you write a HLSL shader, you have to precisely define your vertex attributes and carefully pass them across the stage of your final shader.

Here's an example of a simple HLSL shader that uses the color from the vertex.

```cs
struct VS_IN
{
	float4 pos : POSITION;
	float4 col : COLOR;
};

struct PS_IN
{
	float4 pos : SV_POSITION;
	float4 col : COLOR;
};

PS_IN VS( VS_IN input )
{
	PS_IN output = (PS_IN)0;

	output.pos = input.pos;
	output.col = input.col;

	return output;
}

float4 PS( PS_IN input ) : SV_Target
{
	return input.col;
}

technique10 Render
{
	pass P0
	{
		SetGeometryShader( 0 );
		SetVertexShader( CompileShader( vs_4_0, VS() ) );
		SetPixelShader( CompileShader( ps_4_0, PS() ) );
	}
}
```

Imagine you want to add a normal to your model and modify the resulting color accordingly. You have to modify the code that computes the color and adjust the intermediate structures to pass the attribute from the vertex to the pixel shader. You also have to be careful of the semantics you use.

**Code:** Modified HLSL shader

```cs
struct VS_IN
{
	float4 pos : POSITION;
	float4 col : COLOR;
	float3 normal : NORMAL;
};

struct PS_IN
{
	float4 pos : SV_POSITION;
	float4 col : COLOR;
	float3 normal : TEXCOORD0;
};

PS_IN VS( VS_IN input )
{
	PS_IN output = (PS_IN)0;

	output.pos = input.pos;
	output.col = input.col;
	output.normal = input.normal;

	return output;
}

float4 PS( PS_IN input ) : SV_Target
{
	return input.col * max(input.normal.z, 0.0);
}

technique10 Render
{
	pass P0
	{
		SetGeometryShader( 0 );
		SetVertexShader( CompileShader( vs_4_0, VS() ) );
		SetPixelShader( CompileShader( ps_4_0, PS() ) );
	}
}
```

This example is simple. Real projects have many more shaders, so a single change might mean rewriting lots of shaders, structures, and so on.

Schematically, adding a new attribute requires you to update all the stages and intermediate structures from the vertex input to the last stage you want to use the attribute in.

![media/hlsl_add_normal.png](media/hlsl_add_normal.png) 

## SDSL

SDSL has a convenient way to pass parameters across the different stages of your shader. The stream variables are:

- variables
- defined like any shader member, with the stream keyword
- used with the `streams.` prefix (omitting it results in a compilation error)

Streams regroup the concepts of attributes, varyings and outputs in a single concept.

- An attribute is a stream whose value comes from the vertex buffer. It needs a semantic.
- A varying is a stream whose value goes from one stage to the next.
- An output is a stream that the last stage gives to the GPU, such as `ColorTarget`, which has the semantic `SV_Target0`.

[How streams work](#how-streams-work) below gives the exact rules.

Think of streams as global objects that you can access everywhere without specifying as a parameter of your functions.

>[!Note]
>You don't have to create a semantic for these variables; the compiler creates them automatically. However, keep in mind that **the variables sharing the same semantic will be merged in the final shader**. This behavior can be useful when you want to use a stream variable locally without inheriting from the shader where it was declared.

After you declare a stream, you can access it at any stage of your shader. The shader compiler takes care of everything. The variables just have to be visible from the calling code (inherited, or from an imported [module](shaders-and-modules.md)) like any other variable.

The engine declares its common streams in modules: for example `ShadingPosition` and `ColorTarget` in `ShaderBase`, `TexCoord` in `Texturing`, `PositionWS` in `PositionStream4`. Import the module to use them. A stream declared in a module exists once per effect; in a shader used in a [composition](composition.md), declare a stream shared with the rest of the effect as `module stream`.

**Code:** Stream definition and use:

```cs
shader BaseShader
{
	stream float3 myVar;
 
	float3 Compute()
	{
		return streams.myVar;
	}
};
```

### Example of SDSL shader

Let's look at the same HLSL shader as the first example but in SDSL.

**Code:** Same shader in SDSL

```cs
shader MyShader
{
	import ShaderBase;

	stream float4 pos : POSITION;
	stream float4 col : COLOR;

	module override void VSMain()
	{
		streams.ShadingPosition = streams.pos;
	}

	module override void PSMain()
	{
		streams.ColorTarget = streams.col;
	}
};
```

`ShaderBase` is the engine module that declares the entry points `VSMain` and `PSMain` and the streams `ShadingPosition` and `ColorTarget`. They exist once per effect, so the overrides are `module override` (see [Shaders and modules](shaders-and-modules.md)).

Now let's add the normal computation.

**Code:** Modified shader in SDSL

```cs
shader MyShader
{
	import ShaderBase;

	stream float4 pos : POSITION;
	stream float4 col : COLOR;
	stream float3 normal : NORMAL;

	module override void VSMain()
	{
		streams.ShadingPosition = streams.pos;
	}

	module override void PSMain()
	{
		streams.ColorTarget = streams.col * max(streams.normal.z, 0.0);
	}
};
```

In SDSL, adding a new attribute is as simple as adding it to the pool of streams and using it where you want.

![media/sdsl_add_normal.png](media/sdsl_add_normal.png)

## How streams work

`streams` is one value per stage. It holds the fields of every `stream` in the effect that this stage uses. Each stage has its own generated struct, with only those fields.

### Where a value comes from

A stage gets a stream's value from the stage before, unless a whole write to `streams.X` always runs first. These writes don't count, because they might not run or don't write everything:

- a write inside an `if` or a loop,
- a write after an early `return`,
- a write to one component (`streams.X.x = 1;`),
- a write to a `Streams` copy.

Here the pixel shader's write is inside an `if`, so `tint` also comes from the vertex shader. When `UseWhite` is false, the pixel shader reads the vertex shader's value:

```cs
shader CarriedTint
{
	import ShaderBase;

	stream float4 pos : POSITION;
	stream float4 tint;

	bool UseWhite;

	module override void VSMain()
	{
		streams.ShadingPosition = streams.pos;
		streams.tint = float4(0.2, 0.4, 0.6, 1);
	}

	module override void PSMain()
	{
		if (UseWhite)
			streams.tint = float4(1, 1, 1, 1);
		streams.ColorTarget = streams.tint;
	}
};
```

The value goes back as far as needed:

- A vertex shader that doesn't touch a stream still passes it on. So a stream with a semantic that the pixel shader reads first comes from the vertex buffer.
- In the vertex shader, a stream with a vertex input semantic gets its value from the vertex buffer. So `streams.col.w = 1;` keeps the other components of the attribute. `SV_` semantics aren't vertex inputs, except `SV_VertexID` and `SV_InstanceID`.
- Geometry and tessellation shaders get the values in `input`, not in `streams`. They reach `streams` when the code assigns them (`streams = input[i];`). This overwrites the fields that the input has and keeps the others. See [Shader stages](shader-stages.md).

When nothing can give the value, there are two cases:

- **The stream is read with no write before it:** this is the error [SDSL0194](../../../../diagnostics/SDSL0194.md).
- **The stream is read after a write that might not have run, and no earlier stage uses it:** the stream starts at zero, with no error. A semantic only counts in the vertex shader: if a pixel shader writes `streams.col` in an `if` and no earlier stage uses `col`, the other path reads zero, not the vertex buffer.

>[!Note]
>The compiler doesn't compare the branches of an `if`, so a write in both branches also counts as a write that might not run. The result is correct, but the stage gets one input it doesn't need. In a vertex shader, on a stream with a vertex input semantic, that input is a vertex buffer attribute, so the shader asks for an attribute it never uses. To avoid it, write once: `streams.col = useWhite ? white : grey;`.

### One semantic, one field

Streams with the same semantic are one field in the effect, even with different names in different classes. A write to one name can be read through the other. They must have the same type, or the effect doesn't compile.

Streams with the same name, in different classes, are separate fields unless they also have the same semantic and type. Each class uses its own, so this is correct in one effect, and in a class that inherits both and doesn't name them. In a class that inherits or imports both, `streams.name` is the error [SDSL0152](../../../../diagnostics/SDSL0152.md): rename one of them. One stream reached through two parents is still one stream.

A stream that a class declares itself hides an inherited stream of that name in that class, and they are two fields.

A stream is named without its class: `streams.MyShader.myVar` is the error [SDSL0153](../../../../diagnostics/SDSL0153.md).

### Streams in a composition

In a shader used in a [composition](composition.md), the keyword decides how many fields you get:

```cs
shader LayerStreams
{
	stream float3 layerColor;        // one field for each instance of the shader
	module stream float3 sharedColor; // one field for the effect
};
```

A write to `layerColor` in one instance doesn't change it in another. A write to `sharedColor` does.

### `Streams` values

`Streams` is the type of the `streams` value. Use it to keep a copy or to pass the fields to a method:

```cs
shader KeepStreams
{
	import ShaderBase;

	stream float4 pos : POSITION;
	stream float4 tint;

	float4 TintOf(Streams values)
	{
		return values.tint * 0.5;
	}

	module override void VSMain()
	{
		streams.ShadingPosition = streams.pos;
		streams.tint = float4(0.2, 0.4, 0.6, 1);
	}

	module override void PSMain()
	{
		Streams backup = streams;
		streams.tint = float4(1, 1, 1, 1);
		streams.ColorTarget = TintOf(backup);
	}
};
```

- A copy has the fields of the stage it is in, and a method used in two stages gets the fields of each stage.
- A copy is separate: a write to it doesn't change `streams`, until you assign it back (`streams = backup;`).
- A copy taken before a write keeps the old value: `backup.tint` above is the value from the vertex shader.

## Partial structs

A **partial struct** is a struct that other shaders and modules can add fields to, without editing the file that declares it. Like streams, it lets a plugin add data to engine code. Unlike streams, the data is passed explicitly: as parameters, return values and local variables. So the code that uses it runs in any stage, compute shaders included, and you see in a function's signature what it reads.

The class that declares the struct writes `partial struct`. A class that adds fields writes `extend struct`, and must inherit or [import](shaders-and-modules.md) the class that declares it:

```cs
module SurfaceData
{
    partial struct SurfaceInput
    {
        float3 normalWS;
        float2 uv;
    };
};

module WetSurface
{
    import SurfaceData;

    extend struct SurfaceInput
    {
        float wetness;
    };

    float3 Darken(float3 color, SurfaceInput surface)
    {
        return color * (1.0 - 0.5 * surface.wetness);
    }
};
```

A shader sees the fields of the parts it imports: a shader that imports only `SurfaceData` can't read `wetness`; one that imports `WetSurface` can. Values still keep every field when they pass through code that doesn't see it.

In an effect, the struct has the fields of every part in the effect: first the fields of the declaring class, then the other parts sorted by class name, each in the order written. The same set of modules gives the same layout in every effect. A field name can be used only once across all the parts ([SDSL0120](../../../../diagnostics/SDSL0120.md)): if two plugins need the same value, move it into a module both import.

A plain `struct` stays closed: only a struct declared `partial struct` can get more fields ([SDSL0122](../../../../diagnostics/SDSL0122.md)).

### Struct fields

A field of a partial struct can have a struct type: a plain struct, an array of structs, or another partial struct. Each class sees the fields it imports at every level:

```cs
module LightData
{
    struct LightSample
    {
        float3 color;
        float intensity;
    };

    partial struct ShadingInput
    {
        float roughness;
    };
};

module SurfaceLighting
{
    import SurfaceData;
    import LightData;

    extend struct SurfaceInput
    {
        LightSample lights[2];
        ShadingInput shading;
    };

    float3 DirectLight(SurfaceInput surface)
    {
        return surface.lights[0].color * surface.lights[0].intensity * (1.0 - surface.shading.roughness);
    }
};
```

Here `SurfaceInput` (from `SurfaceData` above) gets an array of `LightSample` and a `ShadingInput`, which other modules can extend too. A struct can't contain itself, directly or through other structs and arrays ([SDSL0128](../../../../diagnostics/SDSL0128.md)).

### Limits

The fields of a partial struct depend on the effect, so it can't be where a layout is shared with C#, with another stage or with another effect. In Stride 4.5, a partial struct (or a struct, or array, that contains one) can be used for:

- local variables, parameters (`in`, `out`, `inout`) and return values,
- `static` variables,
- members of a plain struct, fields of another partial struct, and arrays.

It can't be a stream, a cbuffer member, a buffer element (`StructuredBuffer<T>` and the other buffer types), or a `groupshared` variable ([SDSL0125](../../../../diagnostics/SDSL0125.md)). The class that declares it can't have generic parameters ([SDSL0126](../../../../diagnostics/SDSL0126.md)).

## See also

* [Effects (SDFX)](../effect-language.md)
* [Shading language (SDSL)](index.md)
    - [Shaders and modules](shaders-and-modules.md)
    - [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)
    - [Composition](composition.md)
    - [Shader stages](shader-stages.md)
    - [Generics](templates.md)