# Resource Start Entries

This proposal adds a `"resource-start"` Performance Timeline entry when the browser begins a resource load that will be reported by [PerformanceResourceTiming](https://w3c.github.io/resource-timing/#sec-performanceresourcetiming). The draft uses a shared `resourceId` to correlate the start entry with the existing Resource Timing entry.

## Use cases

The primary use case is navigation lifecycle monitoring, where network inactivity can help determine when a navigation has completed or a page has become idle. This includes soft navigations, where the application updates the page without loading a new document. Resource Timing reports a resource load only after it has completed, failed, or been canceled. Resource Timing alone therefore cannot show whether any resource loads are still in progress.

Start entries cover the same resource loads as Resource Timing, including stylesheets, images, fonts, and module dependencies. Pairing the start and Resource Timing entries allows monitoring code to track resource loads without wrapping the APIs that initiate them.

Monitoring code can filter entries by URL or initiator type to choose which loads to include when observing for network inactivity.

A secondary use case is background prefetching after a page is considered idle. Monitoring code can use resource activity to help determine when to begin prefetching.

## Proposed API

For each resource load covered by Resource Timing, the browser exposes:

1. A new `"resource-start"` entry when the resource load begins.
2. The existing `"resource"` entry after the resource load completes, fails, or is canceled.

Both entry types can be observed with the same `PerformanceObserver`. The API and example below use a shared `resourceId` to match the entries.

The `"resource-start"` entry contains:

1. `name`: The initially requested URL.
2. `entryType`: `"resource-start"`.
3. `startTime`: The same start time later reported by the matching Resource Timing entry.
4. `duration`: `0`.
5. `initiatorType`: The same initiator type reported by Resource Timing, such as `"fetch"`, `"img"`, or `"css"`.
6. `resourceId`: An opaque identifier for this resource load within the current `Window` or worker.

This draft also adds `resourceId` to `PerformanceResourceTiming`, with the same value as the matching start entry.

The start entry records when the load began, not when the observer receives the entry. Its values remain unchanged as the load progresses; the matching Resource Timing entry reports completion.

The proposed scope is the resource loads reported by Resource Timing. Resource Timing can report loads [served from cache](https://w3c.github.io/resource-timing/#dom-performanceresourcetiming-deliverytype) or [handled by a service worker](https://w3c.github.io/resource-timing/#dom-performanceresourcetiming-workerstart) without a network request. Under this proposal, those loads would also produce start entries.

The entries follow these proposed rules:

1. The browser queues a start entry before the matching Resource Timing entry.
2. Every start entry is followed by exactly one Resource Timing entry with the same `resourceId` when the load completes, fails, or is canceled.
3. The start entry and its matching Resource Timing entry are reported to the same `Window` or worker.

## Example

The following example tracks observed resource loads. It adds a `resourceId` to a set when a start entry is received and removes it when the matching Resource Timing entry is received. `includeResource` selects which loads to track; this example excludes requests to `/analytics/beacon`.

For this example, register the observer before the loads being monitored and keep it registered across navigations within the application. `hasObservedLoadsInProgress()` calls `takeRecords()` to process queued entries before checking the set.

```javascript
function observeResourceActivity(includeResource) {
  const inProgressResourceIds = new Set();

  function processEntries(entries) {
    for (const entry of entries) {
      if (entry.entryType === "resource-start" && includeResource(entry)) {
        inProgressResourceIds.add(entry.resourceId);
      }
    }

    for (const entry of entries) {
      if (entry.entryType === "resource") {
        inProgressResourceIds.delete(entry.resourceId);
      }
    }
  }

  const observer = new PerformanceObserver((entryList) => {
    processEntries(entryList.getEntries());
  });
  observer.observe({ entryTypes: ["resource-start", "resource"] });

  return {
    hasObservedLoadsInProgress() {
      processEntries(observer.takeRecords());
      return inProgressResourceIds.size > 0;
    },
  };
}

// Exclude the application's analytics beacon.
const beaconURL = new URL("/analytics/beacon", location.href).href;
const activity = observeResourceActivity(({ name }) => name !== beaconURL);
```

`activity.hasObservedLoadsInProgress()` reports whether any tracked start entries are still awaiting their matching Resource Timing entries. The result reflects the entries processed by the observer. The monitoring code defines when to consider a navigation complete or a page idle.

## Frequently asked questions

### Why add a separate entry type?

Resource Timing includes fields such as `responseEnd` that are not known when a load starts. A separate `"resource-start"` entry reports the start of a load without changing when `"resource"` entries are reported. A single `PerformanceObserver` can observe both entry types, as shown in the example.

[Performance Timeline #13](https://github.com/w3c/performance-timeline/issues/13#issuecomment-108621783) discusses the alternatives, including reporting a `"resource"` entry at both the start and end of a load.

### What happens if observer callbacks are delayed?

`PerformanceObserver` callbacks run asynchronously. For navigation monitoring, delayed delivery is acceptable: measurements can be calculated after the entries arrive, using the recorded start and completion times. A delay can postpone calculating or reporting a measurement without changing the timestamps in the entries.

[`takeRecords()`](https://w3c.github.io/performance-timeline/#takerecords-method) returns the entries already queued in the observer buffer and empties that buffer. In the example, `hasObservedLoadsInProgress()` processes these entries immediately, without waiting for the `PerformanceObserver` callback to run.

For background prefetching, a delayed callback can mean prefetching begins later. Prefetching is an optimization, so the application continues to function if prefetching is delayed or skipped.

### Can an observer track loads that started before it was registered?

The `buffered: true` option requests older entries retained for an entry type. Those buffers have size limits, so the option does not guarantee a complete history. If the retained entries contain a start entry without its matching Resource Timing entry, they do not show whether the load is still in progress or the completion entry is missing.

The example keeps an observer registered while the loads are being monitored. New entries are [queued in its observer buffer](https://w3c.github.io/performance-timeline/#queue-a-performanceentry) even if the separate performance entry buffer for older entries is full.

### Could `startTime` be used instead of `resourceId`?

Using `startTime` to match the entries was raised in the [Web Performance Working Group discussion](https://w3c.github.io/web-performance/meetings/2026/2026-08-13/index.html). Giving both entries the same start time makes their timestamps consistent, but does not make that timestamp unique to a load.

Browsers [limit timestamp precision](https://w3c.github.io/hr-time/#dfn-coarsen-time) to reduce timing attacks. Separate Resource Timing entries can have the same URL and `startTime`, so those fields are not guaranteed to identify a single load. Matching by `startTime` would need to account for multiple loads with the same URL and start time.

## Alternatives considered

- **Wrapping `fetch()` and `XMLHttpRequest`, or instrumenting an HTTP library.** This can track requests issued through those APIs, but misses other resource loads, including stylesheets, images, fonts, and module dependencies. Wrappers must also coexist with other code that modifies the same APIs.
- **Service workers.** A service worker can observe requests from clients it controls. Applications would need to add monitoring to an existing service worker or introduce one for this purpose.
- **A count of all resource loads in progress.** A total count does not identify the loads being counted. It therefore does not let monitoring code exclude loads by URL or initiator type.
- **An idle event or an option to defer loads until the page is idle.** These would use the browser's definition of idle. With individual resource entries, monitoring code can choose which loads to include and when to consider a navigation complete or a page idle.
- **A separate API for observing fetches.** [Fetch #65](https://github.com/whatwg/fetch/issues/65) discusses an observer for fetch groups. Using `PerformanceObserver` lets monitoring code receive start entries and Resource Timing entries through the same API.
- **Progress notifications for individual `fetch()` calls.** [FetchObserver](https://github.com/whatwg/fetch/issues/607) and the [Fetch progress proposal](https://github.com/whatwg/fetch/pull/1843) describe observing individual `fetch()` calls. Monitoring code would still need a way to observe resource loads initiated through HTML or CSS.

## Security and privacy considerations

Start entries include the requested URL and a start time. The security review needs to determine whether reporting these when a load starts would expose information that Resource Timing would withhold, including for redirects, failures, and requests blocked by browser policies.

Cross-origin loads can produce Resource Timing entries even when the [`Timing-Allow-Origin` (TAO) check](https://fetch.spec.whatwg.org/#concept-tao-check) fails. That check restricts timing details; it does not by itself suppress the entry. Fetch also specifies [limited timing information for network errors](https://fetch.spec.whatwg.org/#fetch-finale).

## Open questions

1. Where in Fetch and HTML should start entries be created and queued? How should failures and cancellations be reported so that each start has a matching Resource Timing entry?
2. Is `resourceId` needed for navigation monitoring, or can entries be matched using `startTime`? How should matching handle entries with the same URL and start time?
3. If an identifier is used, what should it identify when a preload and a later load that uses it produce separate Resource Timing entries, as discussed in [Resource Timing #435](https://github.com/w3c/resource-timing/issues/435)? How should it relate to the per-entry `id` in the Performance Timeline draft?
4. How should start entries be retained for observers registered after loads begin?
5. How should missing entries be detected and reported? How should they be handled when calculating navigation measurements?
6. Early Hints can start a load before its document exists. When should its start entry be reported? What should happen to pending entries if a document is destroyed or a worker is terminated?

## Related work

The navigation monitoring use case was described in [Performance Timeline #13](https://github.com/w3c/performance-timeline/issues/13#issuecomment-111152115). [Resource Timing #137](https://github.com/w3c/resource-timing/issues/137) discussed a start entry with a shared identifier, buffering, continuous observation, and `takeRecords()`.

[Resource Timing #259](https://github.com/w3c/resource-timing/issues/259) discusses matching a `FetchEvent` to Resource Timing entries.

The [August 13, 2026 Web Performance Working Group discussion](https://w3c.github.io/web-performance/meetings/2026/2026-08-13/index.html) raised the alternatives, callback timing, and security concerns discussed here.

## References

1. [PerformanceObserver](https://w3c.github.io/performance-timeline/#the-performanceobserver-interface)
2. [PerformanceResourceTiming](https://w3c.github.io/resource-timing/#sec-performanceresourcetiming)
3. [Fetch Standard](https://fetch.spec.whatwg.org/)
