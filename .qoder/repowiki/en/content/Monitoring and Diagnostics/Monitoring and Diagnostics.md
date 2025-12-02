# Monitoring and Diagnostics

<cite>
**Referenced Files in This Document**
- [Monitors.java](file://community/monitoring/src/main/java/org/neo4j/monitoring/Monitors.java)
- [DiagnosticsManager.java](file://community/diagnostics/src/main/java/org/neo4j/internal/diagnostics/DiagnosticsManager.java)
- [DiagnosticsProvider.java](file://community/diagnostics/src/main/java/org/neo4j/internal/diagnostics/DiagnosticsProvider.java)
- [DiagnosticsLogger.java](file://community/diagnostics/src/main/java/org/neo4j/internal/diagnostics/DiagnosticsLogger.java)
- [SystemDiagnostics.java](file://community/kernel/src/main/java/org/neo4j/kernel/diagnostics/providers/SystemDiagnostics.java)
- [DiagnosticsReportCommand.java](file://community/dbms/src/main/java/org/neo4j/commandline/dbms/DiagnosticsReportCommand.java)
- [QueryExecutionMonitor.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/monitoring/QueryExecutionMonitor.java)
- [TransactionCounters.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/transaction/stats/TransactionCounters.java)
- [DatabaseTransactionStats.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/transaction/stats/DatabaseTransactionStats.java)
- [PageCacheTracer.java](file://community/io/src/main/java/org/neo4j/io/pagecache/tracing/PageCacheTracer.java)
- [DefaultPageCacheTracer.java](file://community/io/src/main/java/org/neo4j/io/pagecache/tracing/DefaultPageCacheTracer.java)
- [QueryTransactionStatisticsAggregator.java](file://community/kernel/src/main/java/org/neo4j/kernel/api/query/QueryTransactionStatisticsAggregator.java)
- [QuerySnapshot.java](file://community/kernel/src/main/java/org/neo4j/kernel/api/query/QuerySnapshot.java)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Core Components](#core-components)
4. [Event-Driven Monitoring Architecture](#event-driven-monitoring-architecture)
5. [Metrics Collection System](#metrics-collection-system)
6. [Diagnostics Reporting System](#diagnostics-reporting-system)
7. [Query Performance Tracking](#query-performance-tracking)
8. [Transaction Monitoring](#transaction-monitoring)
9. [Page Cache Tracing](#page-cache-tracing)
10. [Practical Implementation Examples](#practical-implementation-examples)
11. [Best Practices and Guidelines](#best-practices-and-guidelines)
12. [Troubleshooting and Debugging](#troubleshooting-and-debugging)

## Introduction

Neo4j's monitoring and diagnostics system provides comprehensive visibility into database operations, performance characteristics, and operational health. This system enables administrators and developers to understand database behavior, identify performance bottlenecks, and troubleshoot issues effectively.

The monitoring framework is built around an event-driven architecture that captures metrics from various subsystems including query execution, transactions, page caching, and storage operations. The diagnostics system complements this by providing structured reporting capabilities for operational assessment and problem diagnosis.

## Architecture Overview

The monitoring and diagnostics system consists of several interconnected components that work together to provide comprehensive observability:

```mermaid
graph TB
subgraph "Monitoring Layer"
Monitors[Monitors Service]
QueryMonitor[QueryExecutionMonitor]
TransMonitor[TransactionCounters]
PageTracer[PageCacheTracer]
end
subgraph "Collection Layer"
MetricsRegistry[Metrics Registry]
EventListeners[Event Listeners]
Counters[Counters]
end
subgraph "Diagnostics Layer"
DiagnosticsManager[DiagnosticsManager]
Providers[DiagnosticsProviders]
ReportSources[Report Sources]
end
subgraph "Reporting Layer"
DiagnosticsReporter[DiagnosticsReporter]
ArchiveGenerator[Archive Generator]
CLICommands[CLI Commands]
end
Monitors --> QueryMonitor
Monitors --> TransMonitor
Monitors --> PageTracer
QueryMonitor --> MetricsRegistry
TransMonitor --> Counters
PageTracer --> EventListeners
DiagnosticsManager --> Providers
DiagnosticsManager --> ReportSources
DiagnosticsReporter --> ArchiveGenerator
DiagnosticsReporter --> CLICommands
```

**Diagram sources**
- [Monitors.java](file://community/monitoring/src/main/java/org/neo4j/monitoring/Monitors.java#L51-L245)
- [DiagnosticsManager.java](file://community/diagnostics/src/main/java/org/neo4j/internal/diagnostics/DiagnosticsManager.java#L32-L72)

## Core Components

### Monitors Service

The Monitors service serves as the central hub for registering and managing monitoring listeners. It implements a dynamic proxy pattern to enable event-driven monitoring across the system.

```mermaid
classDiagram
class Monitors {
-Map~Method, Set~MonitorListenerInvocationHandler~~ methodMonitorListeners
-MutableBag~Class~ monitoredInterfaces
-Monitors parent
-FailureHandler failureHandler
+newMonitor(Class~T~, String...) T
+addMonitorListener(Object, String...)
+removeMonitorListener(Object)
-cleanupMonitorListeners(Object, Method)
-createInvocationHandler(Object, String[])
}
class MonitorListenerInvocationHandler {
<<interface>>
+getMonitorListener() Object
+invoke(Object, Method, Object[], String...)
}
class MonitorInvocationHandler {
-Monitors monitor
-String[] tags
+invoke(Object, Method, Object[]) Object
-invokeMonitorListeners(Monitors, String[], Object, Method, Object[])
}
Monitors --> MonitorListenerInvocationHandler
Monitors --> MonitorInvocationHandler
```

**Diagram sources**
- [Monitors.java](file://community/monitoring/src/main/java/org/neo4j/monitoring/Monitors.java#L51-L245)

**Section sources**
- [Monitors.java](file://community/monitoring/src/main/java/org/neo4j/monitoring/Monitors.java#L51-L245)

### DiagnosticsManager

The DiagnosticsManager orchestrates the collection and presentation of diagnostic information from various providers. It follows a standardized format for consistent reporting.

```mermaid
classDiagram
class DiagnosticsManager {
-int CAPTION_WIDTH
-String NAME_START
-String NAME_END
+dump(Class~E~, InternalLog, DiagnosticsLogger)
+dump(DiagnosticsProvider, InternalLog, DiagnosticsLogger)
+section(DiagnosticsLogger, String)
-header(DiagnosticsLogger, String)
-title(String) String
}
class DiagnosticsProvider {
<<interface>>
+getDiagnosticsName() String
+dump(DiagnosticsLogger)
}
class DiagnosticsLogger {
<<interface>>
+log(String)
}
DiagnosticsManager --> DiagnosticsProvider
DiagnosticsProvider --> DiagnosticsLogger
```

**Diagram sources**
- [DiagnosticsManager.java](file://community/diagnostics/src/main/java/org/neo4j/internal/diagnostics/DiagnosticsManager.java#L32-L72)
- [DiagnosticsProvider.java](file://community/diagnostics/src/main/java/org/neo4j/internal/diagnostics/DiagnosticsProvider.java#L22-L50)

**Section sources**
- [DiagnosticsManager.java](file://community/diagnostics/src/main/java/org/neo4j/internal/diagnostics/DiagnosticsManager.java#L32-L72)
- [DiagnosticsProvider.java](file://community/diagnostics/src/main/java/org/neo4j/internal/diagnostics/DiagnosticsProvider.java#L22-L50)

## Event-Driven Monitoring Architecture

The monitoring system operates on an event-driven model where various subsystems emit events that are captured by registered listeners. This architecture ensures minimal performance impact while providing comprehensive monitoring coverage.

### Event Flow Architecture

```mermaid
sequenceDiagram
participant Subsystem as Database Subsystem
participant Monitors as Monitors Service
participant Listener1 as Monitor Listener 1
participant Listener2 as Monitor Listener 2
participant Reporter as Metrics Reporter
Subsystem->>Monitors : Emit Event
Monitors->>Listener1 : Invoke Callback
Monitors->>Listener2 : Invoke Callback
Listener1->>Reporter : Collect Metrics
Listener2->>Reporter : Aggregate Data
Reporter->>Reporter : Generate Reports
```

**Diagram sources**
- [Monitors.java](file://community/monitoring/src/main/java/org/neo4j/monitoring/Monitors.java#L198-L222)

### Registration and Lifecycle

The monitoring system supports dynamic registration of listeners with support for tagging and hierarchical propagation:

```mermaid
flowchart TD
Start([Component Initialization]) --> CreateMonitors[Create Monitors Instance]
CreateMonitors --> RegisterListener[Register Monitor Listener]
RegisterListener --> ExtractInterfaces[Extract Monitor Interfaces]
ExtractInterfaces --> RegisterMethods[Register Methods with Handlers]
RegisterMethods --> Ready[Ready for Events]
Ready --> EventTriggered[Event Triggered]
EventTriggered --> FindHandlers[Find Registered Handlers]
FindHandlers --> InvokeHandlers[Invoke Handler Methods]
InvokeHandlers --> PropagateEvents[Propagate to Parent Monitors]
PropagateEvents --> Complete([Event Processing Complete])
```

**Section sources**
- [Monitors.java](file://community/monitoring/src/main/java/org/neo4j/monitoring/Monitors.java#L102-L130)

## Metrics Collection System

### Query Execution Monitoring

Query execution monitoring tracks performance metrics for individual queries and provides insights into execution patterns:

```mermaid
classDiagram
class QueryExecutionMonitor {
<<interface>>
+start(KernelTransaction)
+success(KernelTransaction, QueryResult)
+failure(KernelTransaction, Throwable)
}
class QuerySnapshot {
+String queryText
+Map~String,Object~ parameters
+long startTime
+long cpuTimeMicros
+long elapsedTimeMicros
+long pageHits
+long pageFaults
+long allocatedBytes
+ActiveLock[] waitingLocks
}
class QueryTransactionStatisticsAggregator {
<<interface>>
+recordStatisticsOfTransactionAboutToClose(long, long, long)
+recordStatisticsOfClosedTransaction(long, long, long, CommitPhaseStatisticsListener)
+pageHitsOfClosedTransactions() long
+pageFaultsOfClosedTransactions() long
}
QueryExecutionMonitor --> QuerySnapshot
QuerySnapshot --> QueryTransactionStatisticsAggregator
```

**Diagram sources**
- [QueryExecutionMonitor.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/monitoring/QueryExecutionMonitor.java#L24-L41)
- [QueryTransactionStatisticsAggregator.java](file://community/kernel/src/main/java/org/neo4j/kernel/api/query/QueryTransactionStatisticsAggregator.java#L36-L152)

**Section sources**
- [QueryExecutionMonitor.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/monitoring/QueryExecutionMonitor.java#L24-L41)
- [QueryTransactionStatisticsAggregator.java](file://community/kernel/src/main/java/org/neo4j/kernel/api/query/QueryTransactionStatisticsAggregator.java#L36-L152)

### Transaction Counters

Transaction monitoring provides comprehensive statistics about transaction lifecycle and performance:

| Metric Category | Counters | Purpose |
|----------------|----------|---------|
| **Transaction Volume** | Started, Committed, Rolled Back, Terminated | Track transaction throughput and success rates |
| **Concurrency** | Active Read/Write Transactions, Peak Concurrent | Monitor concurrent access patterns |
| **Performance** | Validation Failures, Retries | Identify transaction stability issues |
| **Resource Usage** | Heap/Native Memory Allocations | Track resource consumption |

**Section sources**
- [TransactionCounters.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/transaction/stats/TransactionCounters.java#L24-L58)
- [DatabaseTransactionStats.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/transaction/stats/DatabaseTransactionStats.java#L28-L205)

## Diagnostics Reporting System

### System Diagnostics

The system diagnostics component provides comprehensive system information for operational assessment:

```mermaid
classDiagram
class SystemDiagnostics {
<<enumeration>>
SYSTEM_MEMORY
JAVA_MEMORY
OPERATING_SYSTEM
JAVA_VIRTUAL_MACHINE
CLASSPATH
LIBRARY_PATH
SYSTEM_PROPERTIES
TIMEZONE_DATABASE
NETWORK
NATIVE_ACCESSOR
+getDiagnosticsName() String
+dump(DiagnosticsLogger)
}
class DiagnosticsReportCommand {
-Set~String~ classifiers
-JMXDumper jmxDumper
+execute()
+createAndRegisterSources(Config, Set~String~)
+registerJMXSources(DiagnosticsReporter)
}
SystemDiagnostics --> DiagnosticsReportCommand
```

**Diagram sources**
- [SystemDiagnostics.java](file://community/kernel/src/main/java/org/neo4j/kernel/diagnostics/providers/SystemDiagnostics.java#L62-L292)
- [DiagnosticsReportCommand.java](file://community/dbms/src/main/java/org/neo4j/commandline/dbms/DiagnosticsReportCommand.java#L69-L337)

### Diagnostic Classifiers

The diagnostics system supports various classifiers for targeted information collection:

| Classifier | Description | Included Information |
|------------|-------------|---------------------|
| **logs** | Log files and rotation history | Application logs, error logs, rotation archives |
| **config** | Configuration files and settings | Neo4j configuration, JVM settings, environment variables |
| **plugins** | Plugin directory structure | Installed plugins, plugin metadata, dependencies |
| **metrics** | Performance metrics | Query performance, transaction statistics, cache hit ratios |
| **threads** | Thread dumps and execution state | Thread stacks, synchronization points, deadlock detection |
| **heap** | Memory heap analysis | Heap dumps, memory allocation patterns, garbage collection |
| **sysprop** | System properties | JVM properties, OS environment, network configuration |

**Section sources**
- [DiagnosticsReportCommand.java](file://community/dbms/src/main/java/org/neo4j/commandline/dbms/DiagnosticsReportCommand.java#L69-L337)

## Query Performance Tracking

### Execution Statistics

Query performance tracking captures detailed execution metrics for analysis and optimization:

```mermaid
flowchart TD
QueryStart[Query Execution Start] --> CaptureMetrics[Capture Initial Metrics]
CaptureMetrics --> MonitorExecution[Monitor Execution]
MonitorExecution --> TrackPageCache[Track Page Cache Access]
MonitorExecution --> TrackMemory[Track Memory Usage]
MonitorExecution --> TrackLocks[Track Lock Wait Times]
TrackPageCache --> UpdateStatistics[Update Statistics]
TrackMemory --> UpdateStatistics
TrackLocks --> UpdateStatistics
UpdateStatistics --> AggregationLayer[Statistics Aggregation]
AggregationLayer --> QueryComplete[Query Execution Complete]
QueryComplete --> GenerateSnapshot[Generate Query Snapshot]
```

**Diagram sources**
- [QuerySnapshot.java](file://community/kernel/src/main/java/org/neo4j/kernel/api/query/QuerySnapshot.java#L249-L290)

### Performance Metrics

The system tracks multiple dimensions of query performance:

| Metric Category | Measurements | Purpose |
|----------------|--------------|---------|
| **Timing** | Elapsed time, CPU time, Wait time, Idle time | Understand query execution bottlenecks |
| **Resource Usage** | Memory allocations, Page cache hits/misses | Identify resource-intensive queries |
| **Concurrency** | Active locks, lock wait times | Detect contention and deadlocks |
| **Cache Behavior** | Cache hits, cache misses, cache utilization | Optimize caching strategies |

**Section sources**
- [QuerySnapshot.java](file://community/kernel/src/main/java/org/neo4j/kernel/api/query/QuerySnapshot.java#L249-L290)

## Transaction Monitoring

### Transaction Lifecycle Tracking

Transaction monitoring provides visibility into the complete transaction lifecycle:

```mermaid
stateDiagram-v2
[*] --> Started : transactionStarted()
Started --> Active : Query Execution
Active --> Committed : transactionFinished(true)
Active --> RolledBack : transactionFinished(false)
Active --> Terminated : transactionTerminated()
Committed --> [*] : Cleanup
RolledBack --> [*] : Cleanup
Terminated --> [*] : Cleanup
note right of Active : Tracks concurrency<br/>and resource usage
note right of Committed : Updates statistics<br/>and metrics
```

**Diagram sources**
- [DatabaseTransactionStats.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/transaction/stats/DatabaseTransactionStats.java#L48-L72)

### Concurrency and Resource Monitoring

The transaction monitoring system tracks concurrent access patterns and resource utilization:

```mermaid
graph LR
subgraph "Concurrency Metrics"
ActiveRead[Active Read Transactions]
ActiveWrite[Active Write Transactions]
PeakConcurrent[Peak Concurrent Transactions]
end
subgraph "Resource Metrics"
HeapAlloc[Heap Allocations]
NativeMem[Native Memory]
PageCache[Page Cache Usage]
end
subgraph "Quality Metrics"
ValidationFailures[Validation Failures]
Retries[Transaction Retries]
TimeoutRate[Timeout Rate]
end
ActiveRead --> PeakConcurrent
ActiveWrite --> PeakConcurrent
ActiveRead --> HeapAlloc
ActiveWrite --> NativeMem
PeakConcurrent --> PageCache
```

**Section sources**
- [DatabaseTransactionStats.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/transaction/stats/DatabaseTransactionStats.java#L28-L205)

## Page Cache Tracing

### Page Cache Operations

Page cache tracing provides detailed insight into storage operations and caching behavior:

```mermaid
classDiagram
class PageCacheTracer {
<<interface>>
+pins(long)
+unpins(long)
+hits(long)
+faults(long)
+evictions(long)
+cooperativeEvictions(long)
+bytesRead(long)
+bytesWritten(long)
+createPageCursorTracer(String) PageCursorTracer
}
class PageCursorTracer {
<<interface>>
+beginPin(boolean, long, PageSwapper) PinEvent
+unpin(long, PageSwapper)
+openCursor()
+closeCursor()
+reportEvents()
}
class PinEvent {
<<interface>>
+setCachePageId(long)
+beginPageFault(long, PageSwapper) PinPageFaultEvent
+hit()
+noFault()
}
PageCacheTracer --> PageCursorTracer
PageCursorTracer --> PinEvent
```

**Diagram sources**
- [PageCacheTracer.java](file://community/io/src/main/java/org/neo4j/io/pagecache/tracing/PageCacheTracer.java#L32-L544)

### Cache Performance Metrics

The page cache tracing system captures comprehensive performance metrics:

| Metric Category | Counters | Purpose |
|----------------|----------|---------|
| **Access Patterns** | Hits, Faults, Pins, Unpins | Understand cache effectiveness |
| **Performance** | Evictions, Cooperative Evictions, Flushes | Monitor cache pressure |
| **I/O Operations** | Bytes Read, Bytes Written, IOPQ | Track storage I/O patterns |
| **Memory Management** | Max Pages, Page Utilization | Monitor memory usage |

**Section sources**
- [PageCacheTracer.java](file://community/io/src/main/java/org/neo4j/io/pagecache/tracing/PageCacheTracer.java#L32-L544)
- [DefaultPageCacheTracer.java](file://community/io/src/main/java/org/neo4j/io/pagecache/tracing/DefaultPageCacheTracer.java#L34-L83)

## Practical Implementation Examples

### Registering Query Performance Monitor

```java
// Example: Registering a custom query performance monitor
Monitors monitors = new Monitors();
QueryExecutionMonitor queryMonitor = new CustomQueryExecutionMonitor();

// Register the monitor with the monitors service
monitors.addMonitorListener(queryMonitor);

// Create a monitor instance for query tracking
QueryExecutionMonitor trackingMonitor = monitors.newMonitor(QueryExecutionMonitor.class);
```

### Creating Custom Diagnostics Provider

```java
// Example: Implementing a custom diagnostics provider
public class CustomDiagnosticsProvider implements DiagnosticsProvider {
    @Override
    public String getDiagnosticsName() {
        return "Custom Performance Metrics";
    }
    
    @Override
    public void dump(DiagnosticsLogger logger) {
        // Collect and log custom performance metrics
        logger.log("Custom metric 1: " + getMetric1());
        logger.log("Custom metric 2: " + getMetric2());
    }
}
```

### Collecting Transaction Statistics

```java
// Example: Accessing transaction statistics programmatically
TransactionCounters counters = database.getDependencyResolver()
    .resolveDependency(TransactionCounters.class);

long activeTransactions = counters.getNumberOfActiveTransactions();
long committedTransactions = counters.getNumberOfCommittedTransactions();
long averageTransactionDuration = calculateAverageDuration();
```

### Generating Diagnostics Reports

```bash
# Example: Using the diagnostics command to generate reports
neo4j-admin report --to-path /tmp/diagnostics \
  --ignore-disk-space-check \
  logs config metrics threads heap
```

## Best Practices and Guidelines

### Monitoring Design Principles

1. **Minimal Performance Impact**: Use lightweight event-driven architecture to minimize overhead
2. **Granular Control**: Support selective monitoring for different subsystems
3. **Hierarchical Organization**: Enable parent-child relationships for distributed monitoring
4. **Extensibility**: Allow custom monitors and diagnostics providers

### Performance Considerations

- **Sampling Strategies**: Implement sampling for high-frequency events
- **Batch Processing**: Aggregate metrics periodically to reduce overhead
- **Memory Management**: Use efficient data structures for metric storage
- **Thread Safety**: Ensure all monitoring components are thread-safe

### Diagnostic Data Management

- **Retention Policies**: Implement appropriate data retention for historical analysis
- **Compression**: Use compression for archived diagnostic data
- **Security**: Protect sensitive diagnostic information
- **Automation**: Schedule regular diagnostic collection for proactive monitoring

## Troubleshooting and Debugging

### Common Monitoring Issues

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **High Monitoring Overhead** | Performance degradation | Reduce sampling rate or disable non-critical monitors |
| **Missing Metrics** | Incomplete monitoring data | Verify monitor registration and listener configuration |
| **Memory Leaks** | Increasing memory usage | Check for proper cleanup of monitoring resources |
| **Inconsistent Data** | Erratic metric values | Review concurrent access patterns and synchronization |

### Diagnostic Analysis Workflow

```mermaid
flowchart TD
Problem[Identify Problem] --> CollectData[Collect Diagnostic Data]
CollectData --> AnalyzeData[Analyze Diagnostic Output]
AnalyzeData --> IdentifyRootCause[Identify Root Cause]
IdentifyRootCause --> ImplementSolution[Implement Solution]
ImplementSolution --> VerifyFix[Verify Fix]
VerifyFix --> DocumentSolution[Document Solution]
```

### Debugging Tools and Techniques

- **Real-time Monitoring**: Use live metrics for immediate issue identification
- **Historical Analysis**: Examine trends over time for pattern recognition
- **Comparative Analysis**: Compare current metrics against baselines
- **Correlation Analysis**: Identify relationships between different metrics

**Section sources**
- [DiagnosticsReportCommand.java](file://community/dbms/src/main/java/org/neo4j/commandline/dbms/DiagnosticsReportCommand.java#L113-L143)

## Conclusion

Neo4j's monitoring and diagnostics system provides a comprehensive foundation for understanding database behavior and optimizing performance. The event-driven architecture ensures minimal performance impact while delivering detailed insights into query execution, transaction processing, and system resource utilization.

The combination of the Monitors service for real-time event capture and the DiagnosticsManager for structured reporting creates a powerful platform for operational excellence. By leveraging these capabilities effectively, organizations can achieve optimal database performance, proactive issue resolution, and informed capacity planning.

The extensible design allows for custom monitoring solutions tailored to specific organizational needs, while the standardized interfaces ensure compatibility and maintainability across different deployment scenarios.