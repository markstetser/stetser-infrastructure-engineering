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

Expected skiplist matches during a bulk recursive refresh are aggregated only per directory rather than per refresh task.

The resulting logging behavior is effectively:

```text
O(number of affected directories)
```

rather than:

```text
O(1) per refresh
```

for a repeated expected condition.

## Proposed Fix

Possible approaches include:

1. Aggregate blocked-share errors at refresh-task scope and emit a single summary when the refresh completes.
2. Suppress expected skiplist-match reporting during scheduled/full refreshes.
3. Provide a configurable verbosity option for blocked-share reporting during bulk refresh operations.

A refresh-level summary could resemble:

```text
Share refresh completed: N files skipped due to configured share rules
```

## Operational Containment

Until the application behavior is changed, external log rotation remains a reasonable defensive control for large deployments.

This limits disk-consumption risk but should be treated as containment rather than root-cause remediation.

## Engineering Lessons

This investigation reinforced several general lessons:

- operational containment and root-cause correction are separate goals;
- harmless-looking informational logging can become a scalability problem;
- aggregation scope matters;
- source archaeology can explain historical behavior that is difficult to reproduce directly;
- binary artifacts can preserve useful forensic evidence long after an incident;
- very large datasets expose behavior that may never appear in ordinary development environments.

## Upstream Status

An upstream AirDC++ pull request is planned based on these findings.

The proposed patch will be developed and validated separately before submission.
