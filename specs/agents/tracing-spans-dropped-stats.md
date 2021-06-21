## Collecting statistics about dropped spans

To still retain some information about dropped spans (for example due to `transaction_max_spans` or `span_min_duration`),
agents SHOULD collect statistics on the corresponding transaction about dropped spans.
These statistics MUST only be sent for sampled transactions.

### Use cases

This allows APM Server to consider these metrics for the service destination metrics.
In practice,
this means that the service map, the dependencies table,
and the backend details view can show accurate throughput statistics for backends like Redis,
even if most of the spans are dropped.

This also allows the transaction details view (aka. waterfall) to show a summary of the dropped spans.

### Data model

This is an example of the statistics that are added do the `transaction` events sent via the intake v2 protocol.

```json
{
  "dropped_spans_stats": [
    {
      "type": "external",
      "subtype": "http",
      "destination_service_resource": "example.com:443",
      "outcome": "failure",
      "count": 28,
      "duration.sum.us": 123456
    },
    {
      "type": "db",
      "subtype": "mysql",
      "destination_service_resource": "mysql",
      "outcome": "success",
      "count": 81,
      "duration.sum.us": 9876543
    }
  ]
}
```

### Limits
TODO: limit the number of `dropped_spans_stats` elements?

### Effects on destination service metrics

As laid out in the [span destination spec](tracing-spans-destination.md#contextdestinationserviceresource),
APM Server tracks span destination metrics.
To avoid dropped spans to skew latency metrics and cause throughput metrics to be under-counted,
APM Server will take `dropped_spans_stats` into account when tracking span destination metrics.