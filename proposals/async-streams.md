# Async Streams

* [x] Proposed
* [ ] Prototype
* [ ] Implementation
* [ ] Specification

## Summary
[summary]: #summary

C# has support for iterator methods and async methods, but no support for a method that is both an iterator and an async method.  We should rectify this by allowing for await to be used in a new form of iterator, one that returns an `IAsyncEnumerable<T>` or `IAsyncEnumerator<T>` rather than an `IEnumerable<T>` or `IEnumerator<T>`.  An optional `IAsyncDisposable` interface is also used to enable for asynchronous cleanup.

## Related discussion
- https://github.com/dotnet/roslyn/issues/261
- https://github.com/dotnet/roslyn/issues/114

## Detailed design
[design]: #detailed-design

## Interfaces

### IAsyncDisposable

There has been much discussion of `IAsyncDisposable` (e.g. https://github.com/dotnet/roslyn/issues/114) and whether it's a good idea.  However, it's a required concept to add in support of async iterators.  Since `finally` blocks may contain `await`s, and since `finally` blocks need to be run as part of disposing of iterators, we need async disposal.  It's also just generally useful any time cleaning up of resources might take any period of time, e.g. closing files (requiring flushes), deregistering callbacks and providing a way to know when deregistration has completed, etc.

The following interface is added to the core .NET libraries (e.g. mscorlib, System.Runtime):
```C#
namespace System
{
    public interface IAsyncDisposable
    {
        Task DisposeAsync();
    }
}
```
Types may implement both `IDisposable` and `IAsyncDisposable`.  If they do, consumers of an instance should not invoke both `DisposeAsync` and `Dispose`; rather, if a type implements `IAsyncDisposable`, `DisposeAsync` should be used, else if it implements `IDisposable`, `Dispose` should be used.

I'm leaving discussion of how `IAsyncDisposable` interacts with `using` to a separate discussion.  And coverage of how it interacts with `foreach` is handled later in this proposal.

Alternatives considered:
- _`DisposeAsync` accepting a `CancellationToken`_: while in theory it makes sense that anything async can be canceled, disposal is about cleanup, closing things out, free'ing resources, etc., which is generally not something that should be canceled; cleanup is still important for work that's canceled.  The same `CancellationToken` that caused the actual work to be canceled would typically be the same token passed to `DisposeAsync`, making `DisposeAsync` worthless because cancellation of the work would cause `DisposeAsync` to be a nop.  If someone wants to avoid being blocked waiting for disposal, they can avoid waiting on the resulting `Task`, or wait on it only for some period of time.
- _`DisposeAsync` returning a value task_: There's no non-generic `ValueTask<T>` because there's no need.  If `DisposeAsync` completes synchronously, it can return `Task.CompletedTask`.
- _Configuring `DisposeAsync` with a `bool continueOnCapturedContext` (`ConfigureAwait`)_: While there may be issues related to how such a concept is exposed to `using`, `foreach`, and other language constructs that consume this, from an interface perspective it's not actually doing any `await`'ing and there's nothing to configure... consumers of the `Task` can consume it however they wish.
- _`IAsyncDisposable` inheriting `IDisposable`_:  Since only one or the other should be used, it doesn't make sense to force types to implement both.
- _`IDisposableAsync` instead of `IAsyncDisposable`_: We've been following the naming that things/types are an "async something" whereas operations are "done async", so types have "Async" as a prefix and methods have "Async" as a suffix.

### IAsyncEnumerable / IAsyncEnumerator

Two interfaces are added to the core .NET libraries:
```C#
namespace System.Collections.Generic
{
    public interface IAsyncEnumerable<out T>
    {
        IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken cancellationToken = default);
    }

    public interface IAsyncEnumerator<out T> : IAsyncDisposable
    {
        Task<bool> MoveNextAsync();
        T Current { get; }
    }
}
```
Typical consumption (without additional language features) would look like:
```C#
IAsyncEnumerator<T> enumerator = enumerable.GetAsyncEnumerator();
try
{
    while (await enumerator.MoveNextAsync())
    {
        Use(enumerator.Current);
    }
}
finally { await enumerator.DisposeAsync(); }
```

Discarded options considered:
- _`ValueTask<bool> MoveNextAsync(); T current { get; }`_: There's no benefit to using `ValueTask<bool>` instead of `Task<bool>`, as all possible `Task<bool>` values for synchronous completion can be cached, for asynchronous completion a `Task<bool>` is allocated anyway to back the `ValueTask<bool>`, and `ValueTask<bool>` adds additional overheads not present with `Task<bool>` (e.g. it has both a `T` and `Task<T>` field, which means its builder does, too, which means it increases the size of state machine types that await it).
- _`ValueTask<(bool, T)> MoveNextAsync();`_: It's not only harder to consume, but it means that `T` can no longer be covariant.
- _`ValueTask<T?> TryMoveNextAsync();`_: Not covariant.
- _`Task<T?> TryMoveNextAsync();`_: Not covariant, allocations on every call, etc.
- _`ITask<T?> TryMoveNextAsync();`_: Not covariant, allocations on every call, etc.
- _`ITask<(bool,T)> TryMoveNextAsync();`_: Not covariant, allocations on every call, etc.
- _`Task<bool> TryMoveNextAsync(out T result);`_: The `out` result would need to be set when the operation returns synchronously, not when it asynchronously completes the task potentially sometime long in the future, at which point there'd be no way to communicate the result.
- _`IAsyncEnumerator<T>` not implementing `IAsyncDisposable`_: We could choose to separate these.  However, doing so complicates certain other areas of the proposal, as code must then be able to deal with the possibility that an enumerator doesn't provide disposal, which makes it difficult to write pattern-based helpers (see `ConfigureAwait` discussion).  Further, it will be common for enumerators to have a need for disposal (e.g. any C# async iterator that has a finally block, most things enumerating data from a network connection, etc.), and if one doesn't, it is simple to implement the method purely as `public Task DisposeAsync() => Task.CompletedTask;` with minimal additional overhead.

#### Viable alternative:
```C#
namespace System.Collections.Generic
{
    public interface IAsyncEnumerable<out T>
    {
        IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken cancellationToken = default);
    }

    public interface IAsyncEnumerator<out T>
    {
        Task<bool> WaitForNextAsync();
        T TryGetNext(out bool success);
    }
}
```
TryGetNext is used in an inner loop to consume items with a single interface call as long as they're available synchronously.  When the next item can't be retrieved synchronously, it returns false, and any time it returns false, a caller must subsequently invoke WaitForNextAsync to either wait for the next item to be available or to determine that there will never be another item. Typical consumption (without additional language features) would look like:
```C#
IAsyncEnumerable<T> enumerator = enumerable.GetAsyncEnumerator();
while (await enumerator.WaitForNextAsync())
{
    while (true)
    {
        int item = e2.TryGetNext(out bool success);
        if (!success) break;
        Use(item);
    }
}
```
Consumption of this interface is obviously more complex. However, the advantage of this is two-fold, one minor and one major:
- _Allows for an enumerator to support multiple consumers_. There may be scenarios where it's valuable for an enumerator to support multiple concurrent consumers.  That can't be achieved when `MoveNextAsync` and `Current` are separate such that an implementation can't make their usage atomic.  In contrast, this approach provides a single method `TryGetNext` that supports pushing the enumerator forward and getting the next item, so the enumerator can enable atomicity if desired.  However, it's likely that such scenarios could also be enabled by giving each consumer its own enumerator from a shared enumerable.  Further, we don't want to enforce that every enumerator support concurrent usage, as that would add non-trivial overheads to the majority case that doesn't require it, which means a consumer of the interface generally couldn't rely on this any way.
- _Performance_. The `MoveNextAsync`/`Current` approach requires two interface calls per operation, whereas the best case for `WaitForNextAsync`/`TryGetNext` is that most iterations complete synchronously, enabling a tight inner loop with `TryGetNext`, such that we only have one interface call per operation.  This can have a measurable impact.

From a performance perspective, here's a set of BenchmarkDotNet microbenchmarks just to highlight the best case impact (typical iterators probably wouldn't benefit as much).  The enumerator in use here simply returns integers counting up from 0 to some max value, and may asynchronously yield (i.e. not complete synchronously when getting an item) every N items.
```C#
using BenchmarkDotNet.Running;
using BenchmarkDotNet.Attributes;
using System;
using System.Collections.Generic;
using System.Runtime.CompilerServices;
using System.Threading;
using System.Threading.Tasks;
using System.Collections;

public class Program
{
    public static void Main() => BenchmarkRunner.Run<Program>();

    private CountingAsyncEnumerable _enumerable;

    [Params(10_000_000)] public uint Items { get; set; }
    [Params(4_294_967_295, 100)] public uint YieldEvery { get; set; }
    [GlobalSetup] public void Init() => _enumerable = new CountingAsyncEnumerable(Items, YieldEvery);

    [Benchmark]
    public void MoveNext_Current()
    {
        _enumerable.Reset();
        var e1 = ((IEnumerable<int>)_enumerable).GetEnumerator();
        while (e1.MoveNext())
        {
            Use(e1.Current);
        }
    }

    [Benchmark]
    public async Task MoveNextAsync_Current()
    {
        _enumerable.Reset();
        var e1 = ((IAsyncEnumerable1<int>)_enumerable).GetAsyncEnumerator();
        while (await e1.MoveNextAsync())
        {
            Use(e1.Current);
        }
    }

    [Benchmark]
    public async Task WaitForNextAsync_TryGetNext()
    {
        _enumerable.Reset();
        var e2 = ((IAsyncEnumerable2<int>)_enumerable).GetAsyncEnumerator();
        while (await e2.WaitForNextAsync())
        {
            while (true)
            {
                int item = e2.TryGetNext(out bool success);
                if (!success) break;
                Use(item);
            }
        }
    }

    [MethodImpl(MethodImplOptions.NoInlining)]
    private static void Use<T>(T item) { }

    private sealed class CountingAsyncEnumerable : IAsyncEnumerable1<int>, IAsyncEnumerable2<int>, IAsyncEnumerator1<int>, IAsyncEnumerator2<int>, IEnumerable<int>, IEnumerator<int>
    {
        private static readonly Task<bool> s_falseTask = Task.FromResult(false);
        private static readonly Task<bool> s_trueTask = Task.FromResult(true);
        private readonly uint _items;
        private readonly uint _yieldEvery;

        private uint _remainingUntilYield;
        private int _current;

        public CountingAsyncEnumerable(uint items, uint yieldEvery)
        {
            if (items < 0) throw new ArgumentOutOfRangeException();
            if (yieldEvery < 1) throw new ArgumentOutOfRangeException();
            _items = items;
            _yieldEvery = yieldEvery;
            Reset();
        }

        public void Reset()
        {
            _remainingUntilYield = 0;
            _current = -1;
        }

        // IAsyncEnumerable1<T>
        IAsyncEnumerator1<int> IAsyncEnumerable1<int>.GetAsyncEnumerator(CancellationToken cancellationToken) => this;
        Task<bool> IAsyncEnumerator1<int>.MoveNextAsync()
        {
            if (_current >= _items - 1) return s_falseTask;
            _current++;

            if (_remainingUntilYield == 0)
            {
                _remainingUntilYield = _yieldEvery;
                return Task.Run(() => true);
            }
            else
            {
                _remainingUntilYield--;
                return s_trueTask;
            }
        }
        int IAsyncEnumerator1<int>.Current => _current;

        // IAsyncEnumerable2<T>
        IAsyncEnumerator2<int> IAsyncEnumerable2<int>.GetAsyncEnumerator(CancellationToken cancellationToken) => this;
        Task<bool> IAsyncEnumerator2<int>.WaitForNextAsync()
        {
            if (_current >= _items - 1) return s_falseTask;

            if (_remainingUntilYield == 0)
            {
                _remainingUntilYield = _yieldEvery;
                return Task.Run(() => true);
            }
            else
            {
                return s_trueTask;
            }
        }
        int IAsyncEnumerator2<int>.TryGetNext(out bool success)
        {
            if (_current >= _items - 1 || _remainingUntilYield == 0)
            {
                success = false;
                return 0;
            }

            success = true;
            _remainingUntilYield--;
            return ++_current;
        }

        // IEnumerable<T>
        IEnumerator<int> IEnumerable<int>.GetEnumerator() => this;
        bool IEnumerator.MoveNext()
        {
            if (_current >= _items - 1) return false;
            _current++;

            if (_remainingUntilYield == 0)
            {
                _remainingUntilYield = _yieldEvery;
                return Task.Run(() => true).GetAwaiter().GetResult();
            }
            else
            {
                _remainingUntilYield--;
                return true;
            }
        }
        int IEnumerator<int>.Current => _current;
        IEnumerator IEnumerable.GetEnumerator() => ((IEnumerable<int>)this).GetEnumerator();
        object IEnumerator.Current => _current;
        void IDisposable.Dispose() { }
        void IEnumerator.Reset() { }
    }
}

namespace System.Collections.Generic
{
    public interface IAsyncEnumerable1<out T>
    {
        IAsyncEnumerator1<T> GetAsyncEnumerator(CancellationToken cancellationToken = default(CancellationToken));
    }
    public interface IAsyncEnumerator1<out T>
    {
        Task<bool> MoveNextAsync();
        T Current { get; }
    }

    public interface IAsyncEnumerable2<out T>
    {
        IAsyncEnumerator2<T> GetAsyncEnumerator(CancellationToken cancellationToken = default(CancellationToken));
    }
    public interface IAsyncEnumerator2<out T>
    {
        Task<bool> WaitForNextAsync();
        T TryGetNext(out bool success);
    }
}
```
On my machine, I get output like the following:
```
                      Method |    Items | YieldEvery |      Mean |     Error |    StdDev |
---------------------------- |--------- |----------- |----------:|----------:|----------:|
            MoveNext_Current | 10000000 |        100 | 277.45 ms | 3.6161 ms | 3.3825 ms |
       MoveNextAsync_Current | 10000000 |        100 | 257.56 ms | 2.0816 ms | 1.9471 ms |
 WaitForNextAsync_TryGetNext | 10000000 |        100 | 205.45 ms | 1.4457 ms | 1.3523 ms |
            MoveNext_Current | 10000000 | 4294967295 |  67.76 ms | 1.3948 ms | 1.3699 ms |
       MoveNextAsync_Current | 10000000 | 4294967295 |  76.94 ms | 0.3676 ms | 0.3259 ms |
 WaitForNextAsync_TryGetNext | 10000000 | 4294967295 |  45.87 ms | 0.2139 ms | 0.1786 ms |
```
In these benchmarks, the enumerator hands back 10,000,000 integers.  In the first three lines, it's yielding (completing asynchronously) every 100 items, and in the second three lines, it's essentially just yielding once at the very beginning and then never again.

Note that the difference becomes even more stark if an async method is used to implement `MoveNextAsync` and `WaitForNextAsync`, as async methods have more overhead associated with them than do normal methods.  If I change the methods accordingly:
```C#
async Task<bool> IAsyncEnumerator1<int>.MoveNextAsync()
{
    if (_current >= _items - 1) return false;
    _current++;

    if (_remainingUntilYield == 0)
    {
        _remainingUntilYield = _yieldEvery;
        await Task.Yield();
    }
    else
    {
        _remainingUntilYield--;
    }
    return true;
}

async Task<bool> IAsyncEnumerator2<int>.WaitForNextAsync()
{
    if (_current >= _items - 1) return false;

    if (_remainingUntilYield == 0)
    {
        _remainingUntilYield = _yieldEvery;
        await Task.Yield();
    }
    return true;
}
```
the difference is even more stark:
```
                      Method |    Items | YieldEvery |      Mean |     Error |    StdDev |
---------------------------- |--------- |----------- |----------:|----------:|----------:|
            MoveNext_Current | 10000000 |        100 | 262.35 ms | 4.0718 ms | 3.4001 ms |
       MoveNextAsync_Current | 10000000 |        100 | 785.74 ms | 4.9710 ms | 4.4066 ms |
 WaitForNextAsync_TryGetNext | 10000000 |        100 | 164.98 ms | 1.1382 ms | 1.0647 ms |
            MoveNext_Current | 10000000 | 4294967295 |  64.39 ms | 0.9235 ms | 0.8639 ms |
       MoveNextAsync_Current | 10000000 | 4294967295 | 669.31 ms | 4.6653 ms | 3.6423 ms |
 WaitForNextAsync_TryGetNext | 10000000 | 4294967295 |  48.88 ms | 0.5535 ms | 0.5177 ms |
```

Discarded options considered:
- `Task<bool> WaitForNextAsync(); bool TryGetNext(out T result);`: `out` parameters can't be covariant.  There's also a small impact here (an issue with the try pattern in general) that this likely incurs a runtime write barrier for reference type results.

(The remainder of this proposal is written in terms of the `MoveNextAsync`/`Current`-based interface, but it could be changed to use the `WaitForNextAsync`/`TryGetNext` interface if desired.)

#### Cancellation
Logically it makes sense for each individual `MoveNextAsync` operation to accept a `CancellationToken` so that it can be canceled in a very fine-grained way.  However:
1. That's prohibitively expensive in many situations, as the code in many iterators needs to register/unregister for a callback on each item.
2. There's rarely a need to supply a different `CancellationToken` per item.
3. Logically the individual `MoveNextAsync` calls are just part of the larger async operation to do with enumerating the whole enumerator, so it makes sense to treat it as a unit from a cancellation perspective.

Given this, cancellation is defined at the enumerator level rather than at the sub-`MoveNextAsync` level.

#### ConfigureAwait

In support of `foreach`, the following types would be added to .NET as well, likely to System.Threading.Tasks.Extensions.dll:
```C#
// Approximate implementation, omitting arg validation and the like
namespace System.Threading.Tasks
{
    public static class AsyncEnumerableExtensions
    {
        public static ConfiguredAsyncEnumerable<T> ConfigureAwait<T>(this IAsyncEnumerable<T> enumerable, bool continueOnCapturedContext) =>
            new ConfiguredAsyncEnumerable<T>(enumerable, continueOnCapturedContext);

        public static ConfiguredAsyncEnumerator<T> ConfigureAwait<T>(this IAsyncEnumerator<T> enumerator, bool continueOnCapturedContext) =>
            new ConfiguredAsyncEnumerable<T>.Enumerator(enumerator, continueOnCapturedContext);

        public struct ConfiguredAsyncEnumerable<T>
        {
            private readonly IAsyncEnumerable<T> _enumerable;
            private readonly bool _continueOnCapturedContext;

            internal ConfiguredAsyncEnumerable(IAsyncEnumerable<T> enumerable, bool continueOnCapturedContext)
            {
                _enumerable = enumerable;
                _continueOnCapturedContext = continueOnCapturedContext;
            }

            public ConfiguredAsyncEnumerator<T> GetAsyncEnumerator() =>
                new ConfiguredAsyncEnumerator<T>(_enumerable.GetAsyncEnumerator(), _continueOnCapturedContext);

            public struct Enumerator
            {
                private readonly IAsyncEnumerator<T> _enumerator;
                private readonly bool _continueOnCapturedContext;

                internal Enumerator(IAsyncEnumerator<T> enumerator, bool continueOnCapturedContext)
                {
                    _enumerator = enumerator;
                    _continueOnCapturedContext = continueOnCapturedContext;
                }

                public ConfiguredTaskAwaitable<bool> MoveNextAsync() =>
                    _enumerator.MoveNextAsync().ConfigureAwait(_continueOnCapturedContext);

                public T Current => _enumerator.Current;

                public ConfiguredTaskAwaitable DisposeAsync() =>
                    _enumerator.DisposeAsync().ConfigureAwait(_continueOnCapturedContext);
            }
        }
    }
}
```
Use of this will be shown in the subsequent section.

## foreach

`foreach` will be augmented to support `IAsyncEnumerable<T>` in addition to its existing support for `IEnumerable<T>`.  Further, `foreach` will be augmented to support `IAsyncEnumerator<T>`.  And it will support these APIs as patterns if the relevant members are exposed publicly, falling back to using the interface directly if not.

### Syntax

Using the syntax:
```C#
foreach (var i in enumerable)
```
C# will continue to treat `enumerable` as a synchronous enumerable, such that even if it exposes the relevant APIs for async enumerables (exposing the pattern or implementing the interface), it will only consider the synchronous APIs.

To force `foreach` to instead only consider the asynchronous APIs, `await` is inserted as follows:
```C#
foreach await (var i in enumerable)
```

No syntax would be provided that would support using either the async or the sync APIs; the developer must choose based on the syntax used.

Discarded options considered:
- _`foreach (var i in await enumerable)`_: This is already valid syntax, and changing its meaning would be a breaking change.  This means to `await` the `enumerable`, get back something synchronously iterable from it, and then synchronously iterate through that.
- _`foreach (var i await in enumerable)`, `foreach (var await i in enumerable)`, `foreach (await var i in enumerable)`_: These all suggest that we're awaiting the next item, but there are other awaits involved in foreach, in particular if the enumerable is an `IAsyncDisposable`, we will be `await`'ing its async disposal.  That await is as the scope of the foreach rather than for each individual element, and thus the `await` keyword deserves to be at the `foreach` level.  Further, having it associated with the `foreach` gives us a way to describe the `foreach` with a different term, e.g. a "foreach await".
- _`await foreach (var it in enumerable)`_: This suggests that the entire `foreach` is somehow returning something that's being `await`'d, but it's not.

### Pattern-based Compilation

The compiler will bind to the pattern-based APIs, if they exist, preferring those over using the interface (the pattern may be satisfied with instance methods or extension methods).  The requirements for the pattern are:
- The enumerable may expose a `GetAsyncEnumerator` method that may be called with no arguments and that returns something that may be enumerated.  If it does, that enumerator is then used for the remainder of the requirements; if it doesn't, then it itself must meet the enumerator requirements.
- The enumerator must expose a `MoveNextAsync` method that may be called with no arguments and that returns something which may be `await`ed and whose `GetResult()` returns a `bool`.
- The enumerator must also expose `Current` property whose getter returns a `T` representing the kind of data being enumerated.
- The enumerator may optionally expose a `DisposeAsync` method that may be invoked with no arguments and that returns something that can be `await`ed and whose `GetResult()` returns `void`.

This code:
```C#
var enumerable = ...;
foreach await (T item in enumerable)
{
   ...
}
```
is translated to the equivalent of:
```C#
var enumerable = ...;
var enumerator = enumerable.GetAsyncEnumerator();
try
{
    while (await enumerator.MoveNextAsync())
    {
       T item = enumerator.Current;
       ...
    }
}
finally
{
    await enumerator.DisposeAsync(); // omitted, along with the try/finally, if the enumerator doesn't expose DisposeAsync
}
```

Further, this code:
```C#
foreach await (T item in enumerator)
{
   ...
}
```
is translated to the equivalent of:
```C#
try
{
    while (await enumerator.MoveNextAsync())
    {
       T item = enumerator.Current;
       ...
    }
}
finally
{
    await enumerator.DisposeAsync(); // omitted, along with the try/finally, if the enumerator doesn't expose DisposeAsync
}
```

If the iterated type doesn't expose the right pattern, the interfaces will be used.

### ConfigureAwait

This pattern-based compilation will allow `ConfigureAwait` to be used on all of the awaits, via the `ConfigureAwait` extension method described earlier:
```C#
foreach await (T item in enumerable.ConfigureAwait(false))
{
   ...
}
```
Note that this approach will not enable `ConfigureAwait` to be used with pattern-based enumerables, but then again it's already the case that the `ConfigureAwait` is only exposed as an extension on `Task`/`Task<T>`/`ValueTask<T>` and can't be applied to arbitrary awaitable things, as it only makes sense when applied to Tasks (it controls a behavior implemented in Task's continuation support), and thus doesn't make sense when using a pattern where the awaitable things may not be tasks.  Anyone returning awaitable things can provide their own custom behavior in such advanced scenarios.

### Cancellation

If code desires to provide a `CancellationToken` that can be used to cancel the enumerator, that can then be done simply by calling `GetEnumerator` on the enumerable rather than having the compiler do it explicitly:
```C#
foreach await (T item in enumerable.GetEnumerator(cancellationToken))
{
   ...
}
```
That `CancellationToken` will be respected by the enumerator however it sees fit. 

## Async Iterators

The language / compiler will support producing `IAsyncEnumerable<T>`s and `IAsyncEnumerator<T>`s in addition to consuming them. Today the language supports writing an iterator like:
```C#
static IEnumerable<int> MyIterator()
{
    try
    {
        for (int i = 0; i < 100; i++)
        {
            Thread.Sleep(1000);
            yield return i;
        }
    }
    finally
    {
        Thread.Sleep(200);
        Console.WriteLine("finally");
    }
}
```
but `await` can't be used in the body of these iterators.  We will add that support.

### Syntax

The existing language support for iterators infers the iterator nature of the method based on whether it contains any `yield`s.  The same will be true for async iterators.  Such async iterators will be demarcated and differentiated from synchronous iterators via adding `async` to the signature, and must then also have either `IAsyncEnumerable<T>` or `IAsyncEnumerator<T>` as its return type.  For example, the above example could be written as an async iterator as follows:
```C#
static IAsyncEnumerable<int> MyIterator()
{
    try
    {
        for (int i = 0; i < 100; i++)
        {
            await Task.Delay(1000);
            yield return i;
        }
    }
    finally
    {
        await Task.Delay(200);
        Console.WriteLine("finally");
    }
}
```

Alternatives considered:
- _Not using `async` in the signature_: This may not be technically required, as the use of `IAsyncEnumerable<T>`/`IAsyncEnumerator<T>` should be sufficient to differentiate from synchronous iterators.  However, we've established that `await` may only be used in methods marked as `async`, and it seems important to keep the consistency.
- _Enabling custom builders for `IAsyncEnumerable<T>`_:  That's something we could look at for the future, but the machinery is complicated and we don't support that for the synchronous counterparts.
- _Having an `iterator` keyword_: Async iterators would use `async iterator` in the signature, and `yield` could only be used in `async` methods that included `iterator`; `iterator` would then be made optional on synchronous iterators.  Depending on your perspective, this has the benefit of making it very clear by the signature of the method whether `yield` is allowed and whether the method is actually meant to return instances of type `IAsyncEnumerable<T>` rather than the compiler manufacturing one based on whether the code uses `yield` or not.  But it is different from synchronous iterators, which don't and can't be made to require one.  Plus some developers don't like the extra syntax.

### Compilation

While there may be alternative approaches, one approach for compilation is to logically treat the compilation as an iterator but generate the MoveNextAsync methods as async methods, e.g. the synchronous iterator above is compiled to something like the following:
```C#
int <>1__state;
int <>2__current;
int <>l__initialThreadId;
int <i>5__1;

bool MoveNext()
{
    try
    {
        int state = this.<>1__state;
        if (state == 0)
        {
            this.<> 1__state = -3;
            this.<i> 5__1 = 0;
            while (this.<i>5__1 < 100)
            {
                Thread.Sleep(1000);
                this.<> 2__current = this.<i>5__1;
                this.<> 1__state = 1;
                return true;

            Label_004B:
                this.<> 1__state = -3;
                this.<i>5__1++
            }
            this.<>m__Finally1();
            return false;
        }
        if (state != 1) return false;
        goto Label_004B;
    }
    fault { this.System.IDisposable.Dispose(); }
}

void <>m__Finally1()
{
    this.<> 1__state = -1;
    Thread.Sleep(200);
    Console.WriteLine("finally");
}

void IDisposable.Dispose()
{
    switch (this.<> 1__state)
    {
        case -3:
        case 1:
            try { } finally
            {
                this.<> m__Finally1();
            }
    }
    break;
}
```

For the async iterator, it could similarly be compiled as something along the lines of (waving my hands a bit):
```C#
int <>1__state;
int <>2__current;
int <>l__initialThreadId;
int <i>5__1;

async Task<bool> MoveNextAsync()
{
    try
    {
        int state = this.<>1__state;
        if (state == 0)
        {
            this.<> 1__state = -3;
            this.<i> 5__1 = 0;
            while (this.<i>5__1 < 100)
            {
                await Task.Delay(1000);
                this.<> 2__current = this.<i>5__1;
                this.<> 1__state = 1;
                return true;

            Label_004B:
                this.<> 1__state = -3;
                this.<i>5__1++
            }

            this.<> 1__state = -1; // unlike sync iterator, don't factor this out into a finally helper, to avoid introducing awaits
            await Task.Delay(200);
            Console.WriteLine("finally");

            return false;
        }
        if (state != 1) return false;
        goto Label_004B;
    }
    catch
    {
        switch (this.<>1__state)
        {
            case -3:
            case 1:
                try { } finally
                {
                    this.<> 1__state = -1;
                    await Task.Delay(200);
                    Console.WriteLine("finally");
                }
        }
        throw;
    }
}

async Task IAsyncDisposable.DisposeAsync()
{
    switch (this.<> 1__state)
    {
        case -3:
        case 1:
            try { } finally
            {
                this.<> 1__state = -1;
                await Task.Delay(200);
                Console.WriteLine("finally");
            }
    }
    break;
}
```
We will need to avoid introducing any `await`s that weren't in the written C# code, so as not to introduce any unexpected behaviors (e.g. introducing a continuation that may then be scheduled back to an undesired context).  And we will want to consider possible optimizations that could avoid various kinds of overhead involved in such a code generation approach.

### Cancellation

Cancellation may be achieved simply by passing a `CancellationToken` into the iterator as is the case for any other method:
```C#
static IAsyncEnumerable<int> MyIterator(CancellationToken cancellationToken)
{
    for (int i = 0; i < 100; i++)
    {
        await Task.Delay(1000, cancellationToken);
        yield return i;
    }
}
```
The compiler doesn't treat a `CancellationToken` argument as special in any way.

This does "bake" the `CancellationToken` into the enumerable, such that multiple calls to `GetEnumerator` will all implicitly share the same token, but we expect it to be rare with async enumerators to iterate through a single call to an iterator multiple times.  In other words, while we expect it'll be common to have code that does something like:
```C#
foreach await (T item in ProduceDataAsync(cancellationToken)) { ... }
foreach await (T item in ProduceDataAsync(cancellationToken)) { ... }
foreach await (T item in ProduceDataAsync(cancellationToken)) { ... }
```
we expect it to be much, much less common to want to do:
```C#
var enumerable = ProduceDataAsync(cancellationToken);
foreach await (T item in enumerable) { ... }
foreach await (T item in enumerable) { ... }
foreach await (T item in enumerable) { ... }
```
and to care about using a different `CancellationToken` in each iteration (if you do care, you can just invoke it multiple times).

Alternatives considered:
- _Exposing the `CancellationToken` passed to the enumerable's `GetEnumerator` into the body of the method_: This could be done, for example, if the compiler added a new keyword like `iterator` that referred to state associated with the iterator, and the `CancellationToken` could be exposed off of that such that the body of the iterator could access `iterator.CancellationToken`.  But that's a lot of machinery for little gain, and it could be introduced in the future if desired / if additional functionality might be exposed off of `iterator` to motivate it.

## LINQ

There are over ~200 overloads of methods on the `System.Linq.Enumerable` class, all of which work in terms of `IEnumerable<T>`; some of these accept `IEnumerable<T>`, some of them produce `IEnumerable<T>`, and many do both.  Adding LINQ support for `IAsyncEnumerable<T>` would likely entail duplicating all of these overloads for it, for another ~200.  And since `IAsyncEnumerator<T>` is likely to be more common as a standalone entity in the asynchronous world than `IEnumerator<T>` is in the synchronous world, we could potentially need another ~200 overloads that work with `IAsyncEnumerator<T>`.  Plus, a large number of the overloads deal with predicates (e.g. `Where` that takes a `Func<T, bool>`), and it may be desirable to have `IAsyncEnumerable<T>`-based overloads that deal with both synchronous and asynchronous predicates (e.g. `Func<T, Task<bool>>` in addition to `Func<T, bool>`).  While this isn't applicable to all of the now ~400 new overloads, a rough calculation is that it'd be applicable to half, which means another ~200 overloads, for a total of ~600 new methods.

That is a staggering number of APIs, with the potential for even more when extension libraries like Interactive Extensions (Ix) is considered.  But Ix already has an implementation of many of these, and there doesn't seem to be a great reason to duplicate that work; we should instead help the community improve Ix and recommend it for when developers want to use LINQ with `IAsyncEnumerable<T>`.

There is also the issue of query comprehension syntax.  The pattern-based nature of query comprehensions would allow them to "just work" with some operators, e.g. if Ix provides the following methods:
```C#
public static IAsyncEnumerable<TResult> Select<TSource, TResult>(this IAsyncEnumerable<TSource> source, Func<TSource, TResult> func);
public static IAsyncEnumerable<T> Where(this IAsyncEnumerable<T> source, Func<T, bool> func);
```
then this C# code will "just work":
```C#
IAsyncEnumerable<int> enumerable = ...;
IAsyncEnumerable<int> result = from item in enumerable
                               where item % 2 == 0
                               select item * 2;
```
However, there is no query comprehension syntax that supports using `await` in the clauses, so if Ix added, for example:
```C#
public static IAsyncEnumerable<TResult> Select<TSource, TResult>(this IAsyncEnumerable<TSource> source, Func<TSource, Task<TResult>> func);
```
then this would "just work":
```C#
IAsyncEnumerable<int> result = from item in enumerable
                               where item % 2 == 0
                               select SomeAsyncMethod(item);

async Task<int> SomeAsyncMethod(int item)
{
    await Task.Yield();
    return item * 2;
}
```
but there'd be no way to write it with the `await` inline in the `select` clause.  As a separate effort, we could look into adding `async { ... }` expressions to the language, at which point we could allow them to be used in query comprehensions and the above could instead be written as:
```C#
IAsyncEnumerable<int> result = from item in enumerable
                               where item % 2 == 0
                               select async
                               {
                                   await Task.Yield();
                                   return item * 2;
                               };
```

## Integration with other asynchronous frameworks

Integration with `IObservable<T>` and other asynchronous frameworks (e.g. reactive streams) would be done at the library level rather than at the language level.  For example, all of the data from an `IAsyncEnumerator<T>` can be published to an `IObserver<T>` simply by `foreach`'ing over the enumerator and `OnNext`'ing the data to the observer, so an `AsObservable<T>` extension method is possible.  Consuming an `IObservable<T>` in a `foreach await` requires buffering the data (in case another item is pushed while the previous item is still being processing), but such a push-pull adapter can easily be implemented to enable an `IObservable<T>` to be pulled from with an `IAsyncEnumerator<T>`.  Etc.  Rx/Ix already provide prototypes of such implementations, and libraries like https://github.com/dotnet/corefxlab/tree/master/src/System.Threading.Tasks.Channels provide various kinds of buffering data structures as well as exposing both observable and async enumerable-based production and consumption.  The language need not be involved at this stage.