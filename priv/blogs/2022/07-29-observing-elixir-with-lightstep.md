# Observing Elixir with OpenTelemetry, Lightstep, and Honeycomb
%{
    title: "Observing Elixir with OpenTelemetry, Lightstep, and Honeycomb",
    author: "Dave Lucia",
    tags: ~w(elixir observability lightstep),
    description: "How to integrate your Elixir application with Open Telemetry to make it observable in Lightstep and Honeycomb's reliability platform"
}
---
## What happens when your server goes bump in the night?
As an [Elixir developer](https://elixir-lang.org/), I get more sleep than most when an issue occurs at 2AM.  In my Elixir app, [supervisors](https://hexdocs.pm/elixir/1.13/Supervisor.html) restart processes when errors occur, keeping my systems available through dropped database connections, exceptions, etc.

When I wake up in the morning, its time to figure out what went wrong, fix the bug, and get back to feature development. *Finding the root-cause of issues is hard*, especially when the issue is not a simple amendment to an [if/else/case expression](https://hexdocs.pm/elixir/1.13/Kernel.SpecialForms.html#case/2), but a performance regression in your infrastructure or database. In order to get to the bottom of these problems, we must have a way to understand our running system. We can achieve this by practicing observability and putting the right tools in place. 

## What is observability?

Observability is simply the ability to observe your running system, with the purpose of understanding its behavior. Forget the [three pillars](https://youtu.be/EJV_CgiqlOE), or anything else you might have read, observability is all about information, tools, and analysis to get to the bottom of your burning questions faster.

If you've ever had questions like:

* Why does my `/api/foo/thing` route suddenly have a P50 of triple its normal latency?
* What is causing this memory spike in my application?
* Is this production outage affecting all of my users?

You might be in need of improving the observability of your system. Luckily, today we have standards, libraries, and tools available to answer the above questions much faster than before. 

## Can I get observability with an APM like New Relic, AppSignal, or DataDog?

Yes...and no. Application Performance Monitors (APMs) are a great way to get some observability of your system with little instrumentation. Out of the box, they will give you useful insights such as latency percentiles for each of the routes in your web application. From this metric data, you'll be able to see when you have a latency or rate spike on a particular route, due to increased traffic, a bug, or some infrastructure level problem. However, these tools only tell you **what** the problem is, they won't help you answer **why** it is happening, which is critical for the step of actually addressing any problems your application may be experiencing.

## You want telemetry data at different levels of granularity

In order to understand what's happening in your system, it's important to be able to see the big picture, and then zoom into various parts of your application to explore and discover regressions and issues. When I think about instrumenting my application with telemetry, I want the following kinds of data coming out of my system

1. **Tracing** - [High-cardinality, high-dimensional](https://www.honeycomb.io/blog/so-you-want-to-build-an-observability-tool/) spans annotating the critical operations of my application (Helps me explore, discover, and correlate specific issues)
2. **Metrics** - Low cardinality aggregate information (What's the summary of what's happening in the system over time?)
3. **Logging** - A time-series of specific events with pre-defined attributes
4. **Error Reporting** - Reporting and cataloging of errors and exceptions with stack traces

Now collecting all of this data is great, but data is not inherently valuable on its own. It is what you [can do with this data](https://lightstep.com/blog/three-pillars-zero-answers-towards-new-scorecard-observability) that is important. Advocates of observability such as [Charity Majors](https://charity.wtf/) and [Ben Sigelman](http://bensigelman.org/) argue that [you need high-cardinality ](https://thenewstack.io/observability-a-3-year-retrospective/) in order to debug production systems. To over-simplify, high-cardinality means that we are able to correlate on really specific information, like a user's unique identifier. Constrast that with low-cardinality data in metrics, where we collapse that information to provide a summary across an aggregate.

### If high-cardinality data is requisite for observability, then we need two things:

1. The ability to capture and export distributed traces of spans in our running system
2. A tool for analyzing our traces to correlate issues with root causes

We can achieve the above with [OpenTelemetry](https://opentelemetry.io/docs/instrumentation/erlang/) and an analysis tool like [Lightstep](https://www.honeycomb.io/) or [Honeycomb](https://www.honeycomb.io/).

## Instrumenting your Elixir app with OpenTelemetry

Let's instrument a brand new Elixir app with OpenTelemetry. For this guide, I'll assume you have a [Phoenix application](https://www.phoenixframework.org/) using [Elixir 1.13](https://elixir-lang.org/blog/2021/12/03/elixir-v1-13-0-released/) or newer.

### Install Dependencies

First we'll add the required dependencies into our `mix.exs` file for producing traces and spans with OpenTelemetry

```elixir
# mix.exs
{:opentelemetry, "~> 1.0"},
{:opentelemetry_api, "~> 1.0"},
{:opentelemetry_exporter, "~> 1.0", only: :prod},
```

Additionally, you should install instrumentation libraries for any of the popular libraries and frameworks you may be using

```elixir
# mix.exs
{:opentelemetry_ecto, "~> 1.0"},
{:opentelemetry_liveview, "~> 1.0.0-rc.4"},
{:opentelemetry_oban, "~> 0.2.0-rc.5"},
{:opentelemetry_phoenix, "~> 1.0"},
```

Once you run `mix deps.get`, its time to instrument your application

### Instrumenting libraries

As we saw above, there are many packages above that can give you great tracing to the libraries you're already using. These libraries typically work by hooking into [telemetry](https://hexdocs.pm/telemetry/readme.html) events produced by these libraries and converting them into OpenTelemetry spans. Note that this is a point of confusion for many people, as the `telemetry` library is an BEAM-specific package for producing telemetry data inside a BEAM application, and is not directly related to OpenTelemetry. However, it is a useful mechanism for libraries, such as [opentelemetry_ecto](https://github.com/open-telemetry/opentelemetry-erlang-contrib/blob/main/instrumentation/opentelemetry_ecto/lib/opentelemetry_ecto.ex#L42) to hook into for producing spans.

To start using these libraries, most simply require calling a `setup/1` function in our Application supervision tree.

```elixir
# my_app/lib/my_app/application.ex
defmodule MyApp.Application do
  use Application
 
  def start(_type, _args) do
    OpentelemetryPhoenix.setup()
    OpentelemetryLiveView.setup()
    OpentelemetryEcto.setup([:my_app, :repo])

    # Only trace jobs to minimize noise
    OpentelemetryOban.setup(trace: [:jobs])
  
    ...
  end
end
```

### Instrumenting your application

With distributed tracing, it is recommended to start by tracing the [boundaries of your application](https://vi.to/hubs/o11yfest/videos/3149?v=%2Fvideos%2F3149). As we saw above, we can get spans into many of the entrypoints of our application just by using out-of-the-box libraries. However, there may be internal operations that may need spans, or maybe you want to add additional information to spans that are already being produced, such as adding the `user_id` to every span in your [plug](https://hexdocs.pm/plug/readme.html) pipeline.

### Configuration

Before we start sending data, we need to add a few configurations to our app

1. Run the `opentelemetry` application as temporary

Although observability is important, if it fails it shouldn't take your app down with it! To ensure this is the case, we need to make sure `openteletry` runs in temporary mode

```elixir
# mix.exs

def project do
  [
    ...,
    releases: [
      my_app: [
          applications: [my_app: :permanent, opentelemetry: :temporary]
      ]
    ]
  ]
end
```

2. Set your runtime specific attributes for your production environment

My application currently runs in [https://fly.io]. In the startup script for my Elixir release, I add the following annotations so that every span has information about the server and environment it is executed in. Note that there are more options that you can set, see the [OpenTelemetry semantic conventions for the cloud](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/resource/semantic_conventions/cloud.md)

```elixir
read -r -d '' attributes << EOM
service.name=${FLY_APP_NAME},
service.instance.id=${FLY_ALLOC_ID},
cloud.provider=fly_io,
cloud.region.id=${FLY_REGION}
EOM

readonly OTEL_RESOURCE_ATTRIBUTES=`echo $attributes | sed 's/[[:space:]]//g'`

OTEL_RESOURCE_ATTRIBUTES=${OTEL_RESOURCE_ATTRIBUTES} \
    PHX_SERVER=true \
    exec ./bitfo start
```

## Sending data to Lightstep or Honeycomb

## Why choose? Send to both using the otel-collector
