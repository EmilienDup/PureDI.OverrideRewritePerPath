# Override rewrite still visits the graph once per dependency path: generation time doubles with every level of a shared graph under a `Func<T1, T>`

**Pure.DI 2.5.7**, .NET SDK 10.0.403, macOS. Repro project attached (`dotnet build`, about 70 s; the table below has the other sizes, up to 17 minutes for 31 classes).

Follow-up to #171. In 2.5.7 the generated code for a shared graph reached through a `Func<T1, T>` delegate is fine (a few hundred lines in every case below), but the time spent before code generation still grows exponentially with the depth of the graph. The time goes into the recursion of `GraphOverrider.Override`, the step that applies the delegate argument to the dependency graph of `T`.

## Minimal code

A binary diamond of **singletons**: `L0` takes a `string`, every `Lk` depends on two `L(k-1)`. The root creates `L26` through a `Func<string, L26>`.

```csharp
using Pure.DI;

var composition = new Composition();
Console.WriteLine(composition.Root.GetType().Name);

public sealed class L0(string label) { public string Label => label; }
public sealed class L1(L0 first, L0 second);
public sealed class L2(L1 first, L1 second);
// ... same pattern ...
public sealed class L26(L25 first, L25 second);
public sealed class Top(Func<string, L26> factory) { public L26 Make(string label) => factory(label); }

partial class Composition
{
    private static void Setup() => DI.Setup(nameof(Composition))
        .Bind<string>().To(_ => "default")
        .Bind<L0>().As(Lifetime.Singleton).To<L0>()
        // ... every Lk as Singleton ...
        .Bind<L26>().As(Lifetime.Singleton).To<L26>()
        .Root<Top>("Root");
}
```

## Expected

Generation time in the order of what the same graph costs when `L26` is the root directly: 2 s for every size below. The graph has 27 nodes; the rewrite under the delegate should not depend on the number of paths through them.

## Actual

| classes | root is `L_N` directly | root is `Top(Func<string, L_N>)` | generated lines (through the `Func`) |
|---|---|---|---|
| 21 | 2 s | 4 s | 580 |
| 23 | | 7 s | 610 |
| 25 | | 18 s | 640 |
| 27 (attached) | 2 s | 73 s | 670 |
| 29 | | 4 min 23 s | 700 |
| 31 | 2 s | 17 min 27 s | 730 |

Every level doubles the time: `L0` is reachable through 2^N paths and the rewrite is performed once per path, even though the result is the same node each time (the output has one initialisation per singleton).

Where the time goes (`dotnet-stack report` on the compiler process while it runs the attached project, same picture at every sample):

```
Pure.DI.Core!Pure.DI.Core.GraphOverrider.Rewrite
Pure.DI.Core!Pure.DI.Core.GraphOverrider.Override   (one frame per level of the diamond)
Pure.DI.Core!Pure.DI.Core.GraphOverrider.Override
...
System.Private.CoreLib!System.Buffer.BulkMoveWithWriteBarrier   (dictionary copies)
```

On a real application (about 2,500 bindings, view models created through `Func<...>` factories over a shared service layer) the 2.5.7 generator shows exactly this stack, 80 `Override` frames deep, for more than 40 minutes before we kill the build; 2.5.6 behaved the same. A real service layer is a dense graph, so the number of paths from a view model down to a leaf service is far larger than in this diamond.
