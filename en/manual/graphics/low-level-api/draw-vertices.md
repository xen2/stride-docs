# Draw vertices

<span class="badge text-bg-primary">Advanced</span>
<span class="badge text-bg-success">Programmer</span>

When loading a scene, Stride automatically handles the draw calls to display the scene throughout the entity system. This page introduces manual drawing.

## Primitives

Stride provides the following set of built-in primitives:

- Plane
- Cube
- Sphere
- Geosphere
- Cylinder
- Torus
- Teapot

They aren't automatically created along with the @'Stride.Graphics.GraphicsDevice', so you have to instantiate them. You can do this through the @'Stride.Graphics.GeometricPrimitives.GeometricPrimitive' class.

**Code:** Creating and using a primitive

```cs
// creation
var myCube = GeometricPrimitive.Cube.New(GraphicsDevice);
var myTorus = GeometricPrimitive.Torus.New(GraphicsDevice);
 
// ...
 
// draw one on screen
myCube.Draw(CommandList, EffectInstance);
```

They have no effect associated with them, so the user has to provide an @'Stride.Rendering.EffectInstance' when drawing. For information on loading effects, please see [Effects and shaders](../effects-and-shaders/index.md).

## Custom drawing

Outside of built-in primitives, any geometry can be drawn by creating custom vertex buffers. To create a vertex buffer, first a @'Stride.Graphics.VertexDeclaration' has to be defined. A vertex declaration describes the elements of each vertex and their layout.
For details, see the @'Stride.Graphics.VertexElement' reference page.

Next, a vertex buffer can be created from an array of vertices. The vertex data type has to match the @'Stride.Graphics.VertexDeclaration'.

Given vertex buffer and declaration, a @'Stride.Graphics.VertexBufferBinding' can be created. 

**Code:** Creating a vertex buffer

```cs
// Create a vertex layout with position and texture coordinate
var layout = new VertexDeclaration(VertexElement.Position<Vector3>(), VertexElement.TextureCoordinate<Vector2>()); 
 
// Create the vertex buffer from an array of vertices
var vertices = new VertexPositionTexture[vertexCount];
var vertexBuffer = Buffer.Vertex.New(GraphicsDevice, vertices);
 
// Create a vertex buffer binding
var vertexBufferBinding = new VertexBufferBinding(vertexBuffer, layout, vertexCount);
```

To draw the newly created vertex buffer, it has to be bound to the pipeline. The vertex layout and the @'Stride.Graphics.PrimitiveType' to draw have to be included in the [pipeline state](pipeline-state.md) object. The buffer itself can be set dynamically.

Afterwards, the vertices are ready to be rendered using @'Stride.Graphics.CommandList.Draw(System.Int32,System.Int32)'.

**Code:** Binding and drawing vertex buffers

```cs
// Set the pipeline state
pipelineStateDescription.InputElements = vertexBufferBinding.Layout.CreateInputElements();
pipelineStateDescription.PrimitiveType = PrimitiveType.TriangleStrip;
 
// Create and set a PipelineState object
// ...

// Bind the vertex buffer to the pipeline
commandList.SetVertexBuffers(0, vertexBuffer, 0, vertexBufferBinding.Stride);
 
// Draw the vertices
commandList.Draw(vertexCount);
```

It is also possible to draw indexed geometry. To use an index buffer, first create it similarly to the vertex buffer and bind it to the pipeline.
It can then be used for drawing using @'Stride.Graphics.CommandList.DrawIndexed(System.Int32,System.Int32,System.Int32)'.

**Code:** Drawing indexed vertices

```cs
// Create the index buffer
var indices = new short[indexCount];
var is32Bits = false;
var indexBuffer = Buffer.Index.New(GraphicsDevice, indices);
 
// set the VAO
commandList.SetIndexBuffer(indexBuffer, 0, is32Bits);

// Draw indexed vertices
commandList.DrawIndexed(indexBuffer.ElementCount);
```

## Indirect draws and dispatches

The arguments of a draw or a dispatch can come from a buffer instead of the CPU. A compute shader can then decide how many instances to draw or how many thread groups to run, without a round trip to the CPU.

Create the argument buffer with @'Stride.Graphics.Buffer.Argument'. To let a compute shader write it, pass `unorderedAccess: true`: shaders write it as a `RWBuffer<uint>`.

| Call | Arguments (one `uint` each) |
|---|---|
| @'Stride.Graphics.CommandList.DrawInstanced(Stride.Graphics.Buffer,System.Int32)' | vertex count per instance, instance count, start vertex, start instance |
| @'Stride.Graphics.CommandList.DrawIndexedInstanced(Stride.Graphics.Buffer,System.Int32)' | index count per instance, instance count, start index, base vertex (signed), start instance |
| @'Stride.Graphics.CommandList.Dispatch(Stride.Graphics.Buffer,System.Int32)' | thread group count in X, Y and Z |

The second parameter is the byte offset of the arguments in the buffer.

**Code:** Dispatch with arguments written by a compute shader

```cs
// 3 uints, writable by compute shaders
var arguments = Buffer.Argument.New(GraphicsDevice, 3 * sizeof(uint), unorderedAccess: true);

// ... a compute shader writes the thread group counts into "arguments" ...

// Make the write visible to the indirect call
commandList.ResourceBarrierTransition(arguments, BarrierLayout.IndirectArgument);

// With a ComputeEffectShader
myComputeEffect.IndirectArguments = arguments;
myComputeEffect.Draw(renderDrawContext);

// Or directly, after setting the pipeline state and the resources
commandList.Dispatch(arguments, 0);
```

> [!Note]
> After a shader writes the argument buffer, transition it to `BarrierLayout.IndirectArgument` before the indirect call. The command list doesn't do it for you.

## Dispatches that read what an earlier dispatch wrote

When a dispatch reads or writes a buffer or texture that an earlier dispatch wrote as unordered access, call @'Stride.Graphics.CommandList.UnorderedAccessBarrier(Stride.Graphics.GraphicsResource)' between them. The resource stays in `BarrierLayout.UnorderedAccess`, so a transition can't order the two calls, and without the barrier the second dispatch can read old values.

**Code:** Count, then use the counts

```cs
// The first pass writes the counts
countEffect.Parameters.Set(MyCountShaderKeys.Counts, counts);
countEffect.Draw(renderDrawContext);

// Make the writes visible to the next pass
commandList.UnorderedAccessBarrier(counts);

// The second pass reads them
scatterEffect.Parameters.Set(MyScatterShaderKeys.Counts, counts);
scatterEffect.Draw(renderDrawContext);
```

When a pass writes several resources, @'Stride.Graphics.CommandList.UnorderedAccessBarrier' without an argument covers all of them; they must already be in `BarrierLayout.UnorderedAccess`. On Direct3D 11, both calls do nothing: the runtime orders the dispatches.

> [!Note]
> `ClearReadWrite` needs no barrier: it is ordered with the shader accesses before and after it.

## Read vertex and index buffers in shaders

A vertex or index buffer created with `BufferFlags.ShaderResource | BufferFlags.RawBuffer` can also be read by shaders as a `ByteAddressBuffer`, for example to fetch vertices in a compute shader. Its size must be a multiple of 4 bytes.

**Code:** Shader-readable vertex and index buffers

```cs
var flags = BufferFlags.ShaderResource | BufferFlags.RawBuffer;
var vertexBuffer = Buffer.Vertex.New(GraphicsDevice, vertices, additionalFlags: flags);

// 16-bit indices: use an even index count, so the size is a multiple of 4 bytes
var indexBuffer = Buffer.Index.New(GraphicsDevice, indices, additionalFlags: flags);
```

**Code:** Fetching a vertex in a shader

```hlsl
stage ByteAddressBuffer Vertices;
stage ByteAddressBuffer Indices;

// 16-bit index n, starting at byte offset IndexOffset
uint address = IndexOffset + n * 2;
uint index = (Indices.Load(address & ~3u) >> ((address & 2u) * 8)) & 0xFFFFu;

// Position at the start of each vertex
float3 position = asfloat(Vertices.Load3(index * VertexStride));
```

For a model, check **Shader readable buffers** in the model asset properties: the model's vertex and index buffers are then built with these flags. The mesh bindings give the byte offsets and strides (`MeshDraw.VertexBuffers[i].Offset` and `.Stride`, `MeshDraw.IndexBuffer.Offset` and `.Is32Bit`).
