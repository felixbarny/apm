[Agent spec home](README.md) > [Handling huge traces](tracing-spans-handling-huge-traces.md) > [Recycling dropped and compressed spans](tracing-spans-recycling.md)

# Recycling dropped and compressed spans

To reduce memory allocation overhead, agents MAY re-use dropped or compressed span objects.

While span compression reduces a lot of overhead if an application excessively accesses a backend,
there's still a lot of churn from creating, compressing, and then discarding compressed span objects.

When compressing a sequence of 1000 fast Redis spans,
agents may re-use span object instances so that only a total of 2 instances are allocated.

## Implementation

### Recycle discarded spans on span end

On span end, after it has been determined that a span is discarded, the span is reset and offered to the recycling buffer of its parent.
A transaction's and span's recycling buffer can hold at most one span.

Reasons for a span to be discarded include, but are not necessarily limited to:
- [span compression](tracing-spans-compress.md)
- [`exit_span_min_duration`](tracing-spans-drop-fast-exit.md)
- [`transaction_max_spans`](tracing-spans-limit.md)

```java
if (span.isDiscarded()) {
    BaseSpan parent = span.getParent()
    span.resetState()
    // noop if the parent already holds a recylced instance
    parent.offerRecycled(span)
}
```

### Re-use a recycled span on span start

Before starting a span, the agent retrieves (atomically gets and removes) the span that may be in the recycling buffer.
If there is no span in the buffer, the agent creates a new span instance.
Otherwise, it will re-use the recycled instance.

## Benefits over using an object pool

One benefit of buffering a recycled span on its parent compared to returning it into a sharded object pool is less contention.
That is because shared object pool is accessed by multiple threads as a shared resource.
If multiple threads retrieve and offer span objects at the same time,
that creates contention and increases latency for these operations.

Contention on the recycling buffer of a span is much less likely to happen in the first place.
That is, because the majority of a particular span's children are more likely to be created on the same thread. 
Even if child spans are started in different threads,
the contention is less severe as it spread across multiple span object instance as opposed to a single shared object pool.

Another benefit is that it's not necessary to perform reference counting.
That is because after a span is discarded, there are no incoming references to it.  

## Profiling

Agents that implement span recycling MUST measure the effect in benchmarks to validate that the savings are worth the additional complexity
and are not dwarfed by other allocation that the agent is doing.

Other than the allocation rate, the increased latency caused by resetting the state has to be weighed against the reduced GC pauses.
Generally, a slightly higher but more constant latency overhead is preferable to a slightly lower average overhead with higher tail latencies. 

The benchmark scenarios should cover both, scenarios that have a high, and a low rate of dropped spans. 
