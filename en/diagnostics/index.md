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

The shader compiler reports `SDSLxxxx` codes when it compiles a shader class (`.sdsl`) or builds an effect: in the asset build, in Game Studio and at run time. Codes from `SDSL0200` are warnings, and so is SDSL0114 since Stride 4.5: the class still compiles, but it most likely doesn't do what you expect. To make them fail the build (they will be errors in a later release), set the MSBuild property `StrideShaderWarningsAsErrors` to `true` in your project.

- [SDSL0046](SDSL0046.md): `mixin clone` in an effect
- [SDSL0047](SDSL0047.md): `mixin remove` in an effect
- [SDSL0048](SDSL0048.md): a reserved word used as a name (`triangle`, `point`, `texture`...)
- [SDSL0049](SDSL0049.md): code after the end of a shader
- [SDSL0050](SDSL0050.md): `compose` or `stream` inside a `cbuffer` or `rgroup`
- [SDSL0114](SDSL0114.md): implementation of an abstract method without `override` (an error in Stride 4.4, a warning since Stride 4.5)
- [SDSL0116](SDSL0116.md): `override` with a signature that no inherited method has
- [SDSL0117](SDSL0117.md): too few generic values
- [SDSL0118](SDSL0118.md): texture or sampler generic parameter
- [SDSL0119](SDSL0119.md): root-only shader in a composition
- [SDSL0120](SDSL0120.md): the same field in two parts of a partial struct
- [SDSL0121](SDSL0121.md): `extend struct` with no partial struct to extend
- [SDSL0122](SDSL0122.md): `extend struct` of a struct that is not partial
- [SDSL0123](SDSL0123.md): a second declaration of a partial struct
- [SDSL0124](SDSL0124.md): two partial structs with the same name
- [SDSL0125](SDSL0125.md): partial struct in a stream, cbuffer, buffer or groupshared variable
- [SDSL0126](SDSL0126.md): partial struct in a generic class
- [SDSL0128](SDSL0128.md): a struct that contains itself
- [SDSL0129](SDSL0129.md): push-constant member of an unsupported type
- [SDSL0130](SDSL0130.md): push-constant cbuffer with instance members
- [SDSL0131](SDSL0131.md): push constants larger than 32 bytes
- [SDSL0132](SDSL0132.md): cbuffer name with and without `[PushConstant]`
- [SDSL0133](SDSL0133.md): no constant buffer register left for push constants on Direct3D 11
- [SDSL0134](SDSL0134.md): misaligned vector in the push constants
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
- [SDSL0200](SDSL0200.md): `override` with no inherited method
- [SDSL0201](SDSL0201.md): `base` call with no inherited method
- [SDSL0202](SDSL0202.md): too many generic values
- [SDSL0203](SDSL0203.md): `[Color]` on a type that isn't a color
- [SDSL0204](SDSL0204.md): module code reads an instance member
- [SDSL0205](SDSL0205.md): module code calls an instance method
- [SDSL0206](SDSL0206.md): plain `override` of a module method
- [SDSL0207](SDSL0207.md): `override module` of an instance method
- [SDSL0208](SDSL0208.md): `base` call to a module method from instance code
- [SDSL0209](SDSL0209.md): state in a `[Pure]` module
- [SDSL0210](SDSL0210.md): `[Pure]` module uses a module that isn't pure
- [SDSL0211](SDSL0211.md): `[Pure]` on a shader
- [SDSL0212](SDSL0212.md): `clone` has no effect
- [SDSL0213](SDSL0213.md): pixel shader operation in a `[Pure]` module
- [SDSL0214](SDSL0214.md): `partial` on a shader or module
- [SDSL0215](SDSL0215.md): two structs with the same name
- [SDSL0216](SDSL0216.md): a shader loaded at run time, written for an older Stride version
- [SDSL0219](SDSL0219.md): comma expression that drops values (a vector type is missing)
- [SDSL0220](SDSL0220.md): `register(...)` or `packoffset(...)` is ignored
- [SDSL0232](SDSL0232.md): a comparison without parentheses inside `&`, `|` or `^`
- [SDSL0236](SDSL0236.md): the value written after a composition is ignored

## References

- [Rule of Thumb Serialization](../manual/scripts/serialization.md#rule-of-thumb)
- [Serialisation](../manual/scripts/serialization.md)
