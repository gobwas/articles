---
title: "Configuration in Go"
date: 2019-12-04T16:13:28+03:00
draft: false
image: configuration-in-go.jpeg
imageWidth: 600
description: "After working with Go for more than five years I have formed a strong opinion for a certain way of configuring programs. In this article I will cover the approach and share a small library, which is an implementation of these ideas."
---
{{< img src="configuration-in-go.jpeg" width="600" alt="Gopher holding big flag and standing near retro computer" >}}

## Introduction

Hi everyone! After working with Go for more than five years I have formed a
strong opinion for a certain way of configuring programs. In this article I
will cover the approach and share a small library, which is an implementation
of these ideas.

It is worth mentioning, that the article is written based on my personal
experience, so it is quite subjective and does not claim to be the ultimate
truth. However, I hope it can be useful to the community and will help to
reduce time spent on such a trivial task.

## What is it all about?

In general, the configuration, in my opinion, is all about defining parameters
and receiving their values from outside of our program during a runtime. It may
be command-line parameters or arguments, environment variables, configuration
files stored on the disk or somewhere on the network, a database table and so
on.

Since Go's type system is strong and static, we'd like to define and receive
values for these parameters with knowledge of their type.

There are lots of already existing open source libraries or even frameworks
solving such tasks. Most of them represent their own vision of how to do it.

But as stated above, I would like to cover, perhaps, less widely used approach
to a program configuration. Especially since this approach seems much simpler.

## The `flag` package

Yes, this is not a joke and I really would like to draw your attention to this
well-known package of the Go's standard library.

At the first glance, `flag` provides command-line arguments parsing and nothing
more. But this package can also be used as an **interface** for program
**parameters definition**. And in the context of the discussing approach,
`flag` is used primarily in this way.

As mentioned above, we would like to have typed parameters. `flag` package
supports the most basic types – `flag.String()`, `flag.Int()` and even
`flag.Duration()`. For more complex types, such as `[]string` or `time.Time`
there is `flag.Value` interface which allows you to implement the parsing of
the value from its *string representation*.

For example, the `time.Time` parameter can be implemented like this:

```go
// TimeValue is an implementation of flag.Value interface.
type TimeValue struct {
	P      *time.Time
	Layout string
}

func (t *TimeValue) Set(s string) error {
	v, err := time.Parse(t.Layout, s)
	if err == nil {
		(*t.P) = v
	}
	return err
}

func (t *TimeValue) String() string {
	return t.P.Format(t.Layout)
}
```

An important property of the package is that it lives in the standard library.
Thus, `flag` is the **standard way of a program configuration**. So, the
probability of `flag` being reused between different projects or libraries is
higher than for the other configuration libraries or frameworks.

## Why is `flag` not being used?

In my opinion, other libraries exist and used for two main reasons:
 - Parameters are required to be read not only from the command-line
 - There is a need for parameters to be structured

The situation related to reading parameters is rather clear (for example,
reading from files; nevertheless, we will talk about this later), yet it is
necessary to say a few words about structured parameters here.

Often you can meet a particular way of defining the program parameters as
a structure, i.e. where fields could be other structures and so on:

```go
type AppConfig struct {
	Port int
	Database struct {
		Endpoint string
		Timeout  time.Duration
	}
	...
}
```

This is probably the main reason why there are configuration libraries and
frameworks that allow you to work with parameters in such a way.

I think, `flag` should not provide the possibilities of a structured
configuration. This can be achieved with a few lines of code (or with the
`flagutil` library, which I will mention later).

Moreover, if you think about it, use of such a structure leads to hard coupling
between all used components of the program.

## Structured configuration

The idea is to define parameters independently and regardless of the program
structure and as close as possible to the place where they are used – that is,
directly on the package level.

Suppose we have an implementation of the client to some service (database, API
or anything else), which is called `yoogle`:

```go
package yoogle

type Config struct {
	Endpoint string
	Timeout  time.Duration
}

func New(c *Config) *Client {
	// ...
}
```

In order to fill the `yoogle.Config` we need a function that registers the
fields of the structure within the received `*flag.FlagSet`.

This function can be defined in `yoogle` package or within `yooglecfg` package
(in case of a third-party library, we could code such function somewhere else):

```go
package yooglecfg

import (
	"flag"

	"app/yoogle"
)

func Export(flag *flag.FlagSet) *yoogle.Config {
	var c yoogle.Config
	flag.StringVar(&c.Endpoint,
		"endpoint", "https://example.com",
		"endpoint for our API",
	)
	flag.DurationVar(&c.Timeout,
		"timeout", time.Second,
		"timeout for operations",
	)
	return &c
}
```

In order to get rid of the `flag` package dependency we can define an interface
with the required methods from `flag.FlagSet`:

```go
package yooglecfg
	
import "app/yoogle"

type FlagSet interface {
	StringVar(p *string, name, value, desc string)
}

func Export(flag FlagSet) *yoogle.Config {
	var c yoogle.Config
	flag.StringVar(&c.Endpoint,
		"endpoint", "https://example.com",
		"endpoint for our API",
	)
	return &c
}
```

If configuration depends on certain values (for example, a parameter specifies
an algorithm), the `yooglecfg.Export()` function could return a factory
function, which must be called **after** parsing of all parameters values:

```go
package yooglecfg
	
import "app/yoogle"

type FlagSet interface {
	StringVar(p *string, name, value, desc string)
}

func Export(flag FlagSet) func() *yoogle.Config {
	var algorithm string
	flag.StringVar(&algorithm,
		"algorithm", "quick",
		"algorithm used to do something",
	)

	var c yoogle.Config
	return func() *yoogle.Config {
		switch algorithm {
		case "quick":
			c.Impl = quick.New()
		case "merge":
			c.Impl = merge.New()
		case "bubble":
			panic(...)
		}
		return c
	}
}
```

> Such export functions allow you to define package parameters regardless of
> their values parsing method or structure of the program configuration.

## [github.com/gobwas/flagutil][flagutil]

Now we've fixed that highly coupled configuration structure and made our
parameters independent. But it is still not yet clear how to collect them all
together and receive their values. 

The `flagutil` package was written to solve exactly this problem.

### Collecting parameters together

All parameters of program packages or third-party libraries receive their
unique prefix and are collected at the `main` package:

```go
package main

import (
	"flag"

	"app/yoogle"
	"app/yooglecfg"
 
	"github.com/gobwas/flagutil"
)

func main() {
	flags := flag.NewFlagSet("my-app", flag.ExitOnError)

	var port int
	flag.IntVar(&port, 
		"port", 4050,
		"port to bind to",
	)

	var config *yoogle.Config
	flagutil.Subset(flags, "yoogle", func(flags *flag.FlagSet) {
		config = yooglecfg.Export(flags)
	})
}
```

The `flagutil.Subset()` function does a simple thing: it adds a prefix
(`"yoogle"`) to all parameters defined within a callback.

In this case program execution may look like this:

```bash
app -port 4050 -yoogle.endpoint https://example.com -yoogle.timeout 10s
```

### Getting parameter values

All parameters defined within `flag.FlagSet` contain an implementation of
`flag.Value`, which has a `Set(string) error` method. That is, every parameter
provides ability to set *string representation* of its value.

All we have to do now is to parse key-value pairs from any source and call
`flag.Set(key, value)`. 

> Note, with that you don't even have to use the command-line syntax of the
> `flag` package. You can parse arguments in any way, for example, as [posix
> program arguments][posix]. 

```go
package main

func main() {
	flags := flag.NewFlagSet("my-app", flag.ExitOnError)

	// ...

	flags.String(
		"config", "/etc/app/config.json", 
		"path to configuration file",
	)

	flagutil.Parse(flags,
		// First, use posix arguments syntax instead of `flag`.
		// Just to illustrate that it is possible.
		flagutil.WithParser(&pargs.Parser{
			Args: os.Args[1:],
		}),	

		// Then lookup for "config" flag value and try to
		// parse its value as a json configuration file.
		flagutil.WithParser(&file.Parser{
			PathFlag: "config",
			Syntax:   &json.Syntax{},
		}),
	)
}
```

## Conclusion

Of course, I won't be the first person who talks about such approach. Many of
the ideas described above were already in use a few years ago, when I was
working at MailRu. 

So, in order to simplify the program configuration and not to spend time
learning (or even writing) the next configuration framework the following is
suggested:

- Use 'flag' as an **interface for program parameters definition** 
- Export parameters of each package separately, without knowing the structure
  and method of subsequent obtaining of values
- Define the way values are parsed, parameter prefixes, and configuration
  structure in `main` 

The creation of the `flagutil` library was greatly inspired by the
[peterburgon/ff][ff] library – and I wouldn't write `flagutil` if there hadn't
been certain design differences.

Thanks for your attention!

## References

- [golang.org/pkg/flag][flag]
- [github.com/gobwas/flagutil][flagutil]

[posix]:    https://www.gnu.org/software/libc/manual/html_node/Argument-Syntax.html
[flag]:     https://golang.org/pkg/flag/
[flagutil]: https://github.com/gobwas/flagutil
[ff]:       https://github.com/peterbourgon/ff
