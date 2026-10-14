# Effects and shaders

Stride uses a programmable shading pipeline. You can write [custom shaders](custom-shaders.md), create @'Stride.Graphics.Effect's from them, and use them for drawing. The @'Stride.Rendering.EffectSystem' class provides an easy way to load an effect.

## Load an effect

Use:

```cs
var myEffect = EffectSystem.LoadEffect("MyEffect").WaitForResult();
```

You can then bind the effect as [pipeline state](../low-level-api/pipeline-state.md).

An effect also often defines a set of parameters. To set these, you need to [bind resources](../low-level-api/resources.md) before drawing.

## Shaders

Shaders are written in the [Stride shading language (SDSL)](shading-language/index.md), an extension of `HLSL`. You build on the engine shaders instead of copying them: [inherit and override](shading-language/shader-classes-mixins-and-inheritance.md) their methods, use [modules](shading-language/shaders-and-modules.md) for code and state shared by the whole effect, plug shaders into [composition](shading-language/composition.md) slots, and let the compiler pass [streams](shading-language/automatic-shader-stage-input-output.md) between shader stages.

To start, follow the [Custom shaders](custom-shaders.md) tutorial and the [Quick tour](shading-language/quick-tour.md).

## Effects

[Effects](effect-language.md) in Stride use C#-like syntax to further combine shaders. They provide conditional composition of shaders to generate effect permutations.

As some platforms can't compile shaders at runtime (eg iOS, Android, etc), effect permutation files (`.sdeffectlog`) are used to enumerate all permutations ahead of time.

## Target everything

Each shader is compiled once, then effects mix and compose the compiled shaders into SPIR-V. Stride converts it automatically to the target graphics platform: `SPIR-V` for Vulkan, `HLSL` for Direct3D, and `MSL` for Metal.

## In this section

* [Custom shaders](custom-shaders.md)
* [Shading language (SDSL)](shading-language/index.md)
    - [Quick tour](shading-language/quick-tour.md)
    - [Shaders and modules](shading-language/shaders-and-modules.md)
    - [Inheritance and overrides](shading-language/shader-classes-mixins-and-inheritance.md)
    - [Composition](shading-language/composition.md)
    - [Streams and stage input/output](shading-language/automatic-shader-stage-input-output.md)
    - [Shader stages](shading-language/shader-stages.md)
    - [Generics](shading-language/templates.md)
* [Effects (SDFX)](effect-language.md)
* Tools
    - [Shader development with Visual Studio Code](shader-development-vscode.md)
    - [Compile shaders](compile-shaders.md)
