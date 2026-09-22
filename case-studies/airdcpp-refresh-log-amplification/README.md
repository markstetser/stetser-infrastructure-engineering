# AirDC++ Scheduled Refresh Log Amplification

## Summary

A production AirDC++ deployment serving a large music library generated extreme volumes of repetitive system-log output during scheduled recursive share refreshes.

The repeated message was:

```text
File matches the share skiplist
```

The affected environment intentionally excluded `.log` files from the share. Because the library contained millions of files across tens of thousands of directories, scheduled refreshes produced log amplification severe enough to require operating-system-level log rotation as a containment measure.

A later forensic investigation traced the behavior through:

- runtime symptoms;
- a previously modified AirDC++ binary;
- byte-level binary comparison;
- source-code inspection;
- recursive refresh logic;
- error aggregation behavior;
- and historical commit analysis.

## Environment

- AirDC++ core: 2.14.0
- Web UI: 4.30
- Linux
- Docker
- Large multi-terabyte music library
- Millions of files
- Tens of thousands of directories
- `.log` files intentionally matched by the share skiplist

## Operational Symptom

Scheduled share refreshes generated very large numbers of repetitive blocked-share messages associated with expected skiplist matches.

The immediate operational response was to limit log growth externally using operating-system log rotation. This reduced production risk but did not address the underlying application behavior.

## Binary Forensics

A leftover executable named `airdcppd` was later discovered during infrastructure cleanup.

Comparison with the current production AirDC++ binary showed:

- identical file size;
- identical reported AirDC++ version;
- identical executable code section;
- exactly 31 differing bytes;
- all 31 differing bytes were located in read-only string data;
- the production binary contained:

```text
File matches the share skiplist
```

- the older artifact contained 31 NUL bytes at the same location.

This showed that the older binary had been deliberately modified to suppress that exact message while leaving executable code unchanged.

## Source-Level Findings

Current AirDC++ source shows that share skiplist matches originate in `SharePathValidator::checkSharedName()`.

A matching filename causes a `ShareValidatorException` using the `SKIPLIST_SHARE_MATCH` resource string.

During recursive share refreshes:

```text
ShareTasks::runRefreshTask()
  -> refreshPath()
  -> ShareManager::handleRefreshPath()
  -> ShareBuilder::buildTree()
  -> validateFileItem()
  -> SharePathValidator::checkSharedName()
```

`validateFileItem()` catches reportable validation exceptions and adds blocked file errors to an `ErrorCollector`.

The important detail is scope:

```cpp
void ShareBuilder::buildTree(...) {
    ErrorCollector errors;
    ...
}
```

Each recursive directory traversal creates a new `ErrorCollector`.

At the end of that directory scan:

```cpp
auto msg = errors.getMessage();
if (!msg.empty()) {
    log(...);
}
```

`ErrorCollector` does aggregate repeated identical errors within a directory. If more than three files share the same error, it reports a count instead of listing every filename.

However, because the collector is created inside each recursive `buildTree()` invocation, aggregation occurs only at directory scope.

This means a scheduled recursive refresh can still generate approximately one blocked-share log message per affected directory.

## Historical Commit Analysis

Historical commit:

```text
314d74f5eb9f001f17dd7908d180b01fabfdca36
Rework error reporting for blocked share files
2017-04-04
```

introduced the `ErrorCollector` behavior and changed share validation error reporting.

Before that change, `SharePathValidator` itself contained a temporal duplicate-message suppression mechanism intended to prevent repeated monitoring messages from spamming the system log.

The 2017 rework removed that suppression from the validator and created two different reporting models.

### Live Share Monitoring

`ShareMonitorManager` retained temporal suppression behavior, avoiding repeated identical messages within a short interval.

### Recursive Share Refresh

`ShareManager::ShareBuilder` aggregated blocked-file errors only within each individual directory.

This produces a scalability gap:

```text
live monitoring:
    repeated event
    -> temporal suppression

scheduled recursive refresh:
    directory A
    -> aggregate
    -> log

    directory B
    -> aggregate
    -> log

    directory C
    -> aggregate
    -> log
```

For small shares this behavior is unlikely to be noticeable.

For very large directory trees containing intentionally skiplisted files, log volume can scale with the number of affected directories per refresh.

## Root Cause

The issue is not that skiplist validation is incorrect.

The issue is the scope of reporting aggregation.

Before the patch, expected skiplist matches during a bulk recursive refresh were aggregated only per directory.

The resulting logging behavior was effectively:

```text
O(number of affected directories)
```

After the patch, blocked-file errors are aggregated across the full recursive traversal of each refresh path, producing approximately one blocked-file summary per refresh path for a repeated expected condition.

## Implemented Fix

The final patch changes the lifetime of `ErrorCollector` from one recursive directory invocation to one refresh path.

Previously, every recursive `ShareBuilder::buildTree(...)` call created a new collector. That meant identical blocked-file errors were aggregated only within an individual directory.

The patch now:

1. Creates one `ErrorCollector` in the top-level `ShareBuilder::buildTree()` call.
2. Passes that collector by reference through the recursive traversal.
3. Emits the blocked-file summary after the entire refresh path has been scanned.

This preserves the existing batching behavior while moving the aggregation boundary to the level where it is useful for large recursive refreshes.

Directory validation logging remains unchanged.

## Operational Containment

External log rotation remains a reasonable defensive control for large deployments.

It limits disk-consumption risk, but it should be treated as containment rather than root-cause remediation. The source-level change addresses the amplification mechanism itself.

## Engineering Lessons

This investigation reinforced several general lessons:

- operational containment and root-cause correction are separate goals;
- harmless-looking informational logging can become a scalability problem;
- aggregation scope matters;
- source archaeology can explain historical behavior that is difficult to reproduce directly;
- binary artifacts can preserve useful forensic evidence long after an incident;
- very large datasets expose behavior that may never appear in ordinary development environments.

## Upstream Status

A source-level fix has been implemented, committed, and validated successfully using the project's existing Windows build pipeline.

The change was submitted upstream as AirDC++ pull request #225, titled `Aggregate blocked share errors per refresh path`.

The pull request is currently open for maintainer review.
