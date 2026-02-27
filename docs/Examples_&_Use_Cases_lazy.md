# Examples & Use Cases

Practical examples and patterns for using the lazy library.

---

## Key Patterns

### Creating Custom Generators

The simplest generator is a zero-arity function returning `{Value, NextGenerator}` or `empty`.

#### Fibonacci Sequence

Generate Fibonacci numbers on-demand:

```erlang
fib(A, B) ->
    fun () ->
        {A, fib(B, A + B)}
    end.

% Usage
Gen = lazy:take(10, fib(0, 1)),
lazy:to_list(Gen).
% [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

Note: This generator is infinite — it will never return `empty`.

#### Countdown Sequence

Generate descending integers:

```erlang
countdown(0) ->
    fun () -> empty end;
countdown(N) ->
    fun () ->
        {N, countdown(N - 1)}
    end.

% Usage
Gen = countdown(5),
lazy:to_list(Gen).
% [5, 4, 3, 2, 1]
```

This generator is finite — it terminates at 0.

#### Collatz Sequence

Generate Collatz sequence (3n+1 problem) for a starting number:

```erlang
collatz(1) ->
    fun () -> {1, fun () -> empty end} end;
collatz(N) when N rem 2 =:= 0 ->
    fun () ->
        {N, collatz(N div 2)}
    end;
collatz(N) ->
    fun () ->
        {N, collatz(3 * N + 1)}
    end.

% Usage: Collatz sequence starting from 27
Gen = collatz(27),
Length = lazy:length(Gen),
% Length = 112 (sequence length)

MaxValue = lazy:foldl(
    fun (X, Max) -> max(X, Max) end,
    0,
    collatz(27)
),
% MaxValue = 9232
```

#### Factorial Sequence

Generate factorials from 0!:

```erlang
fact_seq() ->
    FactGen = fun Fact(N, Acc) ->
        fun () -> {Acc, Fact(N + 1, Acc * (N + 1))} end
    end,
    FactGen(1, 1).

% Usage
Gen = lazy:take(10, fact_seq()),
lazy:to_list(Gen).
% [1, 2, 6, 24, 120, 720, 5040, 40320, 362880, 3628800]
```

### Filtering Patterns

#### Safe Filtering with Limits

**Warning:** Using `filter/2` on infinite sequences without `take/2` can hang.

```erlang
% Good: limit first
Gen0 = lazy:take(100, lazy:seq(1, infinity)),
Gen = lazy:filter(fun (X) -> X rem 7 =:= 0 end, Gen0),
lazy:to_list(Gen).
% [7, 14, 21, 28, 35, 42, 49, 56, 63, 70, 77, 84, 91, 98]

% Dangerous: unlimited filter on infinite sequence
Gen = lazy:filter(
    fun (X) -> X rem 2 =:= 0 end,
    lazy:repeat(something_odd)  % Never matches — hangs!
),
lazy:next(Gen).  % THIS WILL HANG
```

#### Practical Filtering Example

Find prime numbers less than 100:

```erlang
is_prime(2) -> true;
is_prime(N) when N < 2 orelse N rem 2 =:= 0 -> false;
is_prime(N) ->
    not(
        lazy:any(
            fun (D) -> N rem D =:= 0 end,
            lazy:seq(3, trunc(math:sqrt(N)), 2)
        )
    ).

Primes = lazy:filter(fun is_prime/1, lazy:seq(2, 100)),
lazy:to_list(Primes).
% [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97]
```

### Mapping and Transformation

#### Batch Processing with Map

Transform multiple files in a pipeline:

```erlang
% Simulate file contents
read_file(Path) -> {ok, "content of " ++ Path}.

process_content(Content) -> string:to_upper(Content).

Files = ["file1.txt", "file2.txt", "file3.txt"],
Gen0 = lazy:from_list(Files),
Gen1 = lazy:map(fun read_file/1, Gen0),
Gen2 = lazy:map(fun ({ok, Content}) -> process_content(Content) end, Gen1),
Results = lazy:to_list(Gen2),

lists:foreach(fun (R) -> io:format("~s~n", [R]) end, Results).
% CONTENT OF FILE1.TXT
% CONTENT OF FILE2.TXT
% CONTENT OF FILE3.TXT
```

#### Chaining Transformations

Multiple operations compose without intermediate materialization:

```erlang
Gen0 = lazy:seq(1, 100),
Gen1 = lazy:map(fun (X) -> X * 2 end, Gen0),           % Double each
Gen2 = lazy:filter(fun (X) -> X > 50 end, Gen1),       % Keep > 50
Gen3 = lazy:map(fun (X) -> X + 100 end, Gen2),         % Add 100
Gen4 = lazy:take(10, Gen3),                             % Limit to 10
Result = lazy:to_list(Gen4),

Result.
% [152, 154, 156, 158, 160, 162, 164, 166, 168, 170]
```

### Combining Generators

#### Zipping Sequences

Combine two sequences element-wise:

```erlang
Names = lazy:from_list([alice, bob, charlie]),
Ages = lazy:from_list([25, 30, 35]),

People = lazy:zip(Names, Ages),
lazy:to_list(People).
% [{alice, 25}, {bob, 30}, {charlie, 35}]
```

#### Zipping with Function

Combine with a custom operation:

```erlang
Distances = lazy:from_list([100, 200, 150]),
Times = lazy:from_list([5, 10, 15]),

Speeds = lazy:zipwith(
    fun (Dist, Time) -> Dist / Time end,
    Distances,
    Times
),
lazy:to_list(Speeds).
% [20.0, 20.0, 10.0]
```

#### Appending Sequences

Concatenate multiple generators:

```erlang
Gen1 = lazy:seq(1, 3),
Gen2 = lazy:from_list([a, b, c]),
Gen3 = lazy:cycle([x, y]),

Combined = lazy:append([Gen1, Gen2, lazy:take(3, Gen3)]),
lazy:to_list(Combined).
% [1, 2, 3, a, b, c, x, y, x]
```

### Folding and Aggregation

#### Summing Values

Left fold for accumulation:

```erlang
Numbers = lazy:seq(1, 100),
Sum = lazy:foldl(
    fun (X, Acc) -> Acc + X end,
    0,
    Numbers
),
Sum.
% 5050

% Or with scan to see intermediate sums
Gen0 = lazy:seq(1, 10),
Gen1 = lazy:scan(fun (X, Acc) -> Acc + X end, 0, Gen0),
IntermediateSums = lazy:to_list(Gen1),

IntermediateSums.
% [1, 3, 6, 10, 15, 21, 28, 36, 45, 55]
```

#### All/Any Predicates

Test multiple values:

```erlang
% Check if all are positive
Numbers = lazy:from_list([1, 2, 3, 4, 5]),
lazy:all(fun (X) -> X > 0 end, Numbers).
% true

% Check if any are even
AllOdd = lazy:from_list([1, 3, 5, 7]),
lazy:any(fun (X) -> X rem 2 =:= 0 end, AllOdd).
% false
```

---

## Use Cases

### 1. Large File Processing

Process multi-gigabyte files line-by-line without loading into memory:

```erlang
% Conceptual: read file as line generator
read_lines(Path) ->
    {ok, File} = file:open(Path, [read, binary]),
    LineGen = fun ReadNext() ->
        case file:read_line(File) of
            {ok, Line} -> {Line, ReadNext()};
            eof -> empty
        end
    end,
    LineGen.

% Usage: Process log file
LogFile = read_lines("/var/log/app.log"),

Gen0 = lazy:filter(fun (Line) -> string:find(Line, "ERROR") =/= nomatch end, LogFile),
Gen1 = lazy:map(fun string:strip/1, Gen0),
ErrorLines = lazy:take(100, Gen1),  % First 100 errors

% Print without materializing entire file
lazy:foreach(fun (Line) -> io:format("~s~n", [Line]) end, ErrorLines).
```

### 2. Infinite Mathematical Sequences

Generate mathematical sequences on-demand:

```erlang
% Natural numbers
naturals() -> lazy:iterate(fun (N) -> N + 1 end, 1).

% Powers of 2: 1, 2, 4, 8, 16, ...
powers_of_2() -> lazy:iterate(fun (N) -> N * 2 end, 1).

% All integers starting from N
integers_from(N) -> lazy:iterate(fun (X) -> X + 1 end, N).

% Usage: First 10 powers of 2
Gen1 = powers_of_2(),
Gen2 = lazy:take(10, Gen1),
lazy:to_list(Gen2).
% [1, 2, 4, 8, 16, 32, 64, 128, 256, 512]

% Even numbers up to 100
Gen3 = lazy:seq(1, 100),
Gen4 = lazy:filter(fun (X) -> X rem 2 =:= 0 end, Gen3),
lazy:to_list(Gen4).
% [2, 4, 6, 8, ..., 98, 100]
```

### 3. Data Transformation Pipelines

Build ETL pipelines with lazy composition:

```erlang
% Simulated data source
fetch_records() ->
    lazy:from_list([
        #{id => 1, name => "Alice", age => 25, active => true},
        #{id => 2, name => "Bob", age => 30, active => false},
        #{id => 3, name => "Charlie", age => 35, active => true},
        #{id => 4, name => "Diana", age => 28, active => true},
        #{id => 5, name => "Eve", age => 32, active => false}
    ]).

% Process records through pipeline
Gen0 = fetch_records(),
Gen1 = lazy:filter(fun (#{active := A}) -> A end, Gen0),  % Only active
Gen2 = lazy:map(fun (#{name := N, age := Age}) ->
        {N, Age}
    end, Gen1),
Gen3 = lazy:filter(fun ({_N, Age}) -> Age >= 25 end, Gen2),  % Minimum age
Results = lazy:to_list(Gen3),

Results.
% [{alice, 25}, {charlie, 35}, {diana, 28}]
```

### 4. Stream Processing from External Sources

Process real-time data streams:

```erlang
% Simulated: receive messages from a queue
receive_messages() ->
    fun Receive() ->
        case queue:get(message) of  % Blocking get from queue
            {ok, Message} -> {Message, Receive()};
            timeout -> empty
        end
    end.

% Usage: Process messages as they arrive
Gen0 = receive_messages(),
Gen1 = lazy:take(1000, Gen0),  % Process up to 1000 messages
Gen2 = lazy:filter(fun (#{type := important}) -> true; (_) -> false end, Gen1),
MessageStream = lazy:map(fun (Msg) -> process_important(Msg) end, Gen2),

% Process side-by-side (without materializing)
lazy:foreach(fun log_message/1, MessageStream).
```

### 5. Generating Test Data

Create test datasets on-demand:

```erlang
% Infinite user generator
user_generator(Id) ->
    fun () ->
        User = #{
            id => Id,
            name => "User" ++ integer_to_list(Id),
            email => "user" ++ integer_to_list(Id) ++ "@example.com"
        },
        {User, user_generator(Id + 1)}
    end.

% Generate 100 test users
TestUsers = lazy:take(100, user_generator(1)),
lazy:to_list(TestUsers).

% Or use with EUnit
test_user_count() ->
    ?assertEqual(
        100,
        lazy:length(lazy:take(100, user_generator(1)))
    ).
```

### 6. Nested Structure Traversal

Walk tree structures lazily:

```erlang
% Tree generator: depth-first traversal
-record(tree, {value, left, right}).

tree_values(undefined) ->
    fun () -> empty end;
tree_values(#tree{value = V, left = L, right = R}) ->
    fun () ->
        {V, lazy:append([
            tree_values(L),
            tree_values(R)
        ])}
    end.

% Usage
Tree = #tree{
    value = 1,
    left = #tree{value = 2, left = undefined, right = undefined},
    right = #tree{value = 3, left = undefined, right = undefined}
},

lazy:to_list(tree_values(Tree)).
% [1, 2, 3]
```

### 7. Windowing/Sliding Windows

Process data with moving windows:

```erlang
% Create sliding window of size N
window(N, Gen) ->
    WindowList = lazy:take(N, Gen),
    RestGen = lazy:drop(N, Gen),
    
    WindowGen = fun Window(WL, RG) ->
        fun () ->
            case lazy:to_list(lazy:take(N, WL)) of
                [] -> empty;
                Window -> {Window, Window(lazy:drop(1, WL), RG)}
            end
        end
    end,
    WindowGen(WindowList, RestGen).

% Usage: sliding window of 3 on [1..10]
Data = lazy:seq(1, 10),
Windows = window(3, Data),
lazy:to_list(Windows).
% [[1, 2, 3], [2, 3, 4], [3, 4, 5], ..., [8, 9, 10]]
```

---

## Advanced Examples

### Parallel Map-Reduce Pattern

Use lazy generators with parallel processing:

```erlang
% Map phase: process items lazily
map_phase(Items) ->
    Gen = lazy:from_list(Items),
    lazy:map(fun expensive_transform/1, Gen).

% Reduce phase: collect results
reduce_phase(Gen) ->
    lazy:foldl(
        fun (Item, Acc) -> merge_results(Item, Acc) end,
        #{},
        Gen
    ).

% Full pipeline
run_mapreduce(Data) ->
    reduce_phase(map_phase(Data)).
```

### Lazy JSON Parsing

Parse large JSON arrays without materializing:

```erlang
% Generator from JSON array token stream
json_array_generator(Stream) ->
    fun Parse() ->
        case read_next_json_object(Stream) of
            {ok, Object} -> {Object, Parse()};
            eof -> empty
        end
    end.

% Usage: Process million-item JSON without loading
parse_large_json(Path) ->
    {ok, Stream} = open_json_stream(Path),
    Gen0 = json_array_generator(Stream),
    Gen1 = lazy:filter(fun (O) -> maps:get(active, O) end, Gen0),
    Gen2 = lazy:map(fun extract_fields/1, Gen1),
    Result = lazy:take(1000, Gen2),  % First 1000 active items
    
    % Process as you iterate
    lazy:foreach(fun process_item/1, Result).
```

### Memoized Fibonacci

Combine laziness with memoization for performance:

```erlang
% Lazy Fibonacci with memoization using ets
fib_memo() ->
    Table = ets:new(fib_cache, []),
    FibGen = fun Fib(N) ->
        fun () ->
            case ets:lookup(Table, N) of
                [{_, Val}] -> {Val, Fib(N + 1)};
                [] ->
                    NextN = N + 1,
                    FibVal = fib_nth(N),
                    ets:insert(Table, {N, FibVal}),
                    {FibVal, Fib(NextN)}
            end
        end
    end,
    FibGen(0).

% Approximately n-th Fibonacci (with memoization speedup)
fib_nth(0) -> 0;
fib_nth(1) -> 1;
fib_nth(N) ->
    {A, GenA} = lazy:next(lazy:take(N, fib_memo())),
    {B, GenB} = lazy:next(lazy:drop(1, GenA)),
    A + element(1, lazy:next(GenB)).
```

### Interactive REPL-style Generator

Create generators for interactive commands:

```erlang
% User-input generator
input_generator() ->
    fun Input() ->
        case io:get_line("> ") of
            eof ->
                empty;
            Line ->
                Command = string:strip(binary_to_list(Line)),
                {Command, Input()}
        end
    end.

% Usage: Process user commands
Commands = input_generator(),
Gen0 = lazy:filter(fun (C) -> C =/= "" end, Commands),  % Skip empty
Gen1 = lazy:takewhile(fun (C) -> C =/= "quit" end, Gen0),  % Stop on quit
ProcessedCommands = lazy:map(fun parse_command/1, Gen1),

lazy:foreach(fun execute_command/1, ProcessedCommands).
```

---

## Performance Comparison

### Memory Usage Example

Process a sequence of 1,000,000 items:

```erlang
% List-based (materializes all): High memory
Numbers = lists:seq(1, 1000000),
Result = lists:map(fun double/1, Numbers),
EvenNumbers = lists:filter(fun is_even/1, Result),
lists:foreach(fun print/1, lists:take(10, EvenNumbers)).
% Memory: ~40MB for intermediate list

% Lazy (constant memory): Low memory
Gen0 = lazy:seq(1, 1000000),
Gen1 = lazy:map(fun double/1, Gen0),
Gen2 = lazy:filter(fun is_even/1, Gen1),
Gen = lazy:take(10, Gen2),
lazy:foreach(fun print/1, Gen).
% Memory: <1MB regardless of sequence size
```

### Composition Benefit

Avoiding intermediate materialization:

```erlang
% Three separate list operations (2 materializations)
Step1 = lists:map(Model1, Data),           % Materialize: 1M items
Step2 = lists:map(Model2, Step1),          % Materialize: 1M items
Step3 = lists:filter(Pred, Step2),         % Materialize: ~500k items

% Single lazy pipeline (0 materializations)
Gen0 = lazy:from_list(Data),
Gen1 = lazy:map(Model1, Gen0),
Gen2 = lazy:map(Model2, Gen1),
Gen3 = lazy:filter(Pred, Gen2),
Result = lazy:to_list(Gen3).  % Materialize once: ~500k items
```

---

## See Also

- [API_Documentation_lazy.md](API_Documentation_lazy.md) — Function reference
- [Developer_Reference_lazy.md](Developer_Reference_lazy.md) — Design and architecture
- [README.md](../README.md) — Quick start guide

---

**Lazy Authors:**
* Maria Scott ([Maria-12648430](https://github.com/Maria-12648430))
* Jan Uhlig ([juhlig](https://github.com/juhlig))

**Documentation Author:** goatrllr ([https://github.com/goatrllr](https://github.com/goatrllr))
