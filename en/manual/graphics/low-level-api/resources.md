# Resource binding

<span class="badge text-bg-primary">Advanced</span>
<span class="badge text-bg-success">Programmer</span>

When [drawing vertices](draw-vertices.md) using an [effect](../effects-and-shaders/index.md), the shaders expect certain resources to be available, including:

- textures and buffers
- samplers
- constant buffers

## Automatic resource binding

The @'Stride.Rendering.EffectInstance' class handles the details of enumerating these resources from a loaded effect as well as binding them.

It exposes the @'Stride.Graphics.RootSignature', which has to be set as [pipeline state](pipeline-state.md),
and allows to fill constant buffers and bind resources based on a @'Stride.Rendering.ParameterCollection'.

**Code:** Using an EffectInstance

```cs
// Create a EffectInstance and use it to set up the pipeline
var effectInstance = new EffectInstance(EffectSystem.LoadEffect("MyEffect").WaitForResult());
pipelineStateDescription.EffectBytecode = effectInstance.Effect.Bytecode;
pipelineStateDescription.RootSignature = effectInstance.RootSignature;

// Update constant buffers and bind resources
effectInstance.Apply(context.GraphicsContext);
```

## Manual resource binding

When more optimized code is required (eg in the [rendering pipeline](../rendering-pipeline/index.md)), constant buffer updates and resource binding can be done manually.

## Push constants

Push constants are a few small values (32 bytes at most) that the shaders read without a constant buffer or a descriptor set: for example a draw index, an instance offset or a material index that changes at each draw. The shaders declare them in a `[PushConstant] cbuffer` (see [Push constants](../effects-and-shaders/shading-language/shaders-and-modules.md#push-constants) in the shading language).

Set them with `CommandList.SetPushConstants` before the draw or dispatch, with a struct that has the members of the effect's push-constant block, in the same order:

**Code:** Setting push constants

```cs
[StructLayout(LayoutKind.Sequential)]
struct PerDraw
{
    public uint MaterialIndex;
    public uint InstanceOffset;
}

commandList.SetPipelineState(pipelineState);
commandList.SetPushConstants(new PerDraw { MaterialIndex = 3, InstanceOffset = 0 });
commandList.DrawIndexed(indexCount);

commandList.SetPushConstants(new PerDraw { MaterialIndex = 4, InstanceOffset = 120 });
commandList.DrawIndexed(indexCount);
```

To change only some members, pass their bytes and their offset in the block, for example `commandList.SetPushConstants(MemoryMarshal.AsBytes(new ReadOnlySpan<uint>(in instanceOffset)), offset: 4)`. The other values are kept.

The values stay set until you change them: also when the pipeline state changes, and after the command list is flushed or reset. @'Stride.Graphics.CommandList.ClearState' sets them to zero. The command list gives them to the GPU before each draw or dispatch whose shaders read push constants: as push constants on Vulkan, root constants on Direct3D 12, and a constant buffer on Direct3D 11.

The layout of the block is in `EffectReflection.PushConstants`: each member with its offset and size.