# Stride diagnostics

`Stride.Core.CompilerServices` contains Roslyn code analyzers.

> [!IMPORTANT]
> Analyzers are offline tools used by the compiler to assist programmers. They are not used to collect telemetry on how Stride is being used. These same analyzers are also utilized by your IDE to generate CSXXXX diagnostics.

They analyze the code for possible issues in your project with the `Stride.Core` design.
To avoid unexpected runtime/compile time/editor time behaviour these analyzers try to warn as soon as possible for issues that may occur.
Each of the following pages contain information about diagnostic codes that can be reported by the Roslyn analyzers, which are built into `Stride.Core.CompilerServices`.

The information covers:

- When is the diagnostics reported
- An explanation why it is necessary to report the diagnostic
- Examples when such a diagnostics occurs
- Information about how to resolve the diagnostic

If an error is reported it is possible to click in the IDE on the `diagnostic code` in the information box about the diagnostic. This will open the corresponding information page about the diagnostic.

The `Stride.Core.CompilerServices` is linked to `Stride.Core`, the diagnostic Analysis will only occur if your project references `Stride.Core` in the PackageReferences, this will automatically add the `Stride.Core.CompilerServices` to your project.

> [!WARNING]
> Note that diagnostic feature is experimental and may not work as expected. Warnings may contain solutions that don't work yet.

## Shader compiler (SDSL)

The shader compiler reports `SDSLxxxx` codes when it compiles a shader class (`.sdsl`) or builds an effect: in the asset build, in Game Studio and at run time. Codes from `SDSL0200` are warnings: the class still compiles, but it most likely doesn't do what you expect.

- [SDSL0048](SDSL0048.md): a reserved word used as a name (`triangle`, `point`, `texture`...)
- [SDSL0049](SDSL0049.md): code after the end of a shader
- [SDSL0050](SDSL0050.md): `compose` or `stream` inside a `cbuffer` or `rgroup`
- [SDSL0103](SDSL0103.md): an assignment to a value (`+x = 1`), not to a variable
- [SDSL0114](SDSL0114.md): implementation of an abstract method without `override`
- [SDSL0118](SDSL0118.md): texture or sampler generic parameter
- [SDSL0145](SDSL0145.md): a written swizzle that names an element twice
- [SDSL0150](SDSL0150.md): a member of a buffer, such as `.Length` (buffers have `GetDimensions` instead)
- [SDSL0155](SDSL0155.md): a matrix with one row or one column (`float1x3`, `float3x1`) is not supported yet
- [SDSL0156](SDSL0156.md): an initializer list with the wrong number of values
- [SDSL0157](SDSL0157.md): a texture, sampler or buffer in an initializer list
- [SDSL0160](SDSL0160.md): a call with too many or too few arguments
- [SDSL0161](SDSL0161.md): an argument that can't be converted to its parameter type
- [SDSL0162](SDSL0162.md): no overload can take the arguments of a call
- [SDSL0163](SDSL0163.md): an ambiguous call: several overloads match equally well
- [SDSL0164](SDSL0164.md): a value given for an `out` or `inout` parameter
- [SDSL0165](SDSL0165.md): a composition used as a value (call its `Compute()` method)
- [SDSL0166](SDSL0166.md): a call to an abstract method that no shader of the effect overrides
- [SDSL0170](SDSL0170.md): `[StreamOutput]` on a geometry shader (stream output) is not supported
- [SDSL0171](SDSL0171.md): a method attribute that the compiler doesn't support
- [SDSL0172](SDSL0172.md): a stream with an initial value
- [SDSL0173](SDSL0173.md): a member whose initial value is not a constant
- [SDSL0174](SDSL0174.md): a cbuffer with both stage and non-stage members
- [SDSL0175](SDSL0175.md): an override with another return type than the inherited method
- [SDSL0176](SDSL0176.md): an override with other parameters than the inherited method
- [SDSL0177](SDSL0177.md): not all code paths of a method return a value
- [SDSL0180](SDSL0180.md): an `AppendStructuredBuffer` or `ConsumeStructuredBuffer` method parameter
- [SDSL0181](SDSL0181.md): an `out` or `inout` buffer parameter
- [SDSL0185](SDSL0185.md): an `Interlocked*` destination that is not an `int` or `uint`
- [SDSL0186](SDSL0186.md): an `Interlocked*` destination that other threads don't see
- [SDSL0187](SDSL0187.md): an atomic operation on a `float`
- [SDSL0188](SDSL0188.md): a 64-bit atomic operation
- [SDSL0189](SDSL0189.md): a texture offset that is not a constant
- [SDSL0190](SDSL0190.md): an unknown type name (`Sampler` instead of `SamplerState`)
- [SDSL0191](SDSL0191.md): a parent shader that can't be found
- [SDSL0192](SDSL0192.md): `this.X` on a stage member, or `base.X` on a variable
- [SDSL0193](SDSL0193.md): a vector converted to a larger one (`float3` to `float4`)
- [SDSL0194](SDSL0194.md): a stream read before any stage writes it, with no semantic
- [SDSL0195](SDSL0195.md): an int, uint or bool matrix in a cbuffer or a buffer
- [SDSL0219](SDSL0219.md): comma expression that drops values (a vector type is missing)
- [SDSL0220](SDSL0220.md): `register(...)` or `packoffset(...)` is ignored
- [SDSL0232](SDSL0232.md): a comparison without parentheses inside `&`, `|` or `^`
- [SDSL0236](SDSL0236.md): the value written after a composition is ignored
- [SDSL0242](SDSL0242.md): `&` without parentheses inside `|`
- [SDSL0243](SDSL0243.md): `&&` without parentheses inside `||`
- [SDSL0244](SDSL0244.md): `+` or `-` without parentheses inside `<<` or `>>`

## References

- [Rule of Thumb Serialization](../manual/scripts/serialization.md#rule-of-thumb)
- [Serialisation](../manual/scripts/serialization.md)
