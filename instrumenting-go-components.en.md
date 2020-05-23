---
title: "Instrumenting Go Components"
date: 2020-05-23T10:58:54+03:00
draft: false
description: ""
---

## Introduction

In this article I will share my thoughts on how Go programms should be
instrumented in a clean and flexible way.

## TL;DR

Logging, metrics collection and other logic that is not related to main purpose
of some piece of code, must not appear there. Instead, define some _trace
points_ that might be useful to be instrumented for a user of your code.

That is, logging, metrics collection and others are all subsets of **tracing**.

## The problem

Lets assume that we have some package called `lib` and some `lib.Client`
structure which pings its underlying connection every time while making some
request.

```go
package lib

type Client struct {
	conn net.Conn
}

func (c *Client) Request(ctx context.Context) error {
	if err := c.ping(ctx); err != nil {
		return err
	}
	// Some client logic.
}

func (c *Client) ping(ctx context.Context) error {
	return doPing(ctx, c.conn)
}
```

What if we need to write some logs right before and after ping happens? One way
is to inject some logger (or logger interface) into the `Client`:

```go
package lib

type Client struct {
	Logger Logger

	conn net.Conn
}

func (c *Client) ping(ctx context.Context) (err error) {
	c.Logger.Info("ping started")
	err = doPing(ctx, c.conn)
	c.Logger.Info("ping done (err is %v)", err)
	return
}
```

This happens when we also decide to add metrics calculation into our `Client`:

```go
package lib

type Client struct {
	Logger  Logger
	Metrics Registry

	conn net.Conn
}

func (c *Client) ping(ctx context.Context) (err error) {
	start := time.Now()
	c.Logger.Info("ping started")

	err = doPing(ctx, c.conn)

	c.Logger.Info("ping done (err is %v)", err)
	metric := c.Metrics.Get("ping_latency")
	metric.Send(time.Since(start))

	return err
}
```

If we continue to add instrumentation methods to our `Client`, we'll realise
soon that most of its code is related to instrumentation, and not to the main
functionality of the `Client` (which was just a single line with `doPing()`
call).

Number of non-coherent (non-related to the main purpose of our `Client`) lines
of code is just a first problem with that approach.

What if during operations of your program you realise that the metric name, for
example, is wrong and you should rename it? Or you must use some different
library for logging? 

With approach above you will go to the implementation of `Client` (and other
similar components as well) and change it.

It does violate the [SRP
principle](https://en.wikipedia.org/wiki/Single-responsibility_principle).

What if you share your code across multiple programms? What if you even don't
control the consumers of your code at all (and to be honest I suggest to treat
_every_ package like it is reused by unkonwn number of programms, even if in
reality its only one that is yours).

All of this questions point to a design mistake that we made earlier: 

> We must not assume which instrumentation methods user will want to use with
> our component.

## Solution

In my opinion the right way to do it is to define _trace points_ (aka _hooks_)
which user of your code can then initialize with some function (aka _probe_)
during runtime. 

This still will add some extra lines to our code for sure, but will also bring
a flexibility to users to measure our componenet's runtime with any appropriate
method.

Such method is used for example by the standard library's
[`httptrace`](https://golang.org/pkg/net/http/httptrace/) package. 

Lets provide almost the same facility, but with one change. Instead of
providing `OnPingStart()` and `OnPingDone()` _hooks_, lets introduce a single
`OnPing()` hook which will be called right before ping, and will return a
callback which will be called right after ping. This way we can store some
variables in closure, to access them when ping completes (e.g. to calculate
ping latency for example).

Lets look how our `Client` will change with this approach:

```go
package lib

type Client struct {
	OnPing func() func(error)
	conn net.Conn
}

func (c *Client) ping(ctx context.Context) (err error) {
	done := c.OnPing()
	err = doPing(ctx, c.conn)
	done(err)
	return
}
```

Looks neat, but only unless we realise that both `OnPing` hook and callback it
returns might be nil:

```go
func (c *Client) ping(ctx context.Context) (err error) {
	var done func(error)
	if fn := c.OnPing(); fn != nil {
		done = c.OnPing()
	}
	err = doPing(ctx, c.conn)
	if done != nil {
		done(err)
	}
	return
}
```

Now it is correct and is till good at flexibility and SRP principle, but not so
good in code simplicity.

Before making code more simple lets cover another problem we have yet with
current hooks implementation. 

How users are going to setup _multiple_ probes? So, the already mentioned
`httptrace` package has
[`ClientTrace.compose()`](https://github.com/golang/go/blob/b68fa57c599720d33a2d735782969ce95eabf794/src/net/http/httptrace/trace.go#L175)
method that merges two trace structs in third one (it is unexported since
`httptrace.ClientTrace` is used only within contexts, and is not able to be set
as some struct's field). So calling some probe function from resulting trace
will call inside appropriate probes from previous traces (if they were set).

So lets first try to do the same manually (and without `reflect`). To do this,
we move the `OnPing` hook from `Client` to separate structure `ClientTrace`:

```go
package lib

type Client struct {
	Trace ClientTrace
	conn net.Conn
}

type ClientTrace struct {
	OnPing func() func(error)
}
```

And composing two `ClientTrace`'s into one will be as following:

```go
func (a ClientTrace) Compose(b ClientTrace) (c ClientTrace) {
	switch {
	case a.OnPing == nil:
		c.OnPing = b.OnPing
	case b.OnPing == nil:
		c.OnPing = a.OnPing
	default:
		c.OnPing = func() func(error) {
			doneA := a.OnPing()
			doneB := b.OnPing() 
			return func(err error) {
				if doneA != nil {
					doneA(err)
				}
				if doneB != nil {
					doneB(err)
				}
			}
		}
	}
	return c
}
```

Pretty much code for single hook, right? Lets move forward for now and come
back to this later.

One thing that we also may provide to users of our component is context based
tracing. That is, the one that is exactly the same in `httptrace` package –
bring user ability to setup not (only) the structure level tracing hooks, but
also associate hooks with `context.Context` object on some conditions.


```go

type clientTraceContextKey struct{}

func ClientTrace(ctx context.Context) ClientTrace {
	t, _ := ctx.Value(clientTraceContextKey{})
	return t
}

func WithClientTrace(ctx context.Context, t ClientTrace) context.Context {
	prev := ContextClientTrace(ctx)
	return context.WithValue(ctx,
		clientTraceContextKey{},
		prev.Compose(t),
	)
}
```

Huh. Looks like now its almost done and we are ready to bring all the best of
the tracing facilities for users of our component.

But isn't it really tedious to write all that code for every structure we want
to instrument? Of course you can write some vim's macros for this (which I used
to do before actually), but lets look at alternatives.

The good news is that merging hooks and checking them to be nil, as well as
context specific functions are all quite patterned, so we can generate Go code
for them without macros or reflection based helpers.

## github.com/gobwas/gtrace

**gtrace** suggests you to use structures (tagged with `//gtrace:gen`) holding
all _hooks_ related to your component and generates helper functions around
them so that you can merge such structures and call the _hooks_ without any
checks for `nil`. It also can generate context aware helpers to call _hooks_
associated with context.

Example of generated code [is here](examples/pinger/main_gtrace.go).

So we can drop all the boilerplate code written above and leave just this:

```go
package lib

//go:generate gtrace

type Client struct {
	Trace ClientTrace
	conn net.Conn
}

//gtrace:gen
//gtrace:set context
type ClientTrace struct {
	OnPing func() func(error)
}
```

After run of `go generate` we will be able to use the generated **unexported**
versions of trace hooks like so:

```go
func (c *Client) ping(ctx context.Context) (err error) {
	done := c.Trace.onPing(ctx)
	err = doPing(ctx, c.conn)
	done(err)
	return
}
```

That's it! **gtrace** takes care of boilerplate code patterns and allows you to
focus on _trace points_ in your code you wish to make able to be instrumented,
and API of the hook functions.

Thank you for reading!

## References

- [github.com/gobwas/gtrace](https://github.com/gobwas/gtrace)
- [Minimalistic C libraries](https://nullprogram.com/blog/2018/06/10/) by Chris Wellons
