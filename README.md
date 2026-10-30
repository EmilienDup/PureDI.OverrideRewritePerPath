# Pure.DI 2.5.7: override rewrite is still once per dependency path

Minimal repro for Pure.DI 2.5.7: a 27-class singleton diamond generates in 2 s as a direct root but takes about 70 s when reached through a `Func<string, L26>` delegate, and every added level doubles that time. The generated code is small in both cases; the time is spent in the override rewrite before code generation. Follow-up to #171.

[`ISSUE.md`](ISSUE.md) is the text to paste into a Pure.DI GitHub issue; it includes the measurements from 21 to 31 classes.

```
dotnet build
```

To compare with the direct root, replace `.Root<Top>("Root")` with `.Root<L26>("Root")` and remove the `Top` class. Add `L27`, `L28` (and their bindings) to see the time double per level.
