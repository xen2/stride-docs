# Enums

An enum is a closed set of named values. Use it for what a shader chooses between: a channel, a blend mode, a sampler. A value of an enum is written like in C#, and only a cast turns it into a number, so a channel can't be passed where a blend mode is expected.

```cs
[Pure]
module MaterialRecords
{
    enum class SlotChannel : uint { All, R, G, B, A };
    enum class SlotOp : uint { Multiply, Add };

    struct SlotRecord
    {
        float4 constant;
        SlotChannel channel;
        SlotOp op;
    };
};

shader MaterialSlots
{
    import MaterialRecords;

    float4 SelectChannel(float4 value, SlotChannel channel)
    {
        switch (channel)
        {
            case SlotChannel.All: return value;
            case SlotChannel.R: return value.rrrr;
            case SlotChannel.G: return value.gggg;
            case SlotChannel.B: return value.bbbb;
            case SlotChannel.A: return value.aaaa;
        }
        return value;
    }

    float4 Apply(SlotRecord slot, float4 value, float4 detail)
    {
        value = SelectChannel(value, slot.channel);
        return slot.op == SlotOp.Add ? value + detail : value * detail;
    }
};
```

## Declare an enum

An enum is declared in a shader or a module, like a struct ([SDSL0284](../../../../diagnostics/SDSL0284.md)). The classes that inherit or import that class see it.

- It is written `enum class` ([SDSL0270](../../../../diagnostics/SDSL0270.md)).
- The type after the colon is written, and is `uint` or `int` ([SDSL0271](../../../../diagnostics/SDSL0271.md)). A value is stored as that type, 32 bits on every graphics API.
- A value is an integer literal in the range of that type, decimal or hexadecimal, negative for an `int` enum. A value without a number is the number after the previous one, starting at 0 ([SDSL0272](../../../../diagnostics/SDSL0272.md)).
- Each value has its own name and its own number ([SDSL0273](../../../../diagnostics/SDSL0273.md), [SDSL0274](../../../../diagnostics/SDSL0274.md)).

```cs
enum class Bias : int { Low = -1, None, High };     // -1, 0, 1
enum class Flags : uint { Shadow = 0x1, Fog = 0x2 };
```

## Use an enum

- A value is written `SlotChannel.R` ([SDSL0278](../../../../diagnostics/SDSL0278.md) for `SlotChannel::R`, [SDSL0280](../../../../diagnostics/SDSL0280.md) for a value that doesn't exist). Where you write it, no local, parameter or member may have the name of the enum ([SDSL0279](../../../../diagnostics/SDSL0279.md)).
- An enum can be a local, a parameter (also `inout` and `out`), a return value, a `static const`, a struct field, a `cbuffer` member, a push constant, a stream, and the element of a `StructuredBuffer`. A stream of an enum type is not interpolated, like a `uint`.
- An enum takes `==` and `!=` with a value of the same enum, and the conditional operator `?:` ([SDSL0276](../../../../diagnostics/SDSL0276.md)).
- A value becomes a number only with a cast, and a number becomes a value only with a cast ([SDSL0275](../../../../diagnostics/SDSL0275.md)). An enum is cast from an `int` or a `uint`, and to a scalar number. A cast between two enums goes through the stored type: `(Other)(uint)value` ([SDSL0277](../../../../diagnostics/SDSL0277.md)).
- A method can have two overloads, one for `uint` and one for the enum: an enum never converts by itself, so a call picks the overload of its argument.
- A value read from a buffer is not checked when the shader runs: a number that is not a value of the enum is not one of its cases.

For flags, keep a `uint` field and cast the values of the enum:

```cs
bool hasShadow = (record.flags & (uint)Flags.Shadow) != 0;
```

## Switch on an enum

A `switch` on an enum takes values of that enum as cases ([SDSL0281](../../../../diagnostics/SDSL0281.md)). When it has no `default`, it names every value of the enum ([SDSL0282](../../../../diagnostics/SDSL0282.md)): adding a value to the enum then shows every `switch` that doesn't handle it.

A method that returns from every case of a `switch` with no `default` still needs a `return` after the `switch` ([SDSL0177](../../../../diagnostics/SDSL0177.md)), as in the example above.

## What enums can't be yet

- A generic parameter of an enum type ([SDSL0283](../../../../diagnostics/SDSL0283.md)). Declare `<uint C>` and cast it: `(SlotChannel)C`.
- The element of a typed buffer or a texture, such as `Buffer<SlotChannel>` ([SDSL0285](../../../../diagnostics/SDSL0285.md)). Read a `uint` and cast it. An `Interlocked` method doesn't take an enum either.
- An array index: cast the value, `values[(uint)channel]`.

## Enums in C#

An enum has no C# type by itself. Write `[CSharp]` before it to get one, or map it to a C# enum you wrote: see [Data shared with C#](shared-data.md). A `cbuffer` member of an enum without a C# type has no parameter key.

## See also

* [Data shared with C#](shared-data.md)
* [Shaders and modules](shaders-and-modules.md)
