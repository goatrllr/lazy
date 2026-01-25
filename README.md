# Lazy

Lazy sequences for Erlang.

Erlang is an eager language. It materializes terms, including sequences
such as lists, completely into memory at the point that you write them in
the code.

Other functional languages such as Gleam, Clojure, or Haskell and
even some non-functional languages such as Python know lazy sequences
which generate items only when you access them.

`lazy` provides a mechanism to use lazy sequences with Erlang.

Be aware that lazy sequences are a two-edged sword, however. While they
are more memory-friendly than eager sequences and _may_ yield more
performance given circumstances, they are also less predictable and
harder to reason about, especially if your generators rely on side
effects.
Used naively, large amounts of data may explode into memory if you use
them in a way that requires that a sequence must be materialized, and
you may also find yourself in an endless loop if you unwittingly run
down an infinite sequence, or both.

## Generators

Generators are the key components that make lazy sequences possible.
Rather than concrete sequences consisting of concrete values, they
are a recipe to generate values on the fly.

A generator is a function producing the values making up a sequence,
one at a time.
Generators may produce bounded (finite) or unbounded (infinite) sequences.

There are no guarantees regarding to when and how often they will be called.

The next value of a generator can be generated with a call to
`next/1`, which will either return the atom `empty` indicating
that the sequence is exhausted, or the current value and a new
generator to access the next value in a tuple.

### Built-in generators

`lazy` comes with a collection of functions to create generators
for common use cases.

* `append/1` and `append/2`
* `apply/2`
* `cycle/1`
* `drop/2`
* `dropwhile/2`
* `empty/0`
* `filter/2`
* `filtermap/2`
* `from_list/1`
* `iterate/2`
* `map/2`
* `once/1`
* `repeat/1`
* `repeatedly/1`
* `reverse/1`
* `scan/3`
* `seq/2` and `seq/3`
* `take/2`
* `takewhile/2`
* `unfold/2`
* `unzip/1`
* `zip/2`
* `zipwith/2` and `zipwith/3`

### Core Functions

* `next/1` - Materializes and returns the next value of a generator

Some of the listed functions, like `reverse/1`, should only be used with generators
that produce finite sequences.

#### Generator Examples

**`unfold/2`** - Generates values from an accumulator using a function:

```erlang
1> Gen = lazy:unfold(fun (0) -> empty; (V) -> {V, V div 2} end, 256).
#Fun<lazy.11.73700886>

2> lazy:to_list(Gen).
[256, 128, 64, 32, 16, 8, 4, 2, 1]
```

**`iterate/2`** - Generates values by repeatedly applying a function:

```erlang
1> Gen0 = lazy:iterate(fun (V) -> 3 * V end, 1).
#Fun<lazy.7.9483195>

2> lazy:to_list(lazy:take(4, Gen0)).
[1, 3, 9, 27]
```

**`filtermap/2`** - Combines filtering and mapping:

```erlang
1> Gen0 = lazy:seq(0, 10).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:filtermap(fun (0) -> false; (V) when V rem 2 =:= 0 -> {true, -V}; (_) -> true end, Gen0).
#Fun<lazy.23.73700886>

3> lazy:to_list(Gen1).
[1, -2, 3, -4, 5, -6, 7, -8, 9, -10]
```

**`append/1`** - Concatenates multiple generators:

```erlang
1> Gen0 = lazy:seq(1, 3).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:from_list([a, b, c]).
#Fun<lazy.1.73700886>

3> Gen2 = lazy:once("foo").
#Fun<lazy.4.73700886>

4> Gen3 = lazy:append([Gen0, Gen1, Gen2]).
#Fun<lazy.29.73700886>

5> lazy:to_list(Gen3).
[1, 2, 3, a, b, c, "foo"]
```

**`repeatedly/1`** - Generates values by repeated function calls:

```erlang
1> Gen0 = lazy:repeatedly(fun () -> erlang:monotonic_time(millisecond) end).
#Fun<lazy.6.73700886>

2> {_, Gen1} = lazy:next(Gen0).
{-576458245575, #Fun<lazy.48.73700886>}

3> {_, Gen2} = lazy:next(Gen1).
{-576458239259, #Fun<lazy.48.73700886>}
```

**`scan/3`** - Similar to `foldl` but produces intermediate accumulator values:

```erlang
1> Gen0 = lazy:seq(1, 5).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:scan(fun (V, Acc) -> [V * V|Acc] end, [], Gen0).
#Fun<lazy.26.73700886>

3> lazy:to_list(Gen1).
[[1], [4, 1], [9, 4, 1], [16, 9, 4, 1], [25, 16, 9, 4, 1]]
```

**`zipwith/2` and `zipwith/3`** - Combine generators with a function:

```erlang
1> Gen0 = lazy:seq(1, 3).
#Fun<lazy.43.73700886>

2> Gen1 = lazy:seq(4, 6).
#Fun<lazy.43.73700886>

3> Gen2 = lazy:zipwith(fun (V1, V2) -> V1 + V2 end, Gen0, Gen1).
#Fun<lazy.40.73700886>

4> lazy:to_list(Gen2).
[5, 7, 9]
```

**`repeat/1`** - Generates infinite repetitions of a value:

```erlang
1> Gen0 = lazy:repeat(foo).
#Fun<lazy.5.73700886>

2> lazy:to_list(lazy:take(3, Gen0)).
[foo, foo, foo]
```

### Custom generators

To create a custom generator, you must devise a function of arity `0` which, when called,
returns either the atom `empty` to indicate that the generator is exhausted, or a 2-tuple
consisting of the generated value and a new function of the same design.

The following example generator produces the Fibonacci numbers.

```erlang
fib() ->
    fun () -> fib1(0, 1) end.

fib1(N1, N2) ->
    {N1 + N2, fun () -> fib1(N2, N1 + N2) end}.
```

The following example generator produces the Collatz sequence for a given number.

```erlang
collatz(N) when is_integer(N), N > 0 ->
    fun () -> collatz1(N) end.

collatz1(1) ->
    lazy:once(1);
collatz1(N) when N rem 2 =:= 0 ->
    {N, fun () -> collatz1(N div 2) end};
collatz1(N) ->
    {N, fun () -> collatz1(3 * N + 1) end}.
```

The following example generator produces the factorials, starting from 1.

```erlang
fact() ->
	fun () -> fact1(1, 1) end.

fact1(N, Fact) ->
    {Fact, fun () -> fact1(N + 1, (N + 1) * Fact) end}.
```

## Materializing

`lazy` comes with a collection of functions to materialize generators into concrete
terms. Such functions should only be used with generators that produce finite
sequences.

* `to_list/1`
* `foldl/3` and `foldr/3`
* `flush/1`
* `foreach/2`
* `all/2` and `any/2`
* `length/1`

#### Materializing Examples

**`foreach/2`** - Applies a function for side effects, discarding results:

```erlang
1> Gen = lazy:seq(1, 3).
#Fun<lazy.43.73700886>

2> lazy:foreach(fun (V) -> io:format("Item: ~w~n", [V]) end, Gen).
Item: 1
Item: 2
Item: 3
ok
```

## Warnings

Special care must be taken with generators that do a fast-forward with a predicate
(like `filter`, `filtermap` or `dropwhile`) when used on an infinite sequence.

With `filter` and `filtermap`, if the predicate never succeeds, a call to `next`
(implicit or explicit) will hang forever.

```erlang
1> Gen = `lazy:filter(fun (V) -> is_atom(V) end, lazy:seq(1, infinity)).
#Fun<lazy.16.33120069>
2> lazy:next(Gen).
... hangs
```

The same is true for `dropwhile` if the predicate never fails.

```erlang
1> Gen = lazy:dropwhile(fun (V) -> is_integer(V) end, lazy:seq(1, infinity)).
#Fun<lazy.13.33120069>
2> lazy:next(Gen).
... hangs
```

## Authors

* Maria Scott (Maria-12648430)
* Jan Uhlig (juhlig)
