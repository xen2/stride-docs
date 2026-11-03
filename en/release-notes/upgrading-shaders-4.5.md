# Upgrading shaders from 4.4

Opening a project in Stride 4.5 upgrades its shaders, after a backup. This page says what changes in extension points, and what can change in what your shaders do.

## Extension points

Engine methods that are meant to be extended now say how: [Extension points](../manual/graphics/effects-and-shaders/shading-language/extension-points.md).

`override` and `base` stay valid, and they don't need a phase.

What changes:

- **A phase always wins.** Engine classes now run in their phase, wherever they are mixed. An effect that mixes `NormalBase` before `TransformationBase` now runs `TransformationBase` first in `VSMain`.
- **An override without a phase runs outside every phase.** If it doesn't call `base`, it drops the overrides with a phase, also the ones mixed after it. What it drops is error [SDSL0216](../diagnostics/SDSL0216.md) in an effect built by the asset compiler, and information in a mix built at run time. Give it a phase and call `base` to keep them, or write `[Replaces]` to drop them. The upgrade writes `[Replaces]` when the class inherited what it drops: that parent is a module now, so it is imported, and the attribute keeps what 4.4 did.
- **Overriding a `module` method that is not an extension point** is error [SDSL0246](../diagnostics/SDSL0246.md). The upgrade adds the kind when the first definition is in your project: `[Pipeline]` if an override calls `base`, else `[Replace]`. If it is in a library, the upgrade notes say so: write `patch` instead of `override`, or ask its author to add the kind.
- **Two overrides of one phase whose order matters** need an order in an effect built by the asset compiler: [SDSL0137](../diagnostics/SDSL0137.md).
- **Two patches of one method** need an order in every effect: `after` on the patch, or an `order` line in the effect. [SDSL0148](../diagnostics/SDSL0148.md).

The upgrade writes `extend` for overrides that have a phase, or that override a `[Replace]` method, when `extend` can say the same thing. Shaders from 4.4 have no phases, so most of their other overrides stay as they are, and that is fine.

## Mistakes are errors

What Stride 4.4 reported as a warning, or let through silently, is now an error when it is a mistake in the shader. The upgrade fixes the ones a tool can fix:

| Code | Mistake | The upgrade |
|---|---|---|
| [SDSL0212](../diagnostics/SDSL0212.md) | `clone` on a method | removes it |
| [SDSL0214](../diagnostics/SDSL0214.md) | `partial` on a class | removes it |
| [SDSL0236](../diagnostics/SDSL0236.md) | a value on a `compose` | removes it |
| [SDSL0203](../diagnostics/SDSL0203.md) | `[Color]` on a member that is not `float3` or `float4` | removes it |
| [SDSL0204](../diagnostics/SDSL0204.md) to [SDSL0208](../diagnostics/SDSL0208.md) | `module` code that uses instance members, and overrides of the wrong kind | marks what it can as `module`, and adds `[Root]` to a class nothing uses in a composition |
| [SDSL0200](../diagnostics/SDSL0200.md), [SDSL0201](../diagnostics/SDSL0201.md), [SDSL0202](../diagnostics/SDSL0202.md) | an `override` or a `base` call with no inherited method, too many generic values | you fix it |
| [SDSL0209](../diagnostics/SDSL0209.md), [SDSL0210](../diagnostics/SDSL0210.md), [SDSL0211](../diagnostics/SDSL0211.md), [SDSL0213](../diagnostics/SDSL0213.md) | `[Pure]` rules | you fix it |
| [SDSL0215](../diagnostics/SDSL0215.md) | two structs of one name in a class | you fix it |
| [SDSL0216](../diagnostics/SDSL0216.md) | an override that drops others without saying so, in an effect built by the asset compiler | writes `[Replaces]` when the class inherited what it drops; otherwise you fix it: `[Replaces]` or `base` |
| [SDSL0217](../diagnostics/SDSL0217.md) | an attribute misspelled on a class or a `cbuffer` | you fix it |

A class that doesn't have to work in a composition can say so with `[Root]`: the rules of SDSL0204 to SDSL0208 don't apply to it. The upgrade adds it to a class that needs it, when nothing in the project uses that class in a composition and the upgrade knows all its parents. The upgrade can't see what C# or a tool does: if one of them mixes that class into a composition, the compiler reports [SDSL0119](../diagnostics/SDSL0119.md), and you have to change the module code.

[SDSL0219](../diagnostics/SDSL0219.md), [SDSL0220](../diagnostics/SDSL0220.md), [SDSL0232](../diagnostics/SDSL0232.md) and SDSL0242 to SDSL0244 stay warnings: they are about valid HLSL. Other unknown attributes on a class stay allowed, because libraries read attributes of their own.
