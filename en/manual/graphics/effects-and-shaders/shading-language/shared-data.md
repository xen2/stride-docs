# Data shared with C#

A record that C# writes and a shader reads is declared once, in the shader, with `[CSharp]`. When the project builds, the shader generator writes its C# twin: the same enum, or a struct with the same fields at the same offsets. C# fills a buffer with the twin, and nothing is written twice by hand.

```cs
namespace MyGame.Rendering;

[Pure]
module MaterialRecords
{
    [CSharp] enum class SlotChannel : uint { All, R, G, B, A };

    [CSharp] struct SlotRecord
    {
        float4 constant;
        int textureIndex;
        SlotChannel channel;
        float bias;
    };
};
```

The generator writes, in the namespace of the shader file:

```cs
public enum SlotChannel : uint { All = 0, R = 1, G = 2, B = 3, A = 4 }

[StructLayout(LayoutKind.Sequential)]
public partial struct SlotRecord
{
    public Vector4 constant;
    public int textureIndex;
    public SlotChannel channel;
    public float bias;
}
```

C# then fills a structured buffer with it:

```cs
var slots = new[] { new SlotRecord { constant = Vector4.One, channel = SlotChannel.R } };
using var buffer = Buffer.Structured.New(device, slots);
```

A `cbuffer` member of a `[CSharp]` enum or struct gets a key of the C# twin, such as `ValueParameterKey<SlotChannel>`.

## Map an enum to a C# enum

An enum that C# already has, for example one the editor shows with `[Display]` attributes, is mapped with its full name. A nested type is written `Outer+Inner`:

```cs
[CSharp("MyGame.Rendering.HairShared+ShadingModel")]
enum class ShadingModel : int { Approximation, Improved, KajiyaKay };
```

The generator writes no C# type for it, and checks that every value of the shader enum is in the C# enum, with the same name and number, and that both are stored as the same type ([SDSL0288](../../../../diagnostics/SDSL0288.md)). The C# enum may have more values. If the difference is wanted, write `[CSharp("Full.Name", SkipValueCheck = true)]`. A name that C# doesn't have is [SDSL0287](../../../../diagnostics/SDSL0287.md).

A struct can't be mapped to a C# struct yet: it is always generated.

## Which C# types

| In the shader | In C# |
|---|---|
| `float`, `int`, `uint` | `float`, `int`, `uint` |
| `float2`, `float3`, `float4` | `Vector2`, `Vector3`, `Vector4` (`Color3`, `Color4` with `[Color]`) |
| `int2`, `int3`, `int4`, `uint4` | `Int2`, `Int3`, `Int4`, `UInt4` |
| `float4x4` | `Matrix` |
| a `[CSharp]` enum or struct | its twin |

A field of another type is an error with the type to use instead ([SDSL0289](../../../../diagnostics/SDSL0289.md)): `bool` (one byte in C#), `half`, `double`, `uint2` and `uint3`, a `row_major` matrix, other matrices, arrays. An enum or struct field needs a C# twin ([SDSL0290](../../../../diagnostics/SDSL0290.md)). A partial struct has none, as its fields depend on the effect. A field named like a C# keyword gets an `@` in C#.

An enum or struct without `[CSharp]` is for the shaders only: it has no C# type, and a `cbuffer` member of its type has no key. Wrong arguments of `[CSharp]` are [SDSL0286](../../../../diagnostics/SDSL0286.md).

A field or a key can have the type of a `[CSharp]` enum or struct of another shader file of the project, or of a twin that a referenced project generated.

## Layout

The twin has the tight layout of a structured buffer: each field after the previous one, aligned to 4 bytes, as Direct3D lays out a structured buffer. Direct3D 11, Direct3D 12 and Vulkan read a structured buffer of a `[CSharp]` struct with this layout, so C# fills it with no padding to write.

A cbuffer packs a struct in 16-byte rows: a field that would cross a row starts the next one. When this moves a field of a `[CSharp]` struct, the effect doesn't compile ([SDSL0291](../../../../diagnostics/SDSL0291.md)), and the message gives the padding field to add:

```cs
[CSharp] struct Light
{
    float3 direction;
    float pad0;        // color starts a 16-byte row in a cbuffer: the same offset in C#
    float3 color;
};
```

If another C# struct fills that cbuffer, write `[CSharp(SkipLayoutCheck = true)]` on the struct.

## See also

* [Enums](enums.md)
* [Shaders and modules](shaders-and-modules.md)
