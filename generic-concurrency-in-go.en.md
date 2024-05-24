---
title: "Generic Concurrency in Go"
date: 2024-04-27T22:00:00+00:00
draft: false
image: generic-concurrency-in-go.jpg
imageWidth: 600
description: "Generics and goroutines are great tools we can leverage to have reusable general purpose concurrent processing in our programs."
---
{{< img src="generic-concurrency-in-go.jpg" width="600" alt="Gopher-like octopus thoughtfully looks up" >}}

Hello Gophers!

In this article, I want to share my thoughts and ideas that I've accumulated over time regarding generics in Go, and in particular, concurrency patterns, which now can become more reusable and convenient with the use of generics. 

## TL;DR

Generics and goroutines (and iterators in the future) are great tools we can leverage to have reusable general purpose concurrent processing in our programs.

In this article we explore the possibilities of combining them together.

## Introduction

Let's quickly touch a surface with some basic context and small examples to see what problem generics solve and how we can fuse it existing concurrency model.

> In this article we are going to think a lot about mapping of collections (sets, sequences) of elements. So the _mapping_ is a process that results in a new collection of elements where each element is a result of a call to some function `f()` with the corresponding element from the initial collection.

### Pre-Generics era

Let's define the first simple integer numbers mapping (which in Go snippets we will call `transform()` to not confuse with the builtin `map` type):

```go
func transform([]int, func(int) int) []int
```

{{<details summary="Sample implementation">}} 
```go
func transform(xs []int, f func(int) int) []int {
    ret := make([]int, len(xs))
    for i, x := range xs {
        ret[i] = f(x)
    }
    return ret
}
```
{{</details>}}

An example use of such function would look like this:

```go
// Output: [1, 4, 9]
transform([]int{1, 2, 3}, func(n int) int {
    return n * n
})
```

Now lets assume we want to map integers to strings. That's easy -- we can define `transform()` _just slightly_ different:

```go
func transform([]int, func(int) string) []string
```

So we can use it this way:

```go
// Output: ["1", "2", "3"]
transform([]int{1, 2, 3}, strconv.Itoa) 
```

What about reporting whether a number is odd or even? Just _another tiny correction_:

```go
func transform([]int, func(int) bool) []bool
```

So we could use it this way:

```go
// Output: [false, true, false]
transform([]int{1, 2, 3}, func(n int) bool {
    return n % 2 == 0
})
```

_Generalising_ the corrections of `transform()` we've made above for each use case, we can say that **regardless** of the types it operates on, it does exactly the same thing over and over again. If we were to generate the code for each type involved using `text/template` templates, we could do it like this:

```go
func transform_{{ .A }}_{{ .B }}([]{{ .A }}, func({{ .A }}) {{ .B }}) []{{ .B }}

// transform_int_int([]int, func(int) int) []int
// transform_int_string([]int, func(int) string) []string
// transform_int_bool([]int, func(int) bool) []bool
```

> Actually there were a few nice code generation tools that were doing almost this templating for pre-generic versions of Go. [genny][genny] is just one example.

### Generics era

Thanks to the generics, we now have an ability to _parametrize_ functions and types with _type parameters_ and define `tranform()` this way:

```go
func transform[A, B any]([]A, func(A) B) []B
```

{{<details summary="And the implementation changes just a little bit!">}} 
```go
func transform[A, B any](xs []A, f func(A) B) []B {
    ret := make([]B, len(xs))
    for i, x := range xs {
        ret[i] = f(x)
    }
    return ret
}
```
{{</details>}}

So we can use it now for any input and output types (assuming we have `square(int) int` and `isEven(int) bool` defined somewhere in the package):

```go
transform([]int{1, 2, 3}, square)       // [1, 4, 9]
transform([]int{1, 2, 3}, strconv.Itoa) // ["1", "2", "3"]
transform([]int{1, 2, 3}, isEven)       // [false, true, false]
```

## Concurrent mapping

Okay, now let's get on to the main subject of this article and focus on concurrency patterns that can benefit from generics.

### The `x/sync/errgroup` package

Before jumping into lots of coding snippets, let's make a tiny step aside and look at (very popular) [golang.org/x/sync/errgroup][errgroup] Go library. In short, it allows you to start various number of goroutines to perform different tasks and wait for their completion or failure. 

It is supposed to be used this way: 
```go
// Create workers group and a context which will get canceled if any of the
// tasks fails.
g, gctx := errgroup.WithContext(ctx)
g.Go(func() error {
	return doSomeFun(gctx)
})
g.Go(func() error {
	return doEvenMoreFun(gctx)
})
if err := g.Wait(); err != nil {
	// handle error
}
```

The reason I mentioned the package is because, when viewed from a slightly different and a bit generalised perspective, it essentially looks to be the same mapping thing. The package allows you to _concurrently map_ a set of tasks into a corresponding set of results and provides a generalised way for errors handling and propagation, as well as cancellation of subtasks (via context cancellation) if any of them fails.

In this article we want to build something similar, and, as the repeated use of the "generic" word suggests, we will be doing this in a generic way.

### Naive implementation

Getting back to the `transform()` function. Let's assume that all the calls to `f()` can be done concurrently without breaking our (or anyone else's) program. Then we can start with this naive _concurrent_ implementation:

{{<func file="./src/async/async.go" name="transform_00" next="transform" lang="go">}}

That is, we start a goroutine per each element of the input and call `f(elem)`. Then we store the result at the corresponding index in the shared slice `bs`. No context, no cancellations, no errors even -- this one doesn't look like something very helpful in anything besides pure computation.

### Context cancellation

In real world many or even most of the concurrent tasks, especially the i/o related, would be controlled by [`context.Context`][context] instance. Since there is a context, there could be timeout or cancellation. Let's think of it this way (here and after I'll highlight the lines that were added compared to the previous code sample):

{{<func file="./src/async/async.go" name="transform_01" next="transform" lang="go" class="focus" hl_lines="2 4 10 12-13 20-23 28-32">}}

Now we have one more shared slice `es` to store errors potentially returned by `f()`. If any goroutine's `f()` fails, we cancel the entire `transform()` context and expect every inflight `f()` call to respect the cancellation and return as soon as possible.

### Limiting concurrency

In reality, we cannot assume too much about `f()` implicitly. Users of `transform()` might want to limit the number of concurrent calls to `f()`. For example, `f()` can map a url to the result of an http request. Without any limits we can overwhelm the server or get banned ourselves.

Let's not think about the parameters structure for now, and just add a parallelism `int` argument to the function arguments.

At this point we need to switch from using `sync.WaitGroup` to a [semaphore][semaphore] `chan`, as we want to control the (maximum) number of simultaneously running goroutines as well as to handle the context cancellation, both by using `select`. 

{{<func file="./src/async/async.go" name="transform_02" next="transform" lang="go" class="focus" hl_lines="13-14 17-19 21-36 38-42 49-64">}}

> For this and next iterations of `tranform()` we actually could leave the implementation as it is now and leave both use cases at the mercy of `f()` implementation. For example, we could just start `N` goroutines regardless the concurrency limits and let the user of `transform()` to partially serialise them the way they want to. That would require an overhead of starting `N` goroutines instead of `P` (where `P` is the "parallelism" limit, which can be much less than `N`). It also would imply some compute overhead on synchronisation of the goroutines depending on the mechanism used. Since all of this is unnecessary, we proceed with the implementation the hard way, but for many of the cases this complications are optional. 
> {{<details summary="Example user based implementation">}} 
```go
// Initialised x/time/rate.Limiter instance.
var lim *rate.Limiter
transform(ctx, as, func(_ context.Context, url string) (int, error) {
    if err := lim.Wait(ctx); err != nil {
        return 0, err
    }

    // Process url.

    return 42, nil
})
```
{{</details>}}

### Reusing goroutines

In the previous iteration we were starting a goroutine per each task, but no more `parallelism` goroutines at a time. This highlights another interesting option -- users might want to have a custom execution context per each goroutine. For example, suppose we have `N` tasks with maximum `P` running concurrently (and `P` can be significantly less than `N`). If each task requires some form of resource preparation, such as a large memory allocation, a database session, or maybe a single-threaded Cgo "coroutine", it would seem logical to prepare only `P` resources and reuse them among workers through context.

Again, let's keep the structure of passing options aside.

{{<func file="./src/async/async.go" name="transform_03" next="transform" lang="go" class="focus" hl_lines="3 20 33-36 51-59 64-72">}}

At this point we start _up to_ `P` goroutines, and distribute tasks across them using non-buffered channel `wrk`. The channel is non-buffered because we want to have an immediate runtime "feedback" to know if there are any idle workers at the moment or if we should consider starting a new one. Once all the tasks processed or any of the `f()` calls fails, we signal (by doing `close(wrk)`) all the started goroutines to return.

> As in the previous section, this might be done inside `f()` too, for example, by using [`sync.Pool`][sync-pool]. `f()` could acquire a resource (or create, in case when there are no idle resources) and release it once it's not needed anymore. Since the set of goroutines is fixed, odds are resources can have a nice CPU locality, so the overhead could be minimal. 
> {{<details summary="Example user based implementation">}} 
```go
// Note that this snippet assumes `transform()` can limit its concurrency.
var pool sync.Pool
transform(ctx, 8, as, func(_ context.Context, userID string) (int, error) {
    sess := pool.Get().(*db.Session)
    if sess == nil {
        // Initialise database session.
    }
    defer pool.Put(sess)

    // Process userID.

    return 42, nil
})
```
{{</details>}}

## Generalisation of `transform()`

So far our focus has been on mapping slices, which in many cases is enough. However, what if we want to map `map` types, or maybe `chan` even? Can we map anything that we can `range` over? And as in `for` loops, do we always need to _map_ values really?

These all are interesting questions which lead us to an idea that we can generalise our concurrent iteration approach. We can have a "low level" function that behaves almost the same but doing a bit less assumptions on its input and output. Then, it will take just a little effort to build a bit more specific `transform()` on top of it. Let's call the function `iterate()` and represent its input and output as functions instead of data types. We will `pull()` elements from the input and `push()` the results back to the user. This way the user of `iterate()` would control the way it provides the input elements and the way it handles the results.

We also have to consider what results `iterate()` should push to the user. As we plan to make the mapping of input elements optional, `(B, error)` doesn't seem to be the only right and obvious option anymore. This part is really subtle actually, and maybe majority of use cases of the function would benefit of keeping it as it was and returning the error explictily. However, semantically it doesn't make much sense as the `f()` result is only being proxied down to the `push()` call without any processing, which means that `iterate()` really has no any presumptions on the result. In other words, the result makes sense for the `push()` function implementation only, which is given by the user. Additionally, this signature will work better with Go iterators, which we'll cover in the end of this article. So having this in mind let's try to reduce number of the return parameters down to one. Since we intend to push results through a function call, we should likely do it in a serialised way. `transform()` and later `iterate()` have all the needed synchronisation already internally, so this way the user would collect the results without the need for extra synchronisation efforts on their side.

Another thing to cover is the way we were handling errors earlier -- we did not map an error to the input element that caused it. It is true that `f()` could wrap an error, but it is more clean for `f()` to remain unaware of the way it will be called. In other words, `f()` should not assume it's being called as the `iterate()` argument. If it was invoked with a single element `a`, there is no point to wrap `a` into the error, as it's obvious to the caller that this particular `a` caused the error. This principle leads us to another observation -- any potential binding of an input element to an error (or any other result) should also occur during the `push()` execution. For the same reasons, it is the `push()` function that should control the iteration and decide if a faulty result should interrupt the loop.

Additionally, this `iterate()` design naturally provides a nice flow control. If the user does something slow in the `push()` function, the other worker goroutines will eventually pause processing new elements. This is because they will get blocked by sending their `f()` call results into the `res` channel, which in turn is being drained by the function that calls `push()`.

As the function code listing becomes too big, let's cover it part by part.

### Signature

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" class="focus" lo="0" hi="8" hl_lines="1 5-8">}}

Both arguments and return parameters no longer have input and output slices as it was for the `transform()` function previously. Instead, input elements are pulled in by calling the `pull()` function and results are pushed back to the user by calling the `push()` function. Note that the `push()` function returns a `bool` parameter that controls the iteration -- once `false` is returned, no more `push()` calls will be made and all ongoing `f()` executions will get their context canceled. The `iterate()` returns just an error, which can only be non-nil when the iteration is terminated due to the given `ctx` cancellation -- otherwise there is no way of knowing why the iteration has stopped.

> Even though there are just three cases when loop can be terminated:
> - `pull()` returned `false`, meaning no more elements to process.
> - `push()` returned `false`, meaning the user doesn't need any further results.
> - the parent `ctx` got cancelled.
> 
> Without user code complications it's hard to say whether all of the elements were processed before the parent context got canceled.
> 
> {{<details summary="Example" raw="true">}} 
Let's assume we want implement concurrent `forEach()` using `iterate()`:
{{<func file="./src/async/async.go" name="forEach" next="forEach" lang="go">}}
{{</details>}}


### Prologue

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" lo="8" hi="40">}}

In previous versions of `transform()` we stored results according to the index of the input element in the results slice, much like `bs[i] = f(as[i])`. This is no longer possible with function-based input and output. So, as soon we have a result, we likely need to `push()` it to the user immediately. This is why we want to have two goroutines for dispatching the input elements and pushing the results back to the user -- while we are dispatching an input, we might already get an output.

### Dispatch goroutine

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" lo="41" hi="71">}}

### Dispatch loop

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" class="focus" lo="72" hi="105" hl_lines="1 3 6-14 17 23 27-29 32-33">}}

### Worker goroutine

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" class="focus" lo="106" hi="134" hl_lines="2 13-21">}}

### Results collection

{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go" lo="134" hi="181">}}

As you can see, results are pushed back to the user in the random order -- not the way they were pulled in. This is expected and because we process them concurrently.

> Here's an opinionated thought: this combination of `sync.WaitGroup` and `sem` channel is a rare example of a justified co-existence of both synchronisation mechanisms in the same code. I believe that in most cases where a channel exists, the wait group is redundant, and vice versa.

And phew, that's it! It was not easy, but it is what we want. Let's see how can we use it in the next sections.

{{<details summary="Complete code listing" raw="true">}} 
{{<func file="./src/async/async.go" name="iterate" next="iterate" lang="go">}}
{{</details>}}

## Using `iterate()` to `transform()`

To test how the generic iteration function can solve the _mapping_ problem let's re-implement `transform()` using it. It obviously now looks much shorter as we moved the concurrent iteration complexity away from it and can focus basically just on storing mapping results.

{{<func file="./src/async/async.go" name="transform_05" next="transform" lang="go">}}

## Reimplemented `errgroup`

To conclude the analogy with the `errgroup` package, let's try to implement something similar using `iterate()` approach. 

```go
type taskFunc func(context.Context) error
```

{{<func file="./src/async/async.go" name="errgroup" next="errgroup" lang="go">}}

So the use of the function would be very similar to `errgroup` package:
```go
// Create the workers group and a context which will be canceled if any of the
// tasks fails.
g, wait := errgroup(ctx)
g(func(ctx context.Context) error {
	return doSomeFun(gctx)
})
g(func(ctx context.Context) error {
	return doEvenMoreFun(gctx)
})
if err := wait(); err != nil {
	// handle error
}
```

## Go Iterators

Let's briefly look at the near future of Go in relation to the ideas implemented above. 

With the recent (as of Go 1.22) [range over functions experiment][rangefunc-wiki] it is possible to do a usual `range` over functions that are compatible with [sequences iterator][iter-seq] types defined by the `iter` package. A quite new concept in Go which is hopefully being shipped in the future versions of Go as part of the standard library. For more information please read [range over func proposal][rangefunc-proposal] as well as predestining article on [coroutines in Go][coro-article] by Russ Cox, which the experimental `iter` package is built ontop.

Adjusting the `iterate()` to be `iter` compatible is easy as pie:
{{<func file="./src/async/async.go" name="iter_02" next="iterate" lang="go" class="focus" hl_lines="1 5 7-13">}}

Enabling that experiment allows us to do an amazing thing -- to iterate over the results of concurrently processed elements of a _sequence_ in the regular `for` loop!
```go
// Assuming the standard library supports iterators.
seq := slices.Sequence([]int{1, 2, 3})

// Output: [1, 4, 9]
for a, b := range iterate(ctx, nil, 0, seq, square) {
	fmt.Println(a, b)
}
```

## Conclusion

I wish this was a part of the Go standard library.

Initially I wanted to have this first sentence to be the only content for this conclusion section, but probably at least a few words still should be said why. I believe such general purpose utilities can be much better conveyed and accepted by projects if majority of the community agrees on how the utilities designed and built. Of course we can have some libraries solving similar problems, but in my opinion the more different libraries we have the more disagreement in the community we _may_ get about what, when and how to use them. For some cases there is nothing wrong to have widely different approaches and implementations, but for some cases it can also mean not having a complete solution at all. Very often libraries initially get born as a much more specific solution than needed to be widely adopted, and to be really general purpose solution the design, API and then implementation should be well discussed _way before_ the actual work takes place. This is how OSS foundations solve similar problems or Go team in case of Go. Having something for such concurrent/asynchronous processing feels to be a natural evolvement after getting generic slices package and later coroutines and iterators.

## References

- [Parallel Collections in Scala][scala-par]
- [When To Use Generics][when-generics]
- [Rangefunc Experiment][rangefunc-wiki]



[context]: https://pkg.go.dev/context#Context
[coro-article]: https://research.swtch.com/coro
[coro-source]: https://github.com/golang/go/blob/20f052c83c380b8b3700b7aca93017178a692d78/src/runtime/coro.go
[errgroup]: https://pkg.go.dev/golang.org/x/sync/errgroup
[genny]: https://github.com/cheekybits/genny 
[interfaces]: https://research.swtch.com/interfaces
[iter-proposal]: https://github.com/golang/go/issues/61897
[iter-seq]: https://github.com/golang/go/blob/20f052c83c380b8b3700b7aca93017178a692d78/src/iter/iter.go#L19-L27
[method-expressions]: https://go.dev/ref/spec#Method_expressions
[rangefunc-proposal]: https://github.com/golang/go/discussions/56413
[rangefunc-wiki]: https://go.dev/wiki/RangefuncExperiment
[scala-par]: https://docs.scala-lang.org/overviews/parallel-collections/overview.html
[semaphore]: https://en.wikipedia.org/wiki/Semaphore_(programming)
[sync-pool]: https://pkg.go.dev/sync#Pool
[when-generics]: https://go.dev/blog/when-generics
