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

The shader compiler reports `SDSLxxxx` codes when it compiles a shader class (`.sdsl`): in the asset build, in Game Studio and at run time. Codes from `SDSL0200` are warnings: the class still compiles, but it most likely doesn't do what you expect. To make them fail the build (they will be errors in a later release), set the MSBuild property `StrideShaderWarningsAsErrors` to `true` in your project.

- [SDSL0116](SDSL0116.md): `override` with a signature that no inherited method has
- [SDSL0117](SDSL0117.md): too few generic values
- [SDSL0200](SDSL0200.md): `override` with no inherited method
- [SDSL0201](SDSL0201.md): `base` call with no inherited method
- [SDSL0202](SDSL0202.md): too many generic values
- [SDSL0203](SDSL0203.md): `[Color]` on a type that isn't a color

## References

- [Rule of Thumb Serialization](../manual/scripts/serialization.md#rule-of-thumb)
- [Serialisation](../manual/scripts/serialization.md)
