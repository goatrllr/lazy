# Lazy Developer Reference

Design rationale, architecture, and extension guide for the lazy generator library.

---

## Rationale

Erlang provides excellent list operations for finite sequences through pattern matching and recursion, but lists require complete materialization in memory. Many real-world problems involve sequences that are too large to hold entirely in memory, or whose values are unknown until computed on-demand.

The `lazy` library fills this gap by enabling **lazy evaluation** — computing values only when requested, not upfront. This pattern enables:

- **Memory efficiency** — Process multi-gigabyte datasets using constant memory
- **Infinite sequences** — Represent endless mathematical sequences (Fibonacci, primes, natural numbers)
- **Composability** — Chain operations without intermediate materialization
- **Early termination** — Stop computation as soon as you have the answer

### The Generator Model

At the core is the generator — a zero-arity function representing a sequence. When called, it returns:
- `empty` — the sequence is exhausted
- `{Value, NextGenerator}` — the current value and a function for the next value

This model is simple, functional, and composes naturally with higher-order functions.

---

## Design Principles

### Pure Functional

All generator operations are pure functions with no side effects. Generators don't modify state, don't use processes, and don't rely on external resources (until materialized). This ensures:
- Predictability and testability
- Easy reasoning about behavior
- Safe to use in concurrent contexts

### Lazy Evaluation

Generator functions defer all computation until `next/1` is called. This means:
- Creating a generator has zero cost
- Chaining transformations builds a computation pipeline, not intermediate results
- Memory usage is constant regardless of sequence size

### Type Safety

All functions include `-spec` annotations enabling Dialyzer type checking:
- Type errors caught at compile time
- IDE support for argument checking
- Complete API documentation

### Finite vs. Infinite

The distinction between finite and infinite sequences is crucial:
- **Finite** sequences can be safely materialized with `to_list/1`, `foldl/3`, `length/1`
- **Infinite** sequences require lazy operations or `take/2` to limit results
- Predicates (`filter/2`, `filtermap/2`, `dropwhile/2`) on infinite sequences can hang if the condition never succeeds

---

## Architecture

### Generator Functions

The generator type is a zero-arity function:

```erlang
-type generator() :: fun(() -> empty | {term(), generator()}).
```

Each generator encapsulates:
1. Current state (captured in closure)
2. Logic to compute the next value
3. Return value and next generator

### Operation Categories

**Creation** — Build generators from scratch:
- `empty/0`, `once/1`, `repeat/1` — Static generators
- `seq/2,3`, `from_list/1` — From other sources
- `iterate/2`, `unfold/2`, `repeatedly/1`, `cycle/1` — Computed sequences

**Transformation** — Modify generator output:
- `map/2`, `filter/2`, `filtermap/2` — Value operations
- `drop/2`, `dropwhile/2`, `take/2`, `takewhile/2` — Limiting operations
- `scan/3`, `reverse/1` — Advanced operations
- `apply/2` — Meta-operation for custom transforms

**Combination** — Join multiple generators:
- `append/1,2` — Sequential concatenation
- `zip/2`, `zipwith/2,3` — Parallel combination
- `unzip/1` — Tuple deconstruction

**Materialization** — Convert to concrete values:
- `next/1` — Single-step materialization
- `to_list/1`, `foldl/3`, `foldr/3` — Complete materialization
- `foreach/2`, `flush/1` — Side-effect execution
- `all/2`, `any/2` — Predicate testing
- `length/1` — Counting

### Composition Strategy

Lazy operations compose by creating closures that capture parent generators:

Example: `map(F, Gen)` returns a function that, when called:
1. Calls `next(Gen)` to get the next value from the parent
2. If `empty`, returns `empty`
3. If `{V, NextGen}`, returns `{F(V), map(F, NextGen)}`

This pattern stacks cleanly, allowing chains like:
```erlang
Gen0 = lazy:map(fun double/1, Gen),
Gen1 = lazy:filter(fun is_even/1, Gen0),
Gen2 = lazy:take(10, Gen1).
```

---

## Project Structure

```
lazy/
├── src/
│   └── lazy.erl              # Main module: all public functions
├── test/
│   ├── lazy_SUITE.erl        # Common Test suites
│   └── ...
├── README.md                 # Quick start guide
├── docs/
│   ├── API_Documentation.md  # Function reference
│   ├── Developer_Reference.md # This file
│   └── Examples_&_Use_Cases.md # Code examples
└── Makefile                  # Build configuration
```

### Module Organization

**lazy.erl** — Contains all public functions organized by category:
- Creation helpers (lines ~50-200)
- Transformation operators (lines ~200-400)
- Combination functions (lines ~400-500)
- Materialization operations (lines ~500-700)
- Utility: `next/1`, `apply/2`

---

## Testing

### Test Structure

The `test/` directory contains:

**lazy_SUITE.erl** — Common Test suite with test groups:
- `creation_tests` — Generator creation
- `transformation_tests` — Value transforms
- `combination_tests` — Zipping, appending
- `materialization_tests` — Conversion to lists/values
- `infinite_tests` — Infinite sequence behavior
- `edge_case_tests` — Boundary conditions, empty, single-element generators

### Testing Patterns

**Finite sequences:**
```erlang
{ok, [1, 2, 3]} = {ok, lazy:to_list(lazy:seq(1, 3))}.
```

**Infinite sequence limiting:**
```erlang
Expected = [1, 2, 3],
Actual = lazy:to_list(lazy:take(3, lazy:iterate(fun (X) -> X + 1 end, 1))),
?assertEqual(Expected, Actual).
```

**Composition testing:**
```erlang
Gen0 = lazy:seq(1, 10),
Gen1 = lazy:filter(fun (X) -> X rem 2 =:= 0 end, Gen0),
Gen2 = lazy:map(fun (X) -> X * 10 end, Gen1),
Result = lazy:to_list(Gen2),
?assertEqual([20, 40, 60, 80, 100], Result).
```

**Custom generators:**
```erlang
Fib = fun Fib(A, B) -> fun () -> {A, Fib(B, A + B)} end end,
Result = lazy:to_list(lazy:take(5, Fib(0, 1))),
?assertEqual([0, 1, 1, 2, 3], Result).
```

### Running Tests

```bash
# Run all tests
make ct

# Run specific test
make ct SUITES=lazy_SUITE

# Specific test group
make ct SUITES=lazy_SUITE SPEC=creation_tests
```

---

## Comparison with Alternatives

### Erlang Lists

**Lists pros:**
- Built-in language feature
- Native pattern matching
- Extensive standard library
- Familiar to Erlang developers

**Lists cons:**
- Must materialize all values in memory
- Can't represent infinite sequences
- Operations create intermediate lists
- Poor performance on very large datasets

**lazy pros:**
- Constant memory for large/infinite sequences
- Natural composition without intermediates
- No memory overhead until materialization

### Elixir Streams

Elixir provides `Stream` module with similar lazy evaluation. Key differences:

**Streams (Elixir) advantages:**
- Language-integrated syntax
- Rich ecosystem of Stream-compatible functions
- Tight integration with Enum

**lazy advantages:**
- Pure Erlang (no Elixir runtime)
- Simpler model (single generator type)
- Direct composition without boilerplate
- Can integrate with existing Erlang code

### Custom Generator Patterns

Before `lazy`, Erlang developers built custom generators using:
- Recursion with helper functions
- Process-based state machines (via `gen_server`)
- Accumulator-based iteration

**lazy advantages:**
- Reusable, composable building blocks
- Proven patterns (map, filter, fold, zip)
- Type-safe operations
- Clear error handling

---

## Design Decisions

### Single Generator Type

All generators use the same zero-arity function type. This enables:
- Universal composition (any gen × any operation)
- No type conversions or wrapping
- Simple, predictable behavior

Alternative: Separate types for finite and infinite (rejected)
- Adds complexity
- Runtime type checking required
- Limits composition flexibility

### Eager Evaluation of Operation Specs

Functions like `map/2` and `filter/2` return immediately with a new generator, not the eventual results:

```erlang
Gen1 = lazy:map(fun double/1, Gen0),  % Instant
Result = lazy:to_list(Gen1),           % Computation happens here
```

Alternative: Return the results directly (rejected)
- Would break composability
- Can't handle infinite sequences
- Loses the lazy evaluation benefit

### Predicates with Infinite Sequences

`filter/2` and `filtermap/2` can hang on infinite sequences if the predicate never succeeds. We decided not to add timeouts or "maximum depth" because:

1. **Principle** — Pure functions shouldn't make decisions about evaluation
2. **Predictability** — Behavior matches user expectations
3. **Alternative** — Use `take/2` to limit exploration before filtering

### Materialization Functions

`reverse/1`, `foldr/3`, `all/2`, `any/2` materialize the entire sequence or large portions. This is intentional:

- `reverse/1` — Inherently requires full materialization
- `foldr/3` — Right fold requires traversal to the end
- `all/2`, `any/2` — Can provide immediate feedback on infinite sequences

---

## Extensions and Ideas

### Possible Future Enhancements

**Lazy Module Functions** — Functions that work on generators without materializing:
- `lazy:group_by/2` — Group consecutive equal values
- `lazy:partition/2` — Split into two based on predicate
- `lazy:unique/1` — Remove consecutive duplicates
- `lazy:intersperse/2` — Insert separator between values

**Performance Optimizations:**
- Caching intermediate values in long chains
- Limiting closure nesting depth
- Tail call optimization verification

**Error Handling:**
- Exception handling in generators
- Propagation semantics for chained operations
- Recovery strategies for partial failures

**Meta-operations:**
- Generator inspection (is_finite? is_empty?)
- Generator composition statistics
- Debugger-friendly representations

### Custom Generator Patterns

Users can create custom generators for domain-specific sequences:

```erlang
% User-defined Fibonacci generator
fib(A, B) ->
    fun () ->
        {A, fib(B, A + B)}
    end.

% Usage
Gen = lazy:take(10, fib(0, 1)),
lazy:to_list(Gen).  % [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

Custom generators integrate seamlessly with standard operations:

```erlang
Gen0 = fib(0, 1),
Gen1 = lazy:filter(fun (X) -> X > 10 end, Gen0),
Gen2 = lazy:take(5, Gen1),
lazy:to_list(Gen2).  % [13, 21, 34, 55, 89]
```

---

## Performance Characteristics

### Time Complexity

| Operation | Complexity |
|-----------|-----------|
| Creation (`empty`, `once`, `repeat`) | O(1) |
| `next/1` on simple gen | O(1) |
| `next/1` on chained ops | O(k) where k = chain depth |
| `map/2`, `filter/2` | O(1) creation, O(k) per value |
| `take/2` | O(1) creation, O(n) consumption |
| `to_list/1` on finite | O(n) |
| `foldl/3` | O(n) |

### Memory Complexity

| Operation | Memory |
|-----------|--------|
| Generator creation | O(k) where k = operations chained |
| Single value materialization | O(k) for stack frames |
| Full materialization (`to_list`) | O(n) for result list |
| Infinite sequence (limited) | O(min(n, limit)) |

### Optimization Tips

1. **Avoid unnecessary operations** — Each chained operation adds closure overhead
   ```erlang
   % Good: single pass
   lazy:map(fun (X) -> X * 2 + 10 end, Gen)
   
   % Less efficient: two passes
   lazy:map(fun (X) -> X + 10 end, lazy:map(fun (X) -> X * 2 end, Gen))
   ```

2. **Use `filtermap/2` for combined operations**
   ```erlang
   % Efficient: single closure
   lazy:filtermap(Fn, Gen)
   
   % Less efficient: two closures
   lazy:map(Fn, lazy:filter(Pred, Gen))
   ```

3. **Limit before expensive operations**
   ```erlang
   % Good: limit early
   lazy:to_list(lazy:take(100, lazy:map(expensive_fn, gen)))
   
   % Bad: compute everything before limiting
   Gen0 = lazy:take(100, lazy:map(expensive_fn, gen)),
   lazy:to_list(Gen0)
   ```

---

## Function Reference

See [API_Documentation.md](API_Documentation.md) for complete function descriptions, type signatures, and examples.

### Quick Index by Category

**Generator Creation:**
`empty/0`, `once/1`, `repeat/1`, `seq/2`, `seq/3`, `from_list/1`, `iterate/2`, `unfold/2`, `repeatedly/1`, `cycle/1`

**Transformation:**
`map/2`, `filter/2`, `filtermap/2`, `apply/2`, `drop/2`, `dropwhile/2`, `take/2`, `takewhile/2`, `scan/3`, `reverse/1`

**Combination:**
`append/1`, `append/2`, `zip/2`, `zipwith/2`, `zipwith/3`, `unzip/1`

**Materialization:**
`next/1`, `to_list/1`, `foldl/3`, `foldr/3`, `foreach/2`, `flush/1`, `all/2`, `any/2`, `length/1`

---

## See Also

- [API_Documentation.md](API_Documentation.md) — Complete function reference
- [Examples_&_Use_Cases.md](Examples_&_Use_Cases.md) — Code examples and patterns
- [README.md](../README.md) — Quick start guide

---

**Lazy Authors:**
* Maria Scott ([Maria-12648430](https://github.com/Maria-12648430))
* Jan Uhlig ([juhlig](https://github.com/juhlig))

**Documentation Author:** goatrllr ([https://github.com/goatrllr](https://github.com/goatrllr))
