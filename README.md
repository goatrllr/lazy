# Lazy

**Lazy** is an Erlang library providing lazy sequences and generators for memory-efficient processing of large or infinite data streams.

Erlang is inherently eager—it materializes all terms, including lists, into memory immediately. **Lazy** enables lazy evaluation: sequences that generate values only when accessed, making it possible to work with:
- **Large datasets** without loading everything into RAM
- **Infinite sequences** for mathematical or algorithmic exploration
- **Efficient stream processing** with map, filter, and other functional operations

---

## Features

✅ **Lazy Generators** — Create sequences that compute values on-demand instead of upfront

✅ **Rich Function Library** — Map, filter, fold, zip, scan, and 20+ generator combinators

✅ **Infinite Sequences** — Define and work with unbounded streams (repeat, iterate, cycle)

✅ **Memory Efficient** — Process multi-gigabyte datasets without loading into memory

✅ **Composable** — Chain generators together for complex data transformations

✅ **Pure Functional** — No side effects, idiomatic Erlang patterns

---

## Use Cases

- **Large file processing** — Stream files without loading entirely into memory
- **Infinite sequences** — Fibonacci, mathematical sequences, periodic timers
- **Data transformation pipelines** — Map/filter/fold chains for ETL workflows
- **Real-time data streams** — Parse and process unbounded data sources
- **Memory-constrained environments** — Embedded systems with limited RAM
- **Exploratory analysis** — Quickly iterate over generated sequences

---

## Quick Start

### Installation

Add to `rebar.config`:
```erlang
{deps, [
    {lazy, {git, "https://github.com/goatrllr/lazy.git", {branch, "main"}}}
]}.
```

### Basic Usage

Create a generator and materialize it into a list:

```erlang
1> Gen = lazy:seq(1, 5).
#Fun<lazy.43.73700886>

2> lazy:to_list(Gen).
[1, 2, 3, 4, 5]
```

Map over a sequence:

```erlang
1> Gen0 = lazy:seq(1, 3).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:map(fun (X) -> X * 2 end, Gen0).
#Fun<lazy.21.73700886>

3> lazy:to_list(Gen1).
[2, 4, 6]
```

Filter generators:

```erlang
1> Gen0 = lazy:seq(1, 10).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:filter(fun (X) -> X rem 2 =:= 0 end, Gen0).
#Fun<lazy.16.73700886>

3> lazy:to_list(Gen1).
[2, 4, 6, 8, 10]
```

### Key Patterns

**Generate sequences with unfold:**
```erlang
Gen = lazy:unfold(fun (0) -> empty; (N) -> {N, N div 2} end, 256).
lazy:to_list(Gen). % [256, 128, 64, 32, 16, 8, 4, 2, 1]
```

**Infinite sequences with repeat or iterate:**
```erlang
Ones = lazy:repeat(1).
Powers = lazy:iterate(fun (X) -> X * 2 end, 1).
lazy:to_list(lazy:take(5, Powers)). % [1, 2, 4, 8, 16]
```

**Combine generators with zip/append:**
```erlang
Gen0 = lazy:seq(1, 3).
Gen1 = lazy:seq(4, 6).
G = lazy:zipwith(fun (A, B) -> A + B end, Gen0, Gen1).
lazy:to_list(G). % [5, 7, 9]
```

**Process streams without materializing:**
```erlang
Gen = lazy:seq(1, 1000000).
lazy:foreach(fun (X) -> process_item(X) end, Gen).
```

---

## Documentation

- **[API_Documentation.md](docs/API_Documentation_lazy.md)** — Complete function reference, types, and modules
- **[Developer_Reference.md](docs/Developer_Reference_lazy.md)** — Design rationale, architecture, testing, extensions
- **[Examples_&_Use_Cases.md](docs/Examples_&_Use_Cases_lazy.md)** — Key patterns and real-world code examples

---

## API Overview

### Generator Creation
- `append/1,2` — Concatenate multiple generators
- `cycle/1` — Repeat a list infinitely
- `empty/0` — Empty generator
- `from_list/1` — Wrap a list as a generator
- `iterate/2` — Generate by repeatedly applying a function
- `once/1` — Single-value generator
- `repeat/1` — Infinite repetition of a value
- `repeatedly/1` — Call a function repeatedly
- `seq/2,3` — Generate integer sequence
- `unfold/2` — Generate from accumulator and function

### Transformation
- `apply/2` — Apply function and return generator
- `drop/2` — Skip first N values
- `dropwhile/2` — Skip values while predicate is true
- `filter/2` — Keep values matching predicate
- `filtermap/2` — Filter and map in one step
- `map/2` — Transform each value
- `reverse/1` — Reverse finite sequence
- `scan/3` — Like foldl but return intermediate values
- `take/2` — Limit to first N values
- `takewhile/2` — Take values while predicate is true

### Combination
- `unzip/1` — Unzip tuples into separate generators
- `zip/2` — Combine two generators
- `zipwith/2,3` — Combine with a function

### Materialization
- `all/2` — Test if all values match predicate
- `any/2` — Test if any value matches predicate
- `foldl/3` — Fold left over sequence
- `foldr/3` — Fold right over sequence
- `flush/1` — Execute generator for side effects
- `foreach/2` — Apply function for side effects
- `length/1` — Count values in sequence
- `to_list/1` — Materialize to list

### Core
- `next/1` — Get next value and remainder generator

---

## Comparison with Alternatives

| Feature | Lazy | Erlang Lists | Elixir Streams |
|---------|------|--------------|----------------|
| Lazy evaluation | ✅ | ❌ | ✅ |
| Infinite sequences | ✅ | ❌ | ✅ |
| Pure Erlang | ✅ | ✅ | ❌ |
| Memory efficient | ✅ | ❌ | ✅ |
| Familiar functions | ✅ | ✅ | ✅ |
| Composable pipelines | ✅ | ⚠️ | ✅ |

---

## Testing

Comprehensive test coverage including:
- **Unit tests** — Core generator and materializer operations
- **Edge cases** — Empty sequences, infinite sequences, error conditions
- **Custom generators** — User-defined generator patterns

Run tests:
```bash
make tests      # All tests
make eunit      # Unit tests only
```

---

## Warnings

**Infinite sequences with predicates** — Using `filter`, `filtermap`, or `dropwhile` on infinite sequences can hang if the predicate never succeeds (or never fails for `dropwhile`):

```erlang
%% This will hang forever:
Gen = lazy:filter(fun (X) -> is_atom(X) end, lazy:seq(1, infinity)).
lazy:next(Gen). % Hangs - predicate never succeeds
```

**Finite vs. Infinite** — Many functions like `to_list/1` and `reverse/1` only work with finite sequences. Using them on infinite sequences will hang or consume unbounded memory.

---

## FAQ

**Q: How do I create custom generators?**
A: Write a zero-arity function that returns either `empty` or a tuple `{Value, NextGenerator}`. See [Examples_&_Use_Cases.md](docs/Examples_&_Use_Cases.md#custom-generators).

**Q: What if I use `to_list/1` on an infinite sequence?**
A: It will hang forever trying to materialize all values. Always use `take/2` to limit infinite sequences before materializing.

**Q: Can I use `lazy` with streams from files or sockets?**
A: Yes! Create a custom generator that reads chunks on-demand from your source. See [Developer_Reference.md](docs/Developer_Reference.md).

**Q: What's the performance impact?**
A: Lazy generators add minimal overhead. Each `next/1` call invokes a function and pattern matches. For large datasets, the memory savings typically outweigh the function call cost.

---

## License

Unlicensed (check included LICENSE file for details)

---

## Contributing

Contributions welcome! Please:
- Write tests for new functionality
- Follow idiomatic Erlang style
- Update documentation for public functions

---

## Acknowledgments

**Lazy Authors:**
- Maria Scott ([Maria-12648430](https://github.com/Maria-12648430))
- Jan Uhlig ([juhlig](https://github.com/juhlig))

**Documentation Author:** goatrllr ([https://github.com/goatrllr](https://github.com/goatrllr))
