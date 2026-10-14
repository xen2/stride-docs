# Stride shading language (SDSL)

Stride shaders are written in the Stride shading language (SDSL), saved in `.sdsl` files. SDSL is [HLSL](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl) with additions that let you build on the engine shaders instead of copying them:

- **shaders** inherit other shaders and override their methods, so you change only what you need;
- **modules** hold functions and state that exist once per effect, used with `import`;
- **compositions** are slots in a shader where another shader is plugged in, such as the nodes of a material;
- **streams** pass values between the vertex, pixel and other shader stages without writing input and output structures;
- **generics** are parameters of a shader that are fixed when the effect is compiled.

Stride compiles each `.sdsl` file once. An [effect](../effect-language.md) then mixes and composes the compiled shaders into the final shader, as SPIR-V, which is converted for each graphics API (Direct3D, Vulkan, Metal).

New to SDSL? Start with the [Quick tour](quick-tour.md).

> [!Note]
> Stride 4.5 changed SDSL: the `stage` modifier is now `module`, and modules are declared with `module` and used with `import`. The project upgrade rewrites shaders written for older versions. See [Shaders and modules](shaders-and-modules.md).

## In this section

- [Quick tour](quick-tour.md)
- [Shaders and modules](shaders-and-modules.md)
- [Inheritance and overrides](shader-classes-mixins-and-inheritance.md)
- [Composition](composition.md)
- [Streams and stage input/output](automatic-shader-stage-input-output.md)
- [Shader stages](shader-stages.md)
- [Generics](templates.md)
