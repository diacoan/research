# JFrog Log Parsing Standard for Loki and Promtail

## Purpose

Define a standard parsing approach for JFrog logs in Loki and Promtail using the official JFrog log structures documented in:

- [Log Files Location and Naming](https://docs.jfrog.com/administration/docs/log-files-location-and-naming)

This document standardizes parsing for:

- service logs
- request logs
- outbound request logs
- router request logs

## Source of Truth

The official JFrog documentation provides the canonical record structures for:

- `service.log`
- `request.log`
- `request-out.log`
- `router-request.log`

Reference:

- [Log Files Location and Naming](https://docs.jfrog.com/administration/docs/log-files-location-and-naming)

Relevant documented structures:

- service log record structure
- request log record structure
- outbound request log record structure
- router request log JSON structure

## Parsing Principles

- Parse according to JFrog’s documented field order first.
- Do not infer field positions from ad-hoc samples if the official structure already exists.
- Keep raw log lines in Loki even after parsing.
- Normalize fields into a stable query contract.
- Use Promtail to extract fields and labels conservatively.
- Avoid promoting high-cardinality fields such as full URLs, usernames, trace IDs, and user agents to stream labels.

## Labeling Rules

Recommended low-cardinality labels:

- `job`
- `service`
- `log_type`
- `node` or `host`
- `product`

Allowed conditional labels:

- `service_id`
- `request_method`
- `status_class`

Do not use these as labels by default:

- `trace_id`
- `request_url`
- `remote_url`
- `username`
- `request_user_agent`
- `remote_address`

These fields should remain parsed fields in the log payload for query-time extraction.

## Standard Types

### 1. Service Log

Official structure:

```text
Timestamp (UTC) [Service Type] [Level] [Trace Id] [Class and Line Number] [Thread] - Message
```

Reference:

- [JFrog service log structure](https://docs.jfrog.com/administration/docs/log-files-location-and-naming)

Normalized fields:

```text
timestamp
service_type
log_level
trace_id
class_line_number
thread
message
```

Promtail regex shape:

```regex
^(?P<timestamp>[^ ]+)\s\[(?P<service_type>[^\]]*)\]\s\[(?P<log_level>[^\]]*)\]\s+\[(?P<trace_id>[^\]]*)\]\s\[(?P<class_line_number>[^\]]*)\]\s\[(?P<thread>[^\]]*)\]\s-\s?(?P<message>.*)$
```

Notes:

- Service logs may require multiline handling depending on stack traces.
- ANSI color stripping may be needed if colors are present.

### 2. Request Log

Official structure:

```text
Timestamp | Trace ID | Remote Address | Username | Request method | Request URL | Return Status | Request Content Length | Response Content Length | Request Duration | Request User Agent
```

Reference:

- [JFrog request log structure](https://docs.jfrog.com/administration/docs/log-files-location-and-naming)

Normalized fields:

```text
timestamp
trace_id
remote_address
username
request_method
request_url
return_status
request_content_length
response_content_length
request_duration
request_user_agent
```

Promtail regex shape:

```regex
^(?P<timestamp>[^\|]*)\|(?P<trace_id>[^\|]*)\|(?P<remote_address>[^\|]*)\|(?P<username>[^\|]*)\|(?P<request_method>[^\|]*)\|(?P<request_url>[^\|]*)\|(?P<return_status>[^\|]*)\|(?P<request_content_length>[^\|]*)\|(?P<response_content_length>[^\|]*)\|(?P<request_duration>[^\|]*)\|(?P<request_user_agent>.*)$
```

Derived fields:

```text
status_class
request_duration_ms
request_content_length_bytes
response_content_length_bytes
```

Rules:

- `request_content_length=-1` means unknown
- `response_content_length=-1` means unknown
- `request_duration` is in milliseconds

### 3. Outbound Request Log

Official structure:

```text
Timestamp | Trace ID | Remote Repository Name | Username | Request method | Request URL | Return Status | Request Content Length | Response Content Length | Request Duration
```

Reference:

- [JFrog outbound request log structure](https://docs.jfrog.com/administration/docs/log-files-location-and-naming)

Normalized fields:

```text
timestamp
trace_id
remote_repo_name
username
request_method
remote_url
return_status
request_content_length
response_content_length
request_duration
```

Promtail regex shape:

```regex
^(?P<timestamp>[^\|]*)\|(?P<trace_id>[^\|]*)\|(?P<remote_repo_name>[^\|]*)\|(?P<username>[^\|]*)\|(?P<request_method>[^\|]*)\|(?P<remote_url>[^\|]*)\|(?P<return_status>[^\|]*)\|(?P<request_content_length>[^\|]*)\|(?P<response_content_length>[^\|]*)\|(?P<request_duration>[^\|]*)$
```

Derived fields:

```text
status_class
request_duration_ms
request_content_length_bytes
response_content_length_bytes
remote_host
```

Rules:

- `remote_repo_name` is the canonical repository key for remote monitoring
- `request_duration` is in milliseconds
- `response_content_length` should be used for tiny/small/large object bucketing when available

### 4. Router Request Log

Official structure:

- JSON object emitted by JFrog Router

Reference:

- [JFrog router request log structure](https://docs.jfrog.com/administration/docs/log-files-location-and-naming)

Core fields:

```text
BackendAddr
ClientAddr
DownstreamContentSize
DownstreamStatus
Duration
RequestMethod
RequestPath
StartUTC
request_Uber-Trace-Id
request_User-Agent
time
level
msg
```

Normalized fields:

```text
backend_addr
client_addr
downstream_content_size
downstream_status
duration_ns
request_method
request_path
start_utc
trace_id
request_user_agent
timestamp
level
message
```

Rules:

- `Duration` is in nanoseconds
- `time` is the log completion time
- `StartUTC` is request start time

## Promtail Parsing Strategy

### Service Logs

- use `multiline` for stack traces
- use `regex` extraction
- use `timestamp` stage from `timestamp`
- optionally strip ANSI escapes before parsing

### Request and Request-Out Logs

- use one-line `regex` extraction
- use `timestamp` stage from `timestamp`
- derive:
  - `status_class`
  - numeric duration
  - numeric content length fields

### Router Request Logs

- use `json` stage
- normalize field names if needed at query time
- convert `DownstreamStatus` to `status_class`

## Standard Derived Fields

For all request-style logs, derive:

```text
status_class
duration_bucket
size_bucket
```

### `status_class`

Rules:

- `2xx`
- `3xx`
- `4xx`
- `5xx`

### `duration_bucket`

Suggested buckets:

- `fast`
- `elevated`
- `slow`
- `very_slow`

Exact thresholds should be defined per use case.

### `size_bucket`

Suggested buckets:

- `tiny` <= 256 KB
- `small` > 256 KB and <= 5 MB
- `large` > 5 MB
- `unknown`

## Loki Query Contract

Standard field names to use in queries:

- `timestamp`
- `trace_id`
- `request_method`
- `request_url`
- `remote_url`
- `return_status`
- `status_class`
- `request_duration`
- `request_duration_ms`
- `response_content_length`
- `response_content_length_bytes`
- `remote_repo_name`
- `service_type`
- `log_level`

This allows a consistent query model across:

- `artifactory-request.log`
- `artifactory-request-out.log`
- `router-request.log`
- service logs

## Recommended Promtail Job Split

Use separate scrape jobs for:

- service logs
- request logs
- outbound request logs
- router request logs

Reason:

- different parsing method per log family
- lower config ambiguity
- easier debugging
- easier query scoping in Loki

## Practical Conclusion

The official JFrog documentation provides enough structure to standardize Loki parsing for:

- request logs
- outbound request logs
- router request logs
- service logs

The safest implementation approach is:

1. keep one Promtail parser per log family
2. normalize to a shared field contract
3. keep high-cardinality fields out of labels
4. use Loki queries for enrichment and aggregation
