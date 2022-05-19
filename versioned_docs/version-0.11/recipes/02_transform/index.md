# Transform

:::note
All the application code here is available from the docs [git repository](__GIT__).
:::

The `transform` example builds on the [filter example](../01_filter/index.md) and extends the [`example.trickle`](etc/tremor/config/example.trickle) by adding a transformation that modifies the incoming event. The produced event from this query statement has a different structure than the incoming event.

## Environment

It connects to the pipeline `example` in the [`example.trickle`](etc/tremor/config/example.trickle) file using the tremor script language to change the json for the log.

All other configuration is the same as per the passthrough example, and is elided here for brevity.

## Business Logic

```trickle
select {                                     # 1. We can inline new json-like document structures
    "hello": "hi there #{event.hello}",      # 2. Tremor supports flexible string interpolation useful for templating
    "world": event.hello
} from in where event.selected into out
```

## Command line testing during logic development

Run a the passthrough query against a sample input.json

```bash
$ cat input.json 
{ "hello": "world", "selected": false }

# no output
$ tremor run -i input.json ./etc/tremor/config/example.trickle
$
```

Change the `input.json` and toggle the `selected` field to `true` and run again.

```bash
$ cat input.json 
{ "hello": "world", "selected": true }

$ tremor run -i input.json ./etc/tremor/config/example.trickle
{"hello":"hi there world","world":"world"}
```

Deploy the solution into a docker environment - `docker-compose up`.

In a separate terminal, inject test messages via [websocat](https://github.com/vi/websocat)

:::note
Can be installed via `cargo install websocat` for the lazy/impatient amongst us
:::

```bash
$ cat inputs.txt | websocat ws://localhost:4242
...
```

You should see this message in the terminal where you are running `docker-compose`:

```bash
...
>> {"hello":"hi there again","world":"again"}
```

### Discussion

Transformations in tremor query ( `trickle` ) can be any legal type / value supported
by the tremor family of languages:

- A boolean value
- An integer
- A floating point value
- A UTF-8 encoded string
- An array of any legal value
- A record of field / value pairs where the field name is a string, and the value is any legal value

### Examples

#### Templating percentile estimates from HDR Histogram

In this example, we restructure output from the tremor `aggr::stats::hdr` aggregate function
and use string interpolation to generate record templates with a field naming scheme and
structure this is compatible with tremor's influx data offramp.

A nice advantage of tremor, is that the business logic is separate from any externalising
factors. However, one drawback with unstructured transformations is there is no
explicit validation by schema supported by tremor out of the box - although, there are
patterns in use to validate against external schema formats in use in production.

```trickle
select {
  "measurement":  event.measurement,
  "tags":  event.tags,
  "timestamp": event.timestamp,
  "fields":  {
    # the following fields are generated by templating / string interpolation
    "count_#{event.class}":  event.stats.count,
    "min_#{event.class}":  event.stats.min,
    "max_#{event.class}":  event.stats.max,
    "mean_#{event.class}":  event.stats.mean,
    "stdev_#{event.class}":  event.stats.stdev,
    "var_#{event.class}":  event.stats.var,
    "p50_#{event.class}":  event.stats.percentiles["0.5"],
    "p90_#{event.class}":  event.stats.percentiles["0.9"],
    "p99_#{event.class}":  event.stats.percentiles["0.99"],
    "p99.9_#{event.class}":  event.stats.percentiles["0.999"]
  }
}
from normalize
into batch;
```

:::tip
Not all tremor script idioms are allowed in the `select` statement. Most notably we do not allow any mutating operations such as `let` or control flow such as `emit` or `drop`. Those constructs can however still be used inside a `script` block on their own.
:::