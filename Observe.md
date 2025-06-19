---
RFC: 14
Status: Proposed
---

# `OBSERVE` command for enhanced observability in Valkey

## Abstract

Proposal describes a new OBSERVE command to enhance Valkey's observability capabilities.
By enabling advanced time-series metrics, custom gathering pipelines, and in-server data aggregation, the `OBSERVE` command will provide Valkey users with first-class monitoring capabilities, offering granular insights into server behavior and performance.

## Motivation

Currently, Valkey’s observability relies on commands such as `MONITOR`, `SLOWLOG`, and `INFO`.

While these commands are useful, they also have limitations:
- `MONITOR` streams every command, generating high data volume that may overload production environments.
- `SLOWLOG` logs only commands exceeding a set execution time, omitting quick operations and general command patterns.
- `INFO` provides server statistics but lacks detailed insights into specific commands and keys.

These commands lack the flexibility for in-depth, customizable observability that could be exposed directly within the valkey-server instance.
This includes filtering for specific commands executions, sampling data, executing custom processing steps, and aggregating metrics over time windows. 
For example, it's questionable if the current feature set has the ability to expose [The Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/).

## Design

The proposed `OBSERVE` command suite brings observability as a core feature to Valkey. Through user-defined "observability pipelines," Valkey instances can produce detailed insights in a structured, efficient manner.
These pipelines are customizable to support diverse use cases, providing users with foundational building blocks for monitoring without overwhelming server resources. This new functionality could be enhanced with integration with tools like Prometheus and Grafana for visualization or alerting, although its primary purpose is in-server analysis.

## Specification

The `OBSERVE` command set introduces the concept of observability pipelines — user-defined workflows for collecting, filtering, aggregating, and storing metrics.

### Commands

Here is the list of `OBSERVE` subcommands:

#### OBSERVE CREATE

Creates an observability pipeline with a specified configuration. Configuration details, specified in the next section, define steps such as filtering, partitioning, sampling, and aggregation.
Pipeline and it's configuration is persisted in the runtime memory (i.e. user needs to re-create the pipeline after server restart).

Syntax:
```bash
OBSERVE CREATE <pipeline_name> <configuration>
```

#### OBSERVE START

Starts data collection for the specified pipeline.

Syntax:
```bash
OBSERVE START <pipeline_name>
```

#### OBSERVE STOP

Stops data collection for the specified pipeline.

Syntax:
```bash
OBSERVE STOP <pipeline_name>
```

#### OBSERVE RETRIEVE

Retrieves collected data. (Alternatively, GET could potentially serve for this function, but further design discussion is needed.)

Syntax:
```bash
OBSERVE RETRIEVE <pipeline_name> <since_offset>
```

#### OBSERVE LOADSTEPF

Allows defining custom processing steps using Lua, for cases where built-in steps do not meet needed requirements.

Syntax:
```bash
OBSERVE LOADSTEPF <step_name> <lua_code>
```

### Configuration

Configuration of the `OBSERVE` feature is mainly done through specyfing pipelines. It's fully customizable such that we don't limit this feature to hardcoded observability characteristics.

#### Pipelines

Pipelines are configured as chains of data processing stages, including filtering, aggregation, and output buffering. Format is similar to the Unix piping.

Key stages in this pipeline model include:
 - `filter(f)`: Filters data units based on defined conditions (e.g., command type).
 - `partition(f)`: Partitions data units according to a function (e.g., by key prefix).
 - `sample(f)`: Samples data units at a specified rate.
 - `transform(f)`: Transforms each data unit with a specified function. It is append-only, so can only add data to the processed data unit.
 - `window(f)`: Aggregates data within defined time windows.
 - `reduce(f)`: Reduces data over a window via an aggregation function.
 - `output(f)`: Directs output to specified sinks.


Example configuration syntax:
```bash
OBSERVE CREATE get_errors_pipeline "
filter(filter_by_commands(['GET'])) |
filter(filter_for_errors) |
window(window_duration(1m)) |
reduce(count) |
output(output_timeseries_to_key('get_errors_count', max_length=1000))
"
```

#### Output


The goal is to capture time-series metrics within the defined pipeline outputs, f.e. for the pipeline above it would be structured as follows:

```
[<timestamp1, errors_count1>, <timestamp2, errors_count2>, ...] // capped at 1000 items
```

It remains uncertain whether storing output data in a format compatible with direct retrieval via GET (or another existing command) will be feasible. Consequently, we might need to introduce an `OBSERVE RETRIEVE <pipeline_name> <since_offset>` command for clients polling results data. This command would provide:
```
{
    current_offset: <latest_returned_offset as a number>,
    data: [ ... result items ],
    lag_detected: <true or false> // true if `since_offset` points to data that’s been removed, signaling potential data loss.
}
```

Here, offset represents the sequence number of items produced by the pipeline, including any items removed due to buffer constraints. This approach allows clients to poll for results while adjusting their polling frequency based on the lag_detected flag. If lag_detected is true, clients would be advised to increase polling frequency to reduce data loss.


### Authentication and Authorization

The introduction of our new Advanced Configuration Layer (ACL) category is a crucial step in enhancing security and control around the `OBSERVE` commands.
For all deployed Valkey instances, it will be essential to ensure that only authorized personnel can configure and enable observability pipelines, as improper configuration can lead to performance drops.
In light of this, part of the design involves creating a new ACL category specific for `OBSERVE` commands, allowing admins to fine-tune access controls and prevent unaccepted modifications.

The extent to which access will be granted for Lua step functions remains unclear. However, there is a need for some form of limitation to prevent observability steps from consuming excessive computational resources and avoiding unauthorized access to sensitive information stored within Valkey.

### Benchmarking

This is definitely something we have to do once we have a working prototype.

### Testing

Developing e2e tests with enough coverage is definitely something we have to do once we have a working prototype solution.

### Observability

Having a comprehensive observability capabilities is crucial for monitoring and analyzing Valkey performance. 
However, it's unclear whether developing a new custom observability layer for the observability pipelines is truly necessary.
This issue arises from the idea that we likely shouldn't use the `OBSERVE` pipelines to observe themselves, as in case there is something wrong, we won't get valid data.
This topic warrants further discussion, particularly within the context of the first iteration of this RFC. 

Having said that, it may be that the initial version does not require built-in observability capabilities for observability pipelines to effectively observe and monitor the pipelines themselves.

## Examples

Below are examples of how the proposed `OBSERVE` command and pipeline configurations could be used to address various observability needs.


1. **Counting Specific Commands Per Minute with Buffer Size**

   *Use Case:* Count the number of `GET` commands executed per minute.

   **Pipeline Creation:**

   ```valkey
   OBSERVE CREATE get_commands_per_minute "
   filter(filter_by_commands(['GET'])) |
   window(window_duration(1m)) |
   reduce(reduce_count) |
   output(output_timeseries_to_key('get_command_count', buffer_size=1440))
   "
   ```

   *Explanation:* This pipeline filters for `GET` commands, counts them per every minute, and stores the counts
   in a time-series key `get_command_count` with a buffer size of 1440 (e.g., one day's worth of minute-level data).

2. **Average Latency Per Time Window with Buffer**

   *Use Case:* Monitor average latency of `SET` commands per minute.

   **Pipeline Creation:**

   ```valkey
   OBSERVE CREATE set_latency_monitor "
   filter(filter_by_commands('SET')) |
   sample(sample_percentage(0.005)) |
   window(window_duration(1m)) |
   reduce(average_latency) |
   output(timeseries_to_key('set_average_latency', buffer_size=720))
   "
   ```

   *Explanation:* This pipeline filters for `SET` commands, extracts their latency, aggregates the average latency every
 minute, and stores it with a buffer size of 720 (e.g., 12 hours of minute-level data).

3. **Client Statistics**

   *Use Case:* Gather command counts per client for `GET` and `SET` commands, sampled at 5%.

   **Pipeline Creation:**

   ```shell
   OBSERVE CREATE client_stats_per_minute "
   filter(filter_by_commands(['GET', 'SET'])) |
   sample(sample_percentage(0.05)) |
   transform(transform_add_client_info) |
   window(window_duration(1m)) |
   reduce(count_by_client) |
   output(timeseries_to_key('client_stats', buffer_size=1440))
   "
   ```

   *Explanation:* This pipeline filters for `GET` and `SET` commands, samples 5% of them, extracts client information, c
ounts commands per client every minute, and stores the data under `client_stats` with a buffer size of 1440.

4. **Error Tracking**

   *Use Case:* Monitor the number of errors occurring per minute.

   **Pipeline Creation:**

   ```shell
   OBSERVE CREATE error_tracking_pipeline "
   filter(filter_for_errors) |
   window(window_duration(1m)) |
   reduce(count) |
   output(timeseries_to_key('total_errors', buffer_size=1440))
   "
   ```

   *Explanation:* This pipeline filters for commands executions that ended with an 'error', counts them every minute, and stores the totals in `tota
l_errors` with a buffer size of 1440.

5. **TTL Analysis**

   *Use Case:* Analyze the average TTL of keys set with `SETEX` command per minute.

   **Pipeline Creation:**

   ```shell
   OBSERVE CREATE ttl_analysis_pipeline "
   filter(filter_by_commands(['SETEX'])) |
   transform(transform_parse_ttl_as_int) |
   window(window_duration(1m)) |
   reduce(average_ttl) |
   output(timeseries_to_key('average_ttl', buffer_size=1440))
   "
   ```

   *Explanation:* This pipeline filters for `SETEX` commands, extracts the TTL values, calculates the average TTL every
minute, and stores it in `average_ttl` with a buffer size of 1440.

6. **Distribution of Key and Value Sizes**

   *Use Case:* Create a histogram of value sizes for `SET` commands.

   **Pipeline Creation:**

   ```shell
   OBSERVE CREATE value_size_distribution "
   filter(command('SET')) |
   transform(transform_get_value_size) |
   window(window_duration(1m)) |
   reduce(histogram(key='value_size',buckets([0, 64, 256, 1024, 4096, 16384]))) |
   output(timeseries_to_key('value_size_distribution', buffer_size=1440))
   "
   ```

   *Explanation:* This pipeline filters for `SET` commands, extracts the size of the values, aggregates them into histog
ram buckets every minute, and stores the distributions with a buffer size of 1440.


## Appendix

 - RFC is based on this GitHub issue: [OBSERVE command for enhanced observability in Valkey](https://github.com/valkey-io/valkey/issues/1167).

