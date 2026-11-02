# Upgrading shaders from 4.4

Opening a project in Stride 4.5 upgrades its shaders, after a backup. This page says what changes in extension points, and what can change in what your shaders do.

## Extension points

Engine methods that are meant to be extended now say how: [Extension points](../manual/graphics/effects-and-shaders/shading-language/extension-points.md).

`override` and `base` stay valid, and they don't need a phase.

What changes:

- **A phase always wins.** Engine classes now run in their phase, wherever they are mixed. An effect that mixes `NormalBase` before `TransformationBase` now runs `TransformationBase` first in `VSMain`.
- **An override without a phase runs outside every phase.** If it doesn't call `base`, it drops the overrides with a phase, also the ones mixed after it. The compiler reports what it drops: [SDSL0216](../diagnostics/SDSL0216.md). Give it a phase to keep them.
- **Overriding a `module` method that is not an extension point** is error [SDSL0246](../diagnostics/SDSL0246.md). The upgrade adds the kind when the first definition is in your project: `[Pipeline]` if an override calls `base`, else `[Replace]`. If it is in a library, the upgrade notes say so, and its author has to add it, or you can write `patch`.
- **Two overrides of one phase whose order matters** need an order in an effect built by the asset compiler: [SDSL0137](../diagnostics/SDSL0137.md).

The upgrade writes `extend` for overrides that have a phase, or that override a `[Replace]` method, when `extend` can say the same thing. Shaders from 4.4 have no phases, so most of their other overrides stay as they are, and that is fine.
