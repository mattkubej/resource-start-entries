# Resource Start Entries

This proposal adds a `"resource-start"` Performance Timeline entry when the browser begins a resource load that will be reported by [PerformanceResourceTiming](https://w3c.github.io/resource-timing/#sec-performanceresourcetiming). A shared `resourceId` correlates the start entry with the existing Resource Timing entry.

## Use cases

The primary use case is navigation lifecycle monitoring, where network inactivity can help determine when a navigation has completed or a page has become idle. Resource Timing reports a resource load only after it has completed, failed, or been canceled. Resource Timing alone therefore cannot show whether any resource loads are still in progress.

One possible workaround is to wrap or replace `fetch()` and `XMLHttpRequest` and track requests issued through those APIs. This approach misses resource loads not initiated through those APIs, including stylesheets, images, fonts, and module dependencies. It also modifies shared APIs and must coexist with any other code that modifies them.

The browser-generated start entry covers the same resource loads as Resource Timing, regardless of what initiated them. Pairing the start and Resource Timing entries shows which resource loads remain in progress without instrumenting individual request paths. The same signal can support navigation, page-idle, and interaction monitoring.

## Proposed API

For each resource load, the browser exposes:

1. A new `"resource-start"` entry when the resource load begins.
2. The existing `"resource"` entry after the resource load completes, fails, or is canceled.

Both entries carry the same `resourceId`. The same URL can be requested more than once, so URL and timestamp comparisons cannot reliably match the entries.

The `"resource-start"` entry contains:

1. `name`: The initially requested URL.
2. `entryType`: `"resource-start"`.
3. `startTime`: The same start time later reported by the matching Resource Timing entry.
4. `duration`: `0`.
5. `initiatorType`: The same initiator classification used by Resource Timing.
6. `resourceId`: An opaque identifier for this resource load within the current `Window` or worker.

The entries follow these rules:

1. Start and Resource Timing entries cover the same resource loads.
2. Every start entry is followed by exactly one Resource Timing entry with the same `resourceId`, including when the load fails or is canceled.
3. The browser records the `"resource-start"` entry before the matching Resource Timing entry.
4. `startTime` records when the resource load began, not when the observer receives the entry.
5. The `"resource-start"` entry and its matching Resource Timing entry are reported to the same `Window` or worker.

## Example

The following example creates a network-activity signal for navigation monitoring. `hasInProgressResources()` reports whether any observed resource loads have started without yet producing a matching Resource Timing entry.

```javascript
const inProgressResourceIds = new Set();

function hasInProgressResources() {
  return inProgressResourceIds.size > 0;
}

const resourceObserver = new PerformanceObserver((entryList) => {
  for (const { resourceId } of entryList.getEntriesByType("resource-start")) {
    inProgressResourceIds.add(resourceId);
  }

  for (const { resourceId } of entryList.getEntriesByType("resource")) {
    inProgressResourceIds.delete(resourceId);
  }
});

resourceObserver.observe({
  type: "resource-start",
  buffered: true,
});
resourceObserver.observe({
  type: "resource",
  buffered: true,
});
```

## References

1. [PerformanceObserver](https://w3c.github.io/performance-timeline/#the-performanceobserver-interface)
2. [PerformanceResourceTiming](https://w3c.github.io/resource-timing/#sec-performanceresourcetiming)
3. [Fetch Standard](https://fetch.spec.whatwg.org/)
