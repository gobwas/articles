---
title: "Incident pipelines with Spooler"
date: 2026-11-18T14:48:00+00:00
draft: false
image: pagedog.jpg
imageWidth: 400
description: "How an incident investigation can be built using durable message queues."
---
{{< img src="pagedog.jpg" width="400" alt="A fluffy dog ready to investigate incidents" >}}

Hi there! As some of you might know, I recently launched
[spooler.sh](https://spooler.sh) -- a simple, managed message queue.

In this article, I'd like to show you how simple it can be to build
pipelines with clear consistency and durability properties.

The project we're going to build is small, though very much in line with the
trends of today: LLMs are involved.

~We'll build a distributed agentic AI swarm that would perform multi-step
actions with limited or no human supervision to investigate...~

In fact, we'll build a simple program that spins up and shuts down workers
depending on a queue backlog. The workers would pull incidents from a queue and
ask an LLM to investigate them.

## Meet the fluffy one

Building the tool I'm going to describe in this article took a couple of days,
but then I realized it had taken a pretty nice shape and might be worth sharing
with the world as a tool on its own, rather than just a showcase example.

So I called it [Pagedog](https://pagedog.dev) -- a funny name rhyming with
"watchdog" but for pages coming from a pager. It is written in Go, and so are
the snippets below, though nothing in the design depends on the language.

> A page, for those who have never been on call, is the alert that wakes up
> whoever is on call at the moment: something monitoring decided a human must
> see. The pager is whatever sends it, PagerDuty, incident.io and Alertmanager
> alike.

## Design

On one side, we have a webhook handler for your observability stack -- it
receives pages from [Prometheus Alertmanager][alertmanager],
[incident.io][incident-io], [PagerDuty][pagerduty], or whatever else you want
to integrate with.

In many implementations I've seen, including systems I've built myself and
approaches I've discussed with other engineers, the incoming page is usually
stored somewhere before further processing. On the other hand, some of the
implementations do not store it at all and simply process it straight away.

The risk of the latter approach is that you can lose the page, possibly along
with wasted compute and LLM tokens. Depending on your observability stack and
its retry behavior, the page might never arrive again, or arrive too late.

So what we're going to do in this article is store the page in a message
queue -- specifically, a Spooler queue (what a nice coincidence!). This gives
us useful guarantees around durability and delivery.

Once we've received the page and stored it in the queue as an incident, that's
already one half of our project. The second half is to pull it back from the
queue and feed it to an LLM of our choice -- we ask the LLM to analyze the
incident and investigate what might have gone wrong.

Of course, we also need to give it access to some of our infrastructure
observability tools. Otherwise, it won't be able to do much beyond rephrasing
the incident's description. So we give the LLM access to our metrics, logs, and
whatever else we want to provide.

```text {class="diagram"}
 Alertmanager   |   webhook  ┌────────────────┐      ┌────────────────┐
 incident.io    |  --------> │ pagedog server │ ───▶ │ incident queue │
 ...            |            └────────────────┘      └────────────────┘
                |                                            │
                |                                            ▼
 MCPs           |            +----------------+      ┌────────────────┐
 Prometheus     |  <-------> |      LLMs      | <--> │ pagedog worker │
 ...            |            +----------------+      └────────────────┘
                |                                            │
                |                                            ▼
 Slack          |            ┌────────────────┐      ┌────────────────┐
 incident.io    |  ◀──────── │ pagedog server │ ◀─── │  report queue  │
 ...            |            └────────────────┘      └────────────────┘
```

### The promised properties

Here is what this design gives us; the sections below deliver them one by one:

- a page is never lost once the pager has heard `200 OK` from us
- a page sent twice comes down to one investigation
- a worker that crashes mid-investigation is replaced, and its incident
  redelivered
- a report, once written, is never lost on its way to the next stage
- every destination hears of every report, at least once

## Webhook

The webhook server is the plainest part here. It serves one route per
source, e.g. `/webhook/alertmanager`, and the handler is anything that can turn
an HTTP request into incidents:

```go
type IncidentSource interface {
	Handle(*http.Request) ([]pagedog.Incident, error)
}
```

Each source first checks the page really comes from its pager: a bearer
`Authorization` token, a signature over the body or anything else the source's
spec provides.

Then it reads the incidents out of the payload and passes them up the stack to
be sent into the `incidents` queue: our system's "edge" queue holding the
input. Only when every send has succeeded does it answer `200 OK`, confirming
the input is in and is _durable_:

```go
incs, err := src.Handle(req)
if errors.Is(err, ErrForbidden) {
	http.Error(res, "", http.StatusForbidden)
	return
}
if err != nil {
	http.Error(res, "failed parsing body", http.StatusBadRequest)
	return
}
for _, inc := range incs {
	if err := s.enqueue(req.Context(), inc); err != nil {
		// The source may retry the whole notification: the queue
		// deduplicates what was sent already.
		http.Error(res, "failed sending job", http.StatusBadGateway)
		return
	}
}
// 200 OK.
```

That ordering is the entire durability story of this part: the pager hears "got
it" only once the incident is on the queue, and retries otherwise. The status
codes are how we tell it: pagers retry on a `5xx` and give up on a `4xx`, so a
bad token or a payload we can't parse gets a `403` or a `400`, which a retry
wouldn't fix, and a failed send gets a `502`, so that the pager tries again.

This is where the first Spooler property shows up: deduplication. Every
incident has a key the source derives from the page (in case of Alertmanager,
the alert's fingerprint and the time it started firing) and the queue is told
to dedup by it:

```go
incidents := spooler.Queue[pagedog.Incident]{
	Client: client,
	Spool:  "default",
	Queue:  "incidents",
	Dedup: func(inc pagedog.Incident) string {
		return inc.Key
	},
}
```

A second send with the same key within the queue's dedup window -- a setting
of the queue, seconds to minutes long -- is not a second message: a repeated
page or a retried one comes down to one investigation. A page repeated after
the window, as Alertmanager does for an alert still firing hours later, is a
new message and a fresh investigation, and that's intended: by then the
picture may well have changed.

> The `Spool` field above is, in SQL terms, a database: an isolation unit
> holding its queues, with its own write-ahead log, I/O and limits, so one
> spool never slows down another. Every account starts with one named
> `default`, and for Pagedog that one is enough.

## Workers

Once the incident is on the queue, we need something that pumps it further
down the pipeline: we need to spin up workers to do so.

In Pagedog, workers are, deliberately, a rather vague concept: the actual
execution mechanism is hidden behind a runtime interface:

```go
type Runtime interface {
	Stats(context.Context) (Stats, error)
	Launch(context.Context) (Worker, error)
	Shutdown(context.Context, Worker) error
}
```

A runtime implementation can range from goroutines running inside the process
to separate containers or pods running on your platform, or functions for a
serverless one. The interface allows us to keep these execution details
separate from the rest of the system and plug in any compute.

### How much to spin?

To understand when it needs to spin up another worker, Pagedog checks a few
simple metrics from the Spooler queue: the number of currently _visible_
messages in the `incidents` queue -- that is, waiting to be received -- and the
number of messages already being processed by workers -- that is, messages in
_pending_ state.

The sum gives us an estimate of how many workers we would ideally want to run
to process all the available incidents. The number of running workers is
bounded by configuration:

```go
func desired(q QueueStats, minWorkers, maxWorkers int) int {
	want := int(q.Pending + q.Visible)
	want = min(want, maxWorkers)
	want = max(want, minWorkers)
	return want
}
```

The `desired()` is called from the runtime reconciler loop: periodically, it
compares the number of running workers with the desired one, and launches or
shuts down the difference. 

## Sniffing incidents out

The worker itself is very simple. It pulls messages from the `incidents` queue,
where each message represents one incident that needs to be investigated.

For every incident, the worker talks to the configured LLM through its API:

```go
type Model interface {
	Complete(context.Context, CompleteRequest) (CompleteResult, error)
}
type CompleteRequest struct {
	// Messages is the conversation so far: the task text, model replies
	// and tool output.
	Messages []Message

	// Tools holds the specs of the tools the model may call.
	Tools []ToolSpec
}
```

> The LLM API is stateless, so every call carries the whole conversation. Along
> with it we send the specs of the tools: a name, a description and the schema
> of the input. The model answers either with text, or with a request to call
> one of the tools. In the latter case our code runs the tool, appends its
> output to the conversation and asks the model again. This repeats until the
> model answers with text alone -- that is the report. Each round trip is a
> _turn_, and since every turn re-sends the conversation, turns are what the
> investigation costs.

As you can see, we pass the LLM the available tools' specs, letting it use them
in the interest of the investigation -- for example, to fetch logs around the
incident or read the source code involved in the logged activities.

Many systems today have [MCP][mcp] servers that can serve these needs. Pagedog
has a generic MCP tool implementation that connects to those, as well as direct
integrations with [Prometheus][prometheus] and [Loki][loki] as the built-in
tools.

> MCP, the Model Context Protocol, is a protocol between a program driving a
> model and a server offering it tools. The server lists its tools, each with a
> name, a description and a JSON schema of its input, and runs a tool when
> asked, returning text. That listing is the same shape as the `ToolSpec`
> below, which is why Pagedog can hand an MCP server's tools to the model
> without knowing what they do.

The built-in tools are read-only queries, while for MCP the model gets only the
tools you list -- the list is required, so nothing is granted by omission. The
model's reach into your infrastructure is exactly as far as you let it.

Adding a new tool requires implementing the `Tool` interface and wiring it up
in the configuration:

```go
// Tool is something the model may call.
type Tool interface {
	// Spec describes the tool to the model.
	Spec() ToolSpec

	// Call runs the tool on the model's input, which the model shaped after
	// Spec's schema. The text goes back to the model; so does an error.
	Call(ctx context.Context, input []byte) (string, error)
}
```

`ToolSpec` defines the structure of the input, so the model knows what the tool
expects.

The initial message in the converstaion -- the worker's task for the model --
is a text rendered from a template with the incident's fields. The default one
is short:

```text
You are the on-call engineer. Investigate the incident below and write a short
report in Markdown: what happened, the evidence you found in the metrics and
the logs, the likely cause, and what to do next.

The incident started at {{.StartedAt}}; it is now {{.Now}}. Look at the window
from fifteen minutes before the start until now, and give the tools times in
RFC 3339.

The incident, as its source reported it:
---
{{.Raw}}
---
```

What comes back, once the model stops calling tools, is Markdown in the shape
the task asks for: what happened, the evidence, the likely cause and what to do
next.

Depending on how deep you want the investigation to go, all of this takes a
while. And that is where the second Spooler property comes into play.

### Leases and Retries

Spooler leases messages to consumers. This means that when a worker receives a
message, it doesn't immediately remove it from the queue: instead, the worker
holds a _lease_ of that message while processing it. The lease can expire, in
which case the message is either re-delivered to another worker or enters the
failed state if the retry budget is exhausted (the budget is the queue's
`maxRetries` setting).

The receive loop is as simple as:

```go
for ctx.Err() == nil {
	m, ok, err := client.Recv(ctx, spooler.RecvRequest{
		Spool:       spool,
		Queue:       queue,
		WaitSeconds: new(1), // Wait up to one second for a message.
	})
	if err != nil {
		return err
	}
	if !ok {
		// No visible messages within the second: repeat.
		continue
	}
	process(ctx, m)
}
```

Since an LLM investigation can take some time, the worker periodically renews
the message lease so it doesn't lose it. If the worker crashes in the middle
of an investigation, the lease will eventually expire, and another worker can
pick up the same message and retry.

A failed renewal cancels the investigation's context: the message is someone
else's now, and the ongoing investigation should be stopped:

```go
for {
	select {
	case <-ctx.Done():
		return
	case <-time.After(time.Until(expiresAt) / 3 * 2):
		// Renew at two thirds of the lease, leaving a third as a
		// margin for a slow request.
	}
	res, err := client.Renew(ctx, spooler.RenewRequest{
		Spool: spool,
		Lease: lease.token,
	})
	if err != nil {
		cancel() // Cancel investigation context.
		break
	}
	expiresAt = res.LeaseExpiresAt
}
```

Once the investigation is finished, the worker settles the leased message in
one of the ways:
- acknowledges (acks) it on success, which removes it from the queue
- negatively acknowledges it (nacks) to spend one retry and have it redelivered
- releases it to return it to the queue *without* spending a retry (when the
  worker shuts down)
- fails it to put it in the failed state until a human intervenes (encoding
  issues, version mismatch and so on)
- discards it to delete it entirely

In code, the investigation returns the settlement it wants -- the
`ack-and-send` we'll get to in a bit. In case of a failure, the settlement is
decided by the error. `res` is the received message, and `sub` is a context
that survives a shutdown just long enough to settle:

```go
settle, err := investigate(ctx, item)
switch {
case errors.Is(err, ErrDiscard):
	settle = r.discard

case errors.Is(err, ErrFail):
	settle = r.fail

case err != nil && ctx.Err() != nil:
	// A shutdown or a lost lease is not the message's fault: a release
	// spends no retry.
	settle = r.release

case err != nil:
	settle = r.nack
}
_ = settle(sub, res.Lease)
```

## Reports

Once the model finishes the investigation, we have a report. Now we need to
decide what to do with it. Pagedog stores the report outside the message queue:
a message queue isn't really intended to serve as permanent storage for
documents. Its contents are dynamic, and often queues have limits on how much
data can be stored in a single message, Spooler included.

So we introduce another component into our stack, one the diagram above
doesn't show, sitting next to the worker -- a report store:

```go
// ReportStore persists incident reports that can be retrieved later.
type ReportStore interface {
	// Store keeps the report of the incident under its key and returns its
	// URI. The same incident stored again overrides anything stored before.
	Store(ctx context.Context, inc Incident, r Report) (*url.URL, error)

	// Load returns the report Store() kept at u.
	Load(ctx context.Context, u *url.URL) (Report, error)
}
```

Currently Pagedog has two report store backends: the local filesystem (if you
wish to run it all on just one machine), and GitHub issues, where every report
becomes an issue in a repository of yours, and the same incident reported again
updates its issue.

As with all the interfaces before, extending this one with something like S3 or
another distributed storage system should be straightforward.

Once the report is stored, we put a new message into another Spooler queue --
`reports`, for notification processing. The message contains a reference to the
stored report rather than the report itself.

And this brings us to one of my favorite parts of the design.

## Atomic `ack-and-send`

This is the third, and a particularly useful Spooler property, the one I'd like
to draw your attention to: it is what makes building pipelines easy.

Spooler provides an API called `ack-and-send`. It acknowledges a message in one
queue and sends a new message to another queue in a single atomic operation.

In our case, we acknowledge the original incident message in the `incidents`
queue and send a new message containing the report reference to the `reports`
queue. If the operation fails, the incident message isn't acknowledged, and the
report message isn't added. If it succeeds, both changes are committed.

> Why atomic matters? Without it, we would have to pick an order of the
> incident ack and report send:
> - Ack first, then send: a crash in between loses the report, and the incident
>   is gone with it
> - Send first, then ack: a crash in between leaves the incident leased, it
>   gets redelivered, and we investigate it again, paying the model twice and
>   sending a second report.
>
> The usual way around is an outbox table or idempotency keys, i.e. more state
> of ours to keep. `ack-and-send` makes it the queue's problem instead.

This is exactly the kind of consistency property I wanted Spooler to make easy
to use. In Pagedog this is wrapped into a `Pipe` (an abstraction which someday
I may add to the `spooler/x` subpackage for reuse): a thing that receives
messages from one queue as an `A`, turns each into a `B`, and puts the `B` on
another queue. The worker's whole job is that function (trimmed here of the
task templating):

```go
pipe := &spooler.Pipe[pagedog.Incident, pagedog.Report]{
	Client: client,
	Spool:  "default",
	Input:  "incidents",
	Output: "reports",
}
err := pipe.Pump(ctx,
	func(ctx context.Context, inc pagedog.Incident) (r pagedog.Report, err error) {
		res, err := agent.Do(ctx, ai.DoRequest{
			Task:     task(inc), // Task text rendered.
			MaxTurns: maxTurns,  // Limit roundtrips with LLM.
		})
		if err != nil {
			return r, err
		}
		u, err := reportStore.Store(ctx, inc, pagedog.Report{
			Text: res.Output,
		})
		if err != nil {
			return r, err
		}
		return pagedog.Report{
			Incident:  inc,
			ReportURI: u,
		}, nil
	},
)
```

When the callback function succeeds, `Pump` settles the message using
`ack-and-send`.

## Reporting reports

Once the report has been stored and the new message has been placed in the
`reports` queue, there's just one remaining piece: notifying someone about the
investigation results. Currently, this happens inside the main Pagedog process,
in a separate goroutine which drains the `reports` queue in much the same way
as the workers work the `incidents` one.

It leases a message, loads the corresponding report from the store, and tries
to deliver it to all the destinations specified in your configuration. That
might be a Slack channel, a note on a PagerDuty incident, an update to an
incident.io incident, or whatever else you want to integrate with.

Once the report has been successfully delivered to all configured destinations,
Pagedog acknowledges the message.

If any of the destinations refuses, the message is nacked and comes back later
-- to all the destinations again, including those which already got the report.
That is, delivery here is at-least-once per destination: a Slack channel might
see the same report twice, but none of the destinations will miss it. That's
the price of not keeping per-destination state anywhere.

This time, we don't need the atomic `ack-and-send` -- an incident goes through
three states, queued, reported and notified, and this is the last of them,
where a simple `ack` is enough to remove the message from the queue.

And that's it.

## Running Pagedog

All of this is now available as an open-source MIT-licensed project called
[Pagedog](https://pagedog.dev). The repository comes with a `docker compose`
demo under [examples/docker](https://github.com/spoolersh/pagedog/tree/main/examples/docker),
with Prometheus, Alertmanager and Loki around a tiny HTTP server you can break
on purpose, and a single-manifest Kubernetes deployment under
[examples/k8s](https://github.com/spoolersh/pagedog/tree/main/examples/k8s).

Recently, the local in-memory Spooler image became [free to
use](https://docs.spooler.sh/local) in your own infrastructure for your own
purposes, development and production alike, which means you can use Pagedog
without being a hosted Spooler service subscriber.

It keeps the queues in memory, so what a restart loses is what was in flight
during those few minutes -- and a still-firing alert comes back with the next
Alertmanager repeat anyway. The reports live outside the queue, so nothing
already written is lost.

Of course, there is still an option to get stronger persistence or greater
scale if you need one -- you can always sign up for a tier of the hosted
Spooler service, which provides durable and replicated queues.

## Conclusion

So that's Pagedog.

It's a relatively simple project, but I think it demonstrates quite nicely how
you can build useful, reliable processing pipelines with Spooler without
introducing too much complexity into your application.

In Pagedog we had only two stages for a message -- `incidents` and `reports`,
but you can easily add more in between, keeping the wiring simple and
transitions atomic.

The queue takes care of message delivery, leases, retries, and the atomic
transition between stages, leaving us to focus on the actual application logic.

I hope this gives you a useful idea of how Spooler can fit into your own
infrastructure projects.

And if you find Pagedog useful, feel free to try it out, explore the code, or
extend it with your own runtimes, investigation tools, and notification
destinations -- PRs are more than welcome!

## Postscript

One last thing worth mentioning: there's nothing particularly incident-specific
about this design really.

I chose incident investigation because it's a domain I'm familiar with and many
people are as well. But the same architecture can be applied to pretty much any
workflow, LLM-powered or not, especially when you want to run it in a
distributed fashion rather than orchestrating everything from your laptop.

Imagine a GitHub issue describing some work to be done. A worker picks it up
and starts an agent on the codebase; the pull request it opens lands on the
next queue, where another worker reviews it and potentially schedules further
work. Each stage runs on its own machines, with its own scaling and retries.

But the core remains the same -- the runtime reconciler loop and the task queue
semantics.

An LLM agent doesn't have to be a process running on your laptop, and an agent
workflow doesn't have to live inside one long-running process. The main point
here is that switching from one shape to the other doesn't take much, once the
buzzwords are trimmed off: with message queues and relatively simple workers,
you can build distributed pipelines where individual steps take time, fail,
retry, and hand work over to the next stage. Pagedog is just one example of
what you can build with this pattern, and [Spooler](https://spooler.sh) is one
way to make the coordination between those stages reliable.

Thank you for reading!

## References

- [spooler.sh][spooler]
- [docs.spooler.sh][spooler-docs]
- [pagedog.dev][pagedog]
- [modelcontextprotocol.io][mcp]

[spooler]:      https://spooler.sh
[spooler-docs]: https://docs.spooler.sh
[pagedog]:      https://pagedog.dev
[mcp]:          https://modelcontextprotocol.io
[alertmanager]: https://prometheus.io/docs/alerting/latest/alertmanager/
[prometheus]:   https://prometheus.io
[loki]:         https://grafana.com/oss/loki/
[incident-io]:  https://incident.io
[pagerduty]:    https://www.pagerduty.com
