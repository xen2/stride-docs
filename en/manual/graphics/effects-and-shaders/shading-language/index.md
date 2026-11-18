# Shading language

Stride provides a superset of the [HLSL Shading language](http://msdn.microsoft.com/en-us/library/windows/desktop/bb509561%28v=vs.85%29.aspx), bringing advanced and higher level language constructions, with:

- **extensibility** to allow shaders to be extended easily using object-oriented programming concepts such as classes, inheritance, and composition
- **modularity** to provide a set modular shaders each focusing on a single rendering technique, more easily manageable
- **reusability** to maximize code reuse between shaders

Stride Shading Language (SDSL) is compiled to SPIR-V, which is converted for each graphics API (Direct3D 11, Direct3D 12, Vulkan).

## In this section

- [Shader classes, mixins, and inheritance](shader-classes-mixins-and-inheritance.md)
- [Composition](composition.md)
- [Templates](templates.md)
- [Shader stage input/output automatic management](automatic-shader-stage-input-output.md)
