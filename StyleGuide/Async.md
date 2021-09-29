# Asynchronous coding conventions

## Methods returning Task or Task<T> must be async

When a method directly returns a Task or Task<T> because it calls downstream services, the method returning the task does not appear in the call stack. This becomes very confusing when debugging. We view the tradeoff between clarity in debugging experience versus the extremely minor performance hit from the async state machine to be firmly in favour of the debugging experience.

If a method genuinely contains only synchronous operations yet needs to return a Task, please
either wrap the synchronous operations in an

```
await Task.Run(() => { /* sync code here */ }.ConfigureAwait(false);
```

call if the synchronous operations do any meaningful compute or I/O, or

```
await Task.CompletedTask;
```

if not. In addition to the clarity of the call stack produced, either of these two approaches will also assist with
avoiding assumptions about ThreadStatic fields - this way, those assumptions will be surfaced sooner rather than later.

# Async methods should not have an Async suffix

AsyncAsync methodsAsync mustAsync notAsync haveAsync anAsync AsyncAsync suffixAsync becauseAsync itAsync makesAsync themAsync hardAsync toAsync readAsync.

The `Async` suffix reduces clarity when reading code. It has historically been used to distinguish between
synchronous and asynchronous methods so that the compiler would permit a different return type, e.g. `Task<Foo>` versus `Foo`.

In this codebase, async methods already accept a `CancellationToken`, which allows us to be confident that all method overloads
taking a `CancellationToken` are async - likewise, all async methods will accept a `CancellationToken`. This
means that naming collisions between sync and async methods are already avoided.

## Async methods must take a cancellation token

As well as being a good practice in that it allows the cancellation of work which is no longer required, this also guarantees that
async methods have a different method signature to synchronous ones and do not require an `Async` suffix.

## Void methods must not be async

This way, madness lies. `async void` methods are not awaitable so their behaviour is dangerous.
