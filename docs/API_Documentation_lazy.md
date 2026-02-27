# Lazy API Documentation

Complete reference for lazy generator functions, types, and modules.

---

## Description

Lazy generators are functions that produce values on-demand instead of all at once. This module provides tools to create, transform, and materialize lazy sequences in Erlang.

A generator is a zero-arity function that returns either:
- `empty` — indicating the sequence is exhausted
- `{Value, NextGenerator}` — the current value and a function to get the next value

---

## Modules

**lazy.erl** — Main module with generator creation, transformation, and materialization functions

---

## Function Reference

### Summary

| Function | Purpose | Category |
|----------|---------|----------|
| `empty/0` | Empty generator | Generator Creation |
| `once/1` | Single-value generator | Generator Creation |
| `repeat/1` | Infinite repetition | Generator Creation |
| `seq/2,3` | Integer range sequence | Generator Creation |
| `from_list/1` | List to generator | Generator Creation |
| `iterate/2` | Function application | Generator Creation |
| `unfold/2` | Accumulator-based generation | Generator Creation |
| `repeatedly/1` | Call function repeatedly | Generator Creation |
| `cycle/1` | Circular list repetition | Generator Creation |
| `append/1,2` | Concatenate generators | Combination |
| `zip/2` | Combine two generators | Combination |
| `zipwith/2,3` | Combine with function | Combination |
| `unzip/1` | Unzip tuples | Combination |
| `map/2` | Transform values | Transformation |
| `filter/2` | Keep matching values | Transformation |
| `filtermap/2` | Filter and map | Transformation |
| `apply/2` | Apply and wrap | Transformation |
| `drop/2` | Skip N values | Transformation |
| `dropwhile/2` | Skip while predicate | Transformation |
| `take/2` | Limit to N values | Transformation |
| `takewhile/2` | Take while predicate | Transformation |
| `scan/3` | Fold with intermediate values | Transformation |
| `reverse/1` | Reverse sequence | Transformation |
| `next/1` | Get next value | Core |
| `to_list/1` | Materialize to list | Materialization |
| `foldl/3` | Fold left | Materialization |
| `foldr/3` | Fold right | Materialization |
| `foreach/2` | Apply for side effects | Materialization |
| `flush/1` | Execute generator | Materialization |
| `all/2` | Test all values | Materialization |
| `any/2` | Test any value | Materialization |
| `length/1` | Count values | Materialization |

### Types

Global types used across APIs:

```erlang
generator() :: fun(() -> empty | {term(), generator()})
predicate(T) :: fun((T) -> boolean())
mapper(A, B) :: fun((A) -> B)
```

---

## Generator Creation Functions

### `empty/0`

Returns an empty generator that produces no values.

**Spec:**
```erlang
-spec empty() -> generator().
```

**Description:**
Returns a zero-arity function representing an exhausted sequence. Useful as a base case for recursive generator definitions or as a terminator.

**Returns:**
- Generator that immediately returns `empty` when called

**Example:**
```erlang
1> Gen = lazy:empty().
#Fun<lazy.0.73700886>

2> lazy:next(Gen).
empty

3> lazy:to_list(Gen).
[]
```

---

### `once/1`

Creates a generator that produces exactly one value.

**Spec:**
```erlang
-spec once(term()) -> generator().
```

**Parameters:**
- `Value` — The single value to generate

**Returns:**
- Generator that yields the value once, then `empty`

**Example:**
```erlang
1> Gen = lazy:once(hello).
#Fun<lazy.4.73700886>

2> lazy:to_list(Gen).
[hello]
```

---

### `repeat/1`

Creates an infinite generator that repeatedly produces the same value.

**Spec:**
```erlang
-spec repeat(term()) -> generator().
```

**Parameters:**
- `Value` — Value to repeat infinitely

**Returns:**
- Infinite generator producing identical values

**Example:**
```erlang
1> Gen = lazy:repeat(foo).
#Fun<lazy.5.73700886>

2> lazy:to_list(lazy:take(3, Gen)).
[foo, foo, foo]
```

---

### `seq/2`

Creates a generator for an integer range (inclusive, ascending order).

**Spec:**
```erlang
-spec seq(integer(), integer()) -> generator().
```

**Parameters:**
- `From` — Starting integer (inclusive)
- `To` — Ending integer (inclusive)

**Returns:**
- Generator yielding integers from `From` to `To`

**Guards:**
- If `From > To`, returns empty generator

**Example:**
```erlang
1> Gen = lazy:seq(1, 5).
#Fun<lazy.43.73700886>

2> lazy:to_list(Gen).
[1, 2, 3, 4, 5]
```

---

### `seq/3`

Creates a generator for an integer range with custom step.

**Spec:**
```erlang
-spec seq(integer(), integer(), integer()) -> generator().
```

**Parameters:**
- `From` — Starting integer (inclusive)
- `Step` — Increment between values (can be negative)
- `To` — Ending integer (inclusive)

**Returns:**
- Generator yielding integers with step increments

**Example:**
```erlang
1> Gen = lazy:seq(0, 2, 10).
#Fun<lazy.43.73700886>

2> lazy:to_list(Gen).
[0, 2, 4, 6, 8, 10]
```

---

### `from_list/1`

Wraps a list as a generator.

**Spec:**
```erlang
-spec from_list([term()]) -> generator().
```

**Parameters:**
- `List` — List to convert

**Returns:**
- Generator yielding list elements in order

**Example:**
```erlang
1> Gen = lazy:from_list([a, b, c]).
#Fun<lazy.1.73700886>

2> lazy:to_list(Gen).
[a, b, c]
```

---

### `iterate/2`

Creates an infinite generator by repeatedly applying a function.

**Spec:**
```erlang
-spec iterate(fun((term()) -> term()), term()) -> generator().
```

**Parameters:**
- `Function` — Function to apply repeatedly
- `Initial` — Initial value

**Returns:**
- Infinite generator: `[Initial, Function(Initial), Function(Function(Initial)), ...]`

**Example:**
```erlang
1> Gen = lazy:iterate(fun (X) -> X * 2 end, 1).
#Fun<lazy.7.9483195>

2> lazy:to_list(lazy:take(5, Gen)).
[1, 2, 4, 8, 16]
```

---

### `unfold/2`

Creates a generator from an accumulator using a function.

**Spec:**
```erlang
-spec unfold(fun((term()) -> empty | {term(), term()}), term()) -> generator().
```

**Parameters:**
- `Function` — Takes accumulator, returns `empty` or `{Value, NewAcc}`
- `Seed` — Initial accumulator value

**Returns:**
- Generator producing values from the unfold function

**Example:**
```erlang
1> Gen = lazy:unfold(fun (0) -> empty; (N) -> {N, N - 1} end, 5).
#Fun<lazy.11.73700886>

2> lazy:to_list(Gen).
[5, 4, 3, 2, 1]
```

---

### `repeatedly/1`

Creates an infinite generator by repeatedly calling a function.

**Spec:**
```erlang
-spec repeatedly(fun(() -> term())) -> generator().
```

**Parameters:**
- `Function` — Zero-arity function to call repeatedly

**Returns:**
- Infinite generator of function return values

**Example:**
```erlang
1> Counter = fun () -> erlang:monotonic_time() end.
#Fun<erl_eval.45.126501267>

2> Gen = lazy:repeatedly(Counter).
#Fun<lazy.6.73700886>

3> {T1, Gen1} = lazy:next(Gen).
{-576458245575, #Fun<lazy.48.73700886>}
```

---

### `cycle/1`

Creates an infinite generator by cycling through a list repeatedly.

**Spec:**
```erlang
-spec cycle([term()]) -> generator().
```

**Parameters:**
- `List` — List to cycle (must be non-empty)

**Returns:**
- Infinite generator repeating the list elements

**Example:**
```erlang
1> Gen = lazy:cycle([a, b]).
#Fun<lazy.60.73700886>

2> lazy:to_list(lazy:take(5, Gen)).
[a, b, a, b, a]
```

---

## Transformation Functions

### `map/2`

Transforms generator values with a function.

**Spec:**
```erlang
-spec map(fun((term()) -> term()), generator()) -> generator().
```

**Parameters:**
- `Function` — Transformation function
- `Generator` — Source generator

**Returns:**
- Generator yielding transformed values

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 3).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:map(fun (X) -> X * 10 end, Gen0).
#Fun<lazy.21.73700886>

3> lazy:to_list(Gen1).
[10, 20, 30]
```

---

### `filter/2`

Keeps only values matching a predicate.

**Spec:**
```erlang
-spec filter(fun((term()) -> boolean()), generator()) -> generator().
```

**Parameters:**
- `Predicate` — Test function
- `Generator` — Source generator

**Returns:**
- Generator yielding only matching values

**Warning:** On infinite sequences where predicate never succeeds, calling `next/1` will hang forever.

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 10).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:filter(fun (X) -> X rem 2 =:= 0 end, Gen0).
#Fun<lazy.16.73700886>

3> lazy:to_list(Gen1).
[2, 4, 6, 8, 10]
```

---

### `filtermap/2`

Combines filtering and mapping in one operation.

**Spec:**
```erlang
-spec filtermap(fun((term()) -> boolean() | {true, term()}) | {false, term()}, generator()) -> generator().
```

**Parameters:**
- `Function` — Returns `false` to discard, `true` to keep, or `{true, NewValue}` to transform
- `Generator` — Source generator

**Returns:**
- Generator with filtered and mapped values

**Example:**
```erlang
1> Gen0 = lazy:seq(0, 10).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:filtermap(fun (0) -> false; (X) when X rem 2 =:= 0 -> {true, -X}; (_) -> true end, Gen0).
#Fun<lazy.23.73700886>

3> lazy:to_list(Gen1).
[1, -2, 3, -4, 5, -6, 7, -8, 9, -10]
```

---

### `drop/2`

Skips the first N values of a generator.

**Spec:**
```erlang
-spec drop(non_neg_integer(), generator()) -> generator().
```

**Parameters:**
- `Count` — Number of values to skip
- `Generator` — Source generator

**Returns:**
- Generator starting after skipped values

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 5).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:drop(2, Gen0).
#Fun<lazy.36.73700886>

3> lazy:to_list(Gen1).
[3, 4, 5]
```

---

### `dropwhile/2`

Skips values while a predicate is true.

**Spec:**
```erlang
-spec dropwhile(fun((term()) -> boolean()), generator()) -> generator().
```

**Parameters:**
- `Predicate` — Condition to skip
- `Generator` — Source generator

**Returns:**
- Generator starting when predicate becomes false

**Warning:** On infinite sequences where predicate never fails, calling `next/1` will hang forever.

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 10).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:dropwhile(fun (X) -> X < 5 end, Gen0).
#Fun<lazy.19.73700886>

3> lazy:to_list(Gen1).
[5, 6, 7, 8, 9, 10]
```

---

### `take/2`

Limits a generator to the first N values.

**Spec:**
```erlang
-spec take(non_neg_integer(), generator()) -> generator().
```

**Parameters:**
- `Count` — Maximum number of values
- `Generator` — Source generator

**Returns:**
- Generator yielding at most N values

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 100).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:take(3, Gen0).
#Fun<lazy.24.73700886>

3> lazy:to_list(Gen1).
[1, 2, 3]
```

---

### `takewhile/2`

Takes values while a predicate is true.

**Spec:**
```erlang
-spec takewhile(fun((term()) -> boolean()), generator()) -> generator().
```

**Parameters:**
- `Predicate` — Condition to continue
- `Generator` — Source generator

**Returns:**
- Generator yielding values until predicate becomes false

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 10).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:takewhile(fun (X) -> X < 5 end, Gen0).
#Fun<lazy.25.73700886>

3> lazy:to_list(Gen1).
[1, 2, 3, 4]
```

---

### `scan/3`

Like `foldl` but returns intermediate accumulator values.

**Spec:**
```erlang
-spec scan(fun((term(), term()) -> term()), term(), generator()) -> generator().
```

**Parameters:**
- `Function` — Accumulator function
- `Initial` — Initial accumulator
- `Generator` — Source generator

**Returns:**
- Generator yielding accumulator values

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 5).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:scan(fun (X, Acc) -> Acc + X end, 0, Gen0).
#Fun<lazy.26.73700886>

3> lazy:to_list(Gen1).
[1, 3, 6, 10, 15]
```

---

### `reverse/1`

Reverses a finite generator (materializes into list internally).

**Spec:**
```erlang
-spec reverse(generator()) -> generator().
```

**Parameters:**
- `Generator` — Source generator (must be finite)

**Returns:**
- Generator yielding reversed values

**Warning:** Only use with finite sequences. Will hang/crash with infinite generators.

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 5).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:reverse(Gen0).
#Fun<lazy.9.73700886>

3> lazy:to_list(Gen1).
[5, 4, 3, 2, 1]
```

---

### `apply/2`

Applies a function to a generator, wrapping the result.

**Spec:**
```erlang
-spec apply(fun((generator()) -> generator()), generator()) -> generator().
```

**Parameters:**
- `Function` — Function transforming one generator to another
- `Generator` — Source generator

**Returns:**
- Generator with function applied

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 5).
#Fun<lazy.43.73700886>

2> F = fun (G) -> lazy:map(fun (X) -> X * 2 end, G) end.
#Fun<erl_eval.45.126501267>

3> Gen1 = lazy:apply(F, Gen0).
#Fun<lazy.8.73700886>

4> lazy:to_list(Gen1).
[2, 4, 6, 8, 10]
```

---

## Combination Functions

### `append/1`

Concatenates multiple generators in sequence.

**Spec:**
```erlang
-spec append([generator()]) -> generator().
```

**Parameters:**
- `Generators` — List of generators to concatenate

**Returns:**
- Generator yielding all values from each generator in order

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 3).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:from_list([a, b, c]).
#Fun<lazy.1.73700886>

3> Gen2 = lazy:once(end).
#Fun<lazy.4.73700886>

4> Gen3 = lazy:append([Gen0, Gen1, Gen2]).
#Fun<lazy.29.73700886>

5> lazy:to_list(Gen3).
[1, 2, 3, a, b, c, end]
```

---

### `append/2`

Appends two generators.

**Spec:**
```erlang
-spec append(generator(), generator()) -> generator().
```

**Parameters:**
- `Gen1` — First generator
- `Gen2` — Second generator

**Returns:**
- Generator yielding values from both

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 3).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:seq(10, 12).
#Fun<lazy.43.73700886>

3> Gen2 = lazy:append(Gen0, Gen1).
#Fun<lazy.30.73700886>

4> lazy:to_list(Gen2).
[1, 2, 3, 10, 11, 12]
```

---

### `zip/2`

Combines two generators into tuples.

**Spec:**
```erlang
-spec zip(generator(), generator()) -> generator().
```

**Parameters:**
- `Gen1` — First generator
- `Gen2` — Second generator

**Returns:**
- Generator yielding `{V1, V2}` tuples (stops at shortest)

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 3).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:from_list([a, b, c, d]).
#Fun<lazy.1.73700886>

3> Gen2 = lazy:zip(Gen0, Gen1).
#Fun<lazy.31.73700886>

4> lazy:to_list(Gen2).
[{1, a}, {2, b}, {3, c}]
```

---

### `zipwith/2`

Combines two generators with a function (curried).

**Spec:**
```erlang
-spec zipwith(fun((term(), term()) -> term()), generator(), generator()) -> generator().
```

**Parameters:**
- `Function` — Binary function to apply
- `Gen1` — First generator
- `Gen2` — Second generator

**Returns:**
- Generator yielding function results (stops at shortest)

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 3).
#Fun<lazy.43700886>

2> Gen1 = lazy:seq(10, 12).
#Fun<lazy.43.73700886>

3> Gen2 = lazy:zipwith(fun (A, B) -> A + B end, Gen0, Gen1).
#Fun<lazy.40.73700886>

4> lazy:to_list(Gen2).
[11, 13, 15]
```

---

### `zipwith/3`

Combines three generators with a function.

**Spec:**
```erlang
-spec zipwith(fun((term(), term(), term()) -> term()), generator(), generator(), generator()) -> generator().
```

**Parameters:**
- `Function` — Ternary function to apply
- `Gen1` — First generator
- `Gen2` — Second generator
- `Gen3` — Third generator

**Returns:**
- Generator yielding function results (stops at shortest)

---

### `unzip/1`

Unzips tuple-generating generator into multiple generators.

**Spec:**
```erlang
-spec unzip(generator()) -> {generator(), generator()} | {generator(), generator(), generator()}.
```

**Parameters:**
- `Generator` — Generator producing tuples

**Returns:**
- Tuple of generators (matched to tuple size)

**Example:**
```erlang
1> Gen0 = lazy:zip(lazy:seq(1, 3), lazy:from_list([a, b, c])).
#Fun<lazy.31.73700886>

2> {Gen1, Gen2} = lazy:unzip(Gen0).
{#Fun<lazy.42.73700886>, #Fun<lazy.42.73700886>}

3> {lazy:to_list(Gen1), lazy:to_list(Gen2)}.
{[1, 2, 3], [a, b, c]}
```

---

## Core Function

### `next/1`

Materializes and returns the next value of a generator.

**Spec:**
```erlang
-spec next(generator()) -> empty | {term(), generator()}.
```

**Parameters:**
- `Generator` — Source generator

**Returns:**
- `empty` — if generator is exhausted
- `{Value, NextGenerator}` — next value and remainder

**Example:**
```erlang
1> Gen0 = lazy:seq(1, 3).
#Fun<lazy.43.73700886>

2> {V1, Gen1} = lazy:next(Gen0).
{1, #Fun<lazy.48.73700886>}

3> {V2, Gen2} = lazy:next(Gen1).
{2, #Fun<lazy.48.73700886>}

4> {V3, Gen3} = lazy:next(Gen2).
{3, #Fun<lazy.48.73700886>}

5> lazy:next(Gen3).
empty
```

---

## Materialization Functions

### `to_list/1`

Materializes a finite generator into a list.

**Spec:**
```erlang
-spec to_list(generator()) -> [term()].
```

**Parameters:**
- `Generator` — Source generator (must be finite)

**Returns:**
- List of all values

**Warning:** Will hang/crash on infinite generators.

**Example:**
```erlang
1> Gen = lazy:seq(1, 5).
#Fun<lazy.43.73700886>

2> lazy:to_list(Gen).
[1, 2, 3, 4, 5]
```

---

### `foldl/3`

Left fold over generator values.

**Spec:**
```erlang
-spec foldl(fun((term(), term()) -> term()), term(), generator()) -> term().
```

**Parameters:**
- `Function` — Accumulator function
- `Initial` — Initial accumulator
- `Generator` — Source generator

**Returns:**
- Final accumulator value

**Example:**
```erlang
1> Gen = lazy:seq(1, 5).
#Fun<lazy.43.73700886>

2> lazy:foldl(fun (X, Acc) -> Acc + X end, 0, Gen).
15
```

---

### `foldr/3`

Right fold over generator values (materializes completely).

**Spec:**
```erlang
-spec foldr(fun((term(), term()) -> term()), term(), generator()) -> term().
```

**Parameters:**
- `Function` — Accumulator function
- `Initial` — Initial accumulator
- `Generator` — Source generator

**Returns:**
- Final accumulator value

---

### `foreach/2`

Applies a function for side effects, returns `ok`.

**Spec:**
```erlang
-spec foreach(fun((term()) -> term()), generator()) -> ok.
```

**Parameters:**
- `Function` — Side-effect function
- `Generator` — Source generator

**Returns:**
- `ok` — always

**Example:**
```erlang
1> Gen = lazy:seq(1, 3).
#Fun<lazy.43.73700886>

2> lazy:foreach(fun (X) -> io:format("~w ", [X]) end, Gen).
1 2 3 ok
```

---

### `flush/1`

Executes a generator for side effects (faster than foreach).

**Spec:**
```erlang
-spec flush(generator()) -> ok.
```

**Parameters:**
- `Generator` — Source generator

**Returns:**
- `ok` — always

---

### `all/2`

Tests whether all values match a predicate.

**Spec:**
```erlang
-spec all(fun((term()) -> boolean()), generator()) -> boolean().
```

**Parameters:**
- `Predicate` — Test function
- `Generator` — Source generator

**Returns:**
- `true` if all match, `false` if any doesn't

**Example:**
```erlang
1> Gen = lazy:seq(1, 5).
#Fun<lazy.43.73700886>

2> lazy:all(fun (X) -> X > 0 end, Gen).
true

3> lazy:all(fun (X) -> X > 3 end, Gen).
false
```

---

### `any/2`

Tests whether any value matches a predicate.

**Spec:**
```erlang
-spec any(fun((term()) -> boolean()), generator()) -> boolean().
```

**Parameters:**
- `Predicate` — Test function
- `Generator` — Source generator

**Returns:**
- `true` if any matches, `false` if none do

---

### `length/1`

Counts the number of values in a finite generator.

**Spec:**
```erlang
-spec length(generator()) -> non_neg_integer().
```

**Parameters:**
- `Generator` — Source generator (must be finite)

**Returns:**
- Count of values

**Warning:** Will hang on infinite generators.

**Example:**
```erlang
1> Gen = lazy:seq(1, 100).
#Fun<lazy.43.73700886>

2> lazy:length(Gen).
100
```

---

## See Also

- [Examples_&_Use_Cases_lazy.md](Examples_&_Use_Cases_lazy.md) — Code examples and patterns
- [Developer_Reference_lazy.md](Developer_Reference_lazy.md) — Design and architecture

---

**Lazy Authors:**
* Maria Scott ([Maria-12648430](https://github.com/Maria-12648430))
* Jan Uhlig ([juhlig](https://github.com/juhlig))

**Documentation Author:** goatrllr ([https://github.com/goatrllr](https://github.com/goatrllr))
