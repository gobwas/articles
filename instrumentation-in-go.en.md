---
title: "Instrumentation in Go"
date: 2020-05-23T10:58:54+03:00
draft: false
image: instrumentation-in-go.jpg
imageWidth: 600
description: "My thoughts on how Go programs should be instrumented in a clean and flexible way."
---
{{< img src="instrumentation-in-go.jpg" width="600" alt="Gopher looking at reader through magnifying glass" >}}

## Introduction

In this article I will share my thoughts on how Go programs should be
instrumented in a clean and flexible way.

## TL;DR

Logging, metrics collection and anything that is not related to main
functionality of some code, must not appear within that code. Instead, define
some _trace points_ of the code that can be instrumented by a user.

That is, logging and metrics collection are subsets of the **tracing**.

There is a code generation tool called [gtrace][gtrace] that generates
boilerplate code for the tracing.

## The problem

Let's assume that we have some package called `lib` and some `lib.Client`
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
	// Some logic here.
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

If we continue to add instrumentation methods to our `Client`, we'll realize
soon that most of its code is related to instrumentation and not to the main
functionality of the `Client` (which was just a single line with `doPing()`
call).

Number of non-coherent (non-related to the main purpose of our `Client`) lines
of code is just a first problem with that approach.

What if during operations of your program you realize that the metric name, for
example, is wrong and you should rename it? Or you must use some different
library for logging? 

With approach above you will need to go to the implementation of `Client` (and
other similar components as well) and change it.

This means that you are going to change code every time when something that is
not related to component's _main functionality_ changes. In other words such
design does violate the [SRP principle][wikipedia:srp].

What if you share your code across multiple programs? What if you even don't
control the consumers of your code at all (and to be honest I suggest treating
_every_ package as it reused by unknown number of programs, even if in reality
it is only one that yours).

All of these questions point to a design mistake that we made earlier:

> We must not assume which instrumentation methods user will want to use with
> our component.

## Solution

In my opinion the right way to do it is to define _trace points_ (aka _hooks_)
which user of your code can then initialize with some function (aka _probe_)
during runtime. 

This still will add some extra code lines for sure, but will also bring the
flexibility to users to measure our component's runtime with any appropriate
method.

Such approach is used for example by the standard library's
[`httptrace`][go:httptrace] package. 

Let's provide almost the same mechanics, but with one change. Instead of
providing `OnPingStart()` and `OnPingDone()` hooks, let's introduce a single
`OnPing()` hook which will be called right before ping and will return a
callback, which will be called right after ping. This way we can store some
variables in closure to access them when ping completes (e.g. to calculate ping
latency).

Let's look how our `Client` will change with this approach:

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

Looks neat, but only unless we realize that both `OnPing` hook and callback it
returns might be `nil`:

```go
func (c *Client) ping(ctx context.Context) (err error) {
	var done func(error)
	if fn := c.OnPing; fn != nil {
		done = fn()
	}
	err = doPing(ctx, c.conn)
	if done != nil {
		done(err)
	}
	return
}
```

Now it is correct and is still good at flexibility and SRP principle, but not
so good at code simplicity.

Before making code simpler let's cover another problem we have yet with
current hooks implementation. 

### Composing hooks

How users are going to set up _multiple_ probes? So, the already mentioned
`httptrace` package has [`ClientTrace.compose()`][github:httptrace] method that
merges two trace structs in third one. So every probe function from resulting
trace will call appropriate probes from previous traces (if they were set).

So let's first try to do the same manually (and without `reflect`). To do this,
we move the `OnPing` hook from the `Client` to separate structure
`ClientTrace`:

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
			switch {
			case doneA == nil:
				return doneB
			case doneB == nil:
				return doneA
			default:
				return func(err error) {
					doneA(err)
					doneB(err)
				}
			}
		}
	}
	return c
}
```

Pretty much code for single hook, right? Let's move forward for now and come
back to this later.

Now user can set up or change any instrumentation method independently:

```go
package main

import (
	"log"
	
	"some/path/to/lib"
)

func main() {
	var trace lib.ClientTrace

	// Logging hooks.
	trace = trace.Compose(lib.ClientTrace{
		OnPing: func() func(error) {
			log.Println("ping start")
			return func(err error) {
				log.Println("ping done", err)
			}
		},
	})

	// Some metrics hooks.
	trace = trace.Compose(lib.ClientTrace{
		OnPing: func() func(error) {
			start := time.Now()
			return func(err error) {
				metric := stats.Get("ping_latency")
				metric.Send(time.Since(start))
			}
		},
	})

	c := lib.Client{
		Trace: trace,
	}
}
```

### Context based tracing

One thing that we also may provide users is context based tracing. That is, the
one that is exactly the same as in `httptrace` package – the ability to
associate hooks with `context.Context` passed to the `Client.Request()`:

```go
package lib

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

Huh. Looks like now it's almost done, and we are ready to bring all the best of
the tracing facilities for users of our component.

But isn't it really tedious to write all that code for every structure we want
to instrument? Of course, you can write some Vim's macros for this (actually I
used to do them before), but let's look at alternatives.

The good news is that merging hooks and checking for `nil`, as well as context
specific functions are all quite patterned, so we can [generate][go:generate]
Go code for them without macros or reflection.

## github.com/gobwas/gtrace

**gtrace** is a command line tool that generates Go code for the tracing
approach discussed above. It suggests you to use structures (tagged with
`//gtrace:gen`) with hook fields and generates helper functions around them, so
that you can merge such structures and call the hooks without any checks for
`nil`. It also can generate context aware helpers to call hooks associated with
context.

Example of generated code [is here][github:gtrace:example].

So we can drop all the boilerplate code written above and leave just this:

```go
package lib

//go:generate gtrace

//gtrace:gen
//gtrace:set context
type ClientTrace struct {
	OnPing func() func(error)
}

type Client struct {
	Trace ClientTrace
	conn net.Conn
}

func (c *Client) ping(ctx context.Context) (err error) {
	done := c.Trace.onPing(ctx)
	err = doPing(ctx, c.conn)
	done(err)
	return
}
```

After run of `go generate` we will be able to use the generated **non-exported**
versions of trace hooks we defined in `ClientTrace`.

That's it! **gtrace** takes care of boilerplate code and allows you to focus on
_trace points_ you wish to make able to be instrumented by users.

Thank you for reading!

## References

- [github.com/gobwas/gtrace][gtrace]
- [Minimalistic C libraries](https://nullprogram.com/blog/2018/06/10/) by Chris
  Wellons. I read this article long time ago and was inspired by that clear
  thoughts on how libraries could be organized.

[gtrace]:                https://github.com/gobwas/gtrace
[wikipedia:srp]:         https://en.wikipedia.org/wiki/Single-responsibility_principle
[go:httptrace]:          https://golang.org/pkg/net/http/httptrace
[go:generate]:           https://blog.golang.org/generate
[github:httptrace]:      https://github.com/golang/go/blob/b68fa57c599720d33a2d735782969ce95eabf794/src/net/http/httptrace/trace.go#L175
[github:gtrace:example]: https://github.com/gobwas/gtrace/blob/216866e1680bb30fb108f26ee926c471f0920239/examples/pinger/main_gtrace.go
