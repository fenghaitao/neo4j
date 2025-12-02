# Memory Management

<cite>
**Referenced Files in This Document**
- [MuninnPageCache.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/MuninnPageCache.java)
- [PageList.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/PageList.java)
- [MemoryTracker.java](file://community/common/src/main/java/org/neo4j/memory/MemoryTracker.java)
- [MemoryAllocator.java](file://community/io/src/main/java/org/neo4j/io/mem/MemoryAllocator.java)
- [VictimPageReference.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/VictimPageReference.java)
- [LatchMap.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/LatchMap.java)
- [EvictionTask.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/EvictionTask.java)
- [BackgroundTask.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/BackgroundTask.java)
- [SwapperSet.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/SwapperSet.java)
- [UnsafeUtil.java](file://community/unsafe/src/main/java/org/neo4j/internal/unsafe/UnsafeUtil.java)
- [ByteBuffers.java](file://community/io/src/main/java/org/neo4j/io/memory/ByteBuffers.java)
- [ConfigurableIOBuffer.java](file://community/configuration/src/main/java/org/neo4j/configuration/pagecache/ConfigurableIOBuffer.java)
- [CachingOffHeapBlockAllocator.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/util/collection/CachingOffHeapBlockAllocator.java)
- [DefaultScopedMemoryTracker.java](file://community/common/src/main/java/org/neo4j/memory/DefaultScopedMemoryTracker.java)
- [HighWaterMarkMemoryPool.java](file://community/common/src/main/java/org/neo4j/memory/HighWaterMarkMemoryPool.java)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Core Memory Management Components](#core-memory-management-components)
4. [Memory Allocation Strategies](#memory-allocation-strategies)
5. [Page Caching Implementation](#page-caching-implementation)
6. [Memory Tracking and Monitoring](#memory-tracking-and-monitoring)
7. [Garbage Collection Optimization](#garbage-collection-optimization)
8. [Performance Considerations](#performance-considerations)
9. [Common Issues and Solutions](#common-issues-and-solutions)
10. [Best Practices](#best-practices)

## Introduction

Neo4j's Memory Management subsystem is a sophisticated implementation designed to handle large-scale data caching with minimal garbage collection overhead. The system centers around the MuninnPageCache, which provides efficient off-heap memory management for cached pages, direct byte buffers, and memory-mapped file operations.

The memory management architecture is built on several key principles:
- **Off-heap memory allocation** to avoid garbage collection pressure
- **Direct byte buffer access** for high-performance I/O operations  
- **Memory pooling** to reduce allocation overhead
- **Explicit memory management** to prevent memory leaks
- **NUMA-aware allocation** for optimal memory locality

## Architecture Overview

The memory management system follows a layered architecture with clear separation of concerns:

```mermaid
graph TB
subgraph "Application Layer"
PC[PageCache Interface]
PF[PagedFile Interface]
MC[PageCursor Interface]
end
subgraph "Memory Management Layer"
MPC[MuninnPageCache]
PL[PageList]
VPR[VictimPageReference]
LM[LatchMap]
end
subgraph "Memory Allocation Layer"
MA[MemoryAllocator]
MT[MemoryTracker]
BA[BufferFactory]
end
subgraph "Operating System Layer"
UB[UnsafeUtil]
DM[Direct Memory]
MM[Memory Mapping]
end
PC --> MPC
PF --> PL
MC --> PL
MPC --> PL
MPC --> VPR
MPC --> LM
PL --> MA
MA --> MT
MA --> UB
UB --> DM
UB --> MM
```

**Diagram sources**
- [MuninnPageCache.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/MuninnPageCache.java#L123-L1223)
- [PageList.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/PageList.java#L50-L515)
- [MemoryAllocator.java](file://community/io/src/main/java/org/neo4j/io/mem/MemoryAllocator.java#L26-L61)

## Core Memory Management Components

### MemoryTracker Interface

The MemoryTracker serves as the central interface for memory monitoring and allocation tracking:

```mermaid
classDiagram
class MemoryTracker {
+long usedNativeMemory()
+long estimatedHeapMemory()
+void allocateNative(long bytes)
+void releaseNative(long bytes)
+void allocateHeap(long bytes)
+void releaseHeap(long bytes)
+long heapHighWaterMark()
+void reset()
+MemoryTracker getScopedMemoryTracker()
}
class HeapMemoryTracker {
<<interface>>
+void allocateHeap(long bytes)
+void releaseHeap(long bytes)
+long heapHighWaterMark()
}
class DefaultScopedMemoryTracker {
-MemoryTracker delegate
-long trackedNative
-long trackedHeap
-boolean isClosed
+void close()
+boolean isClosed()
}
class HighWaterMarkMemoryPool {
-LongAccumulator highWaterMark
+void reserveHeap(long bytes)
+long heapHighWaterMark()
+void reset()
}
MemoryTracker --|> HeapMemoryTracker
DefaultScopedMemoryTracker --|> MemoryTracker
HighWaterMarkMemoryPool --|> MemoryTracker
```

**Diagram sources**
- [MemoryTracker.java](file://community/common/src/main/java/org/neo4j/memory/MemoryTracker.java#L24-L87)
- [DefaultScopedMemoryTracker.java](file://community/common/src/main/java/org/neo4j/memory/DefaultScopedMemoryTracker.java#L27-L115)
- [HighWaterMarkMemoryPool.java](file://community/common/src/main/java/org/neo4j/memory/HighWaterMarkMemoryPool.java#L27-L50)

**Section sources**
- [MemoryTracker.java](file://community/common/src/main/java/org/neo4j/memory/MemoryTracker.java#L24-L87)
- [DefaultScopedMemoryTracker.java](file://community/common/src/main/java/org/neo4j/memory/DefaultScopedMemoryTracker.java#L27-L115)

### MemoryAllocator Interface

The MemoryAllocator provides controlled off-heap memory allocation with alignment guarantees:

```mermaid
classDiagram
class MemoryAllocator {
<<interface>>
+static MemoryAllocator createAllocator(long expectedMemory, MemoryTracker memoryTracker)
+static MemoryAllocator createAllocator(long expectedMemory, Long grabSize, MemoryTracker memoryTracker)
+long usedMemory()
+long availableMemory()
+long allocateAligned(long bytes, long alignment)
+void close()
}
class GrabAllocator {
-long expectedMemory
-Long grabSize
-MemoryTracker memoryTracker
-long usedMemory
-long availableMemory
+long allocateAligned(long bytes, long alignment)
+void close()
}
class CachingOffHeapBlockAllocator {
-Queue~MemoryBlock~[] caches
-boolean released
-long maxCacheableBlockSize
+MemoryBlock allocate(long size, MemoryTracker tracker)
+void free(MemoryBlock block, MemoryTracker tracker)
+void release()
}
MemoryAllocator <|.. GrabAllocator
MemoryAllocator <|.. CachingOffHeapBlockAllocator
```

**Diagram sources**
- [MemoryAllocator.java](file://community/io/src/main/java/org/neo4j/io/mem/MemoryAllocator.java#L26-L61)
- [CachingOffHeapBlockAllocator.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/util/collection/CachingOffHeapBlockAllocator.java#L68-L150)

**Section sources**
- [MemoryAllocator.java](file://community/io/src/main/java/org/neo4j/io/mem/MemoryAllocator.java#L26-L61)
- [CachingOffHeapBlockAllocator.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/util/collection/CachingOffHeapBlockAllocator.java#L68-L150)

### PageList Management

The PageList manages off-heap metadata for cached pages with efficient memory layout:

```mermaid
classDiagram
class PageList {
-int pageCount
-int cachePageSize
-MemoryAllocator memoryAllocator
-SwapperSet swappers
-long victimPageAddress
-long baseAddress
-long bufferAlignment
+static final int META_DATA_BYTES_PER_PAGE = 32
+long deref(int pageId)
+void initBuffer(long pageRef)
+boolean tryEvict(long pageRef, EvictionEventOpportunity evictionOpportunity)
+void clearMemory(long baseAddress, long pageCount)
}
class OffHeapPageLock {
+static long tryOptimisticReadLock(long pageRef)
+static boolean validateReadLock(long pageRef, long stamp)
+static boolean isModified(long pageRef)
+static boolean tryWriteLock(long pageRef, boolean multiVersioned)
+static void unlockWrite(long pageRef)
+static boolean tryExclusiveLock(long pageRef)
+static void unlockExclusive(long pageRef)
}
class SwapperSet {
-SwapperMapping[] swapperMappings
-LinkedList~Integer~ free
-MutableIntSet postponedIds
+int allocate(PageSwapper swapper)
+void free(int id)
+SwapperMapping getAllocation(int id)
+void sweep(Consumer~IntSet~ evictAllLoadedPagesCallback)
}
PageList --> OffHeapPageLock
PageList --> SwapperSet
```

**Diagram sources**
- [PageList.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/PageList.java#L50-L515)
- [SwapperSet.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/SwapperSet.java#L45-L154)

**Section sources**
- [PageList.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/PageList.java#L50-L515)
- [SwapperSet.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/SwapperSet.java#L45-L154)

## Memory Allocation Strategies

### Direct Byte Buffer Management

Neo4j employs sophisticated direct byte buffer allocation strategies to minimize garbage collection pressure:

```mermaid
sequenceDiagram
participant App as Application
participant MT as MemoryTracker
participant UB as UnsafeUtil
participant DM as Direct Memory
App->>MT : allocateNative(bytes)
MT->>UB : allocateMemory(bytes, memoryTracker)
UB->>DM : malloc(bytes)
DM-->>UB : address
UB-->>MT : address
MT-->>App : ByteBuffer
Note over App,DM : Buffer operations occur directly on native memory
App->>MT : releaseNative(bytes)
MT->>UB : free(address, bytes, memoryTracker)
UB->>DM : free(address)
```

**Diagram sources**
- [UnsafeUtil.java](file://community/unsafe/src/main/java/org/neo4j/internal/unsafe/UnsafeUtil.java#L638-L729)
- [ByteBuffers.java](file://community/io/src/main/java/org/neo4j/io/memory/ByteBuffers.java#L36-L95)

### Memory Pooling and Reuse

The system implements intelligent memory pooling to reduce allocation overhead:

```mermaid
flowchart TD
A[Memory Request] --> B{Pool Available?}
B --> |Yes| C[Check Pool Size]
B --> |No| D[Allocate New Block]
C --> E{Block Size Match?}
E --> |Yes| F[Return Cached Block]
E --> |No| G[Allocate New Block]
F --> H[Use Block]
G --> H
D --> H
H --> I[Use Block]
I --> J[Release Block]
J --> K[Return to Pool]
K --> L[Pool Full?]
L --> |Yes| M[Free Extra Blocks]
L --> |No| N[Wait for Next Request]
M --> N
```

**Section sources**
- [CachingOffHeapBlockAllocator.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/util/collection/CachingOffHeapBlockAllocator.java#L68-L150)

### Buffer Alignment and Memory Layout

The system ensures optimal memory alignment for performance:

| Component | Alignment Requirement | Purpose |
|-----------|----------------------|---------|
| Page Buffers | Cache page size | Minimize TLB misses |
| Metadata | 8 bytes (long) | Atomic operations |
| IO Buffers | Page size or memory page size | Optimal I/O performance |
| Victim Pages | Cache page size | Safe fallback mechanism |

**Section sources**
- [PageList.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/PageList.java#L53-L54)
- [VictimPageReference.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/VictimPageReference.java#L24-L42)

## Page Caching Implementation

### MuninnPageCache Architecture

The MuninnPageCache implements a sophisticated page replacement algorithm with background eviction:

```mermaid
graph TB
subgraph "Page Cache Core"
MPC[MuninnPageCache]
FL[Free List]
ET[Eviction Thread]
PM[Page Manager]
end
subgraph "Memory Management"
MA[MemoryAllocator]
MT[MemoryTracker]
BA[BufferFactory]
end
subgraph "Page Operations"
PF[Page Fault]
EV[Eviction]
FLUSH[Page Flush]
end
MPC --> FL
MPC --> ET
MPC --> PM
PM --> PF
PM --> EV
PM --> FLUSH
ET --> EV
PF --> MA
EV --> MA
MA --> MT
BA --> MA
```

**Diagram sources**
- [MuninnPageCache.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/MuninnPageCache.java#L123-L1223)
- [EvictionTask.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/EvictionTask.java#L29-L39)

### Page Fault Handling

Page faults are handled efficiently with concurrent access control:

```mermaid
sequenceDiagram
participant TC as Thread
participant PC as PageCache
participant LM as LatchMap
participant PF as PageFault
participant SW as Swapper
TC->>PC : getPage(pageId)
PC->>LM : takeOrAwaitLatch(pageId)
LM->>LM : check existing latch
alt No latch exists
LM->>LM : create new latch
LM-->>PC : latch acquired
else Latch exists
LM->>LM : await latch release
LM-->>PC : latch acquired
end
PC->>PF : faultPage(pageId)
PF->>SW : read(filePageId, buffer)
SW-->>PF : data read
PF->>PF : bind page to file
PF-->>PC : page ready
PC->>LM : release latch
LM->>LM : unblock waiting threads
```

**Diagram sources**
- [LatchMap.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/LatchMap.java#L78-L98)
- [MuninnPageCache.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/MuninnPageCache.java#L898-L922)

**Section sources**
- [LatchMap.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/LatchMap.java#L78-L98)
- [MuninnPageCache.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/MuninnPageCache.java#L898-L922)

### Background Eviction Process

The eviction system operates asynchronously to maintain cache performance:

```mermaid
flowchart TD
A[Eviction Task Start] --> B[Scan Free Pages]
B --> C{Free Pages < Threshold?}
C --> |No| D[Sleep Until Woken]
C --> |Yes| E[Select Victim Page]
E --> F[Acquire Exclusive Lock]
F --> G{Page Locked?}
G --> |No| H[Retry Selection]
G --> |Yes| I{Page Loaded?}
I --> |No| J[Unlock & Continue]
I --> |Yes| K{Page Modified?}
K --> |No| L[Clear Binding]
K --> |Yes| M{Page Flushable?}
M --> |Yes| N[Flush to Disk]
M --> |No| O[Mark Unmodified]
N --> L
O --> L
L --> P[Unlock Page]
P --> Q[Add to Free List]
Q --> R[Notify Waiters]
H --> E
D --> B
```

**Diagram sources**
- [EvictionTask.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/EvictionTask.java#L29-L39)
- [BackgroundTask.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/BackgroundTask.java#L25-L49)

**Section sources**
- [EvictionTask.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/EvictionTask.java#L29-L39)
- [BackgroundTask.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/BackgroundTask.java#L25-L49)

## Memory Tracking and Monitoring

### Scoped Memory Tracking

The system provides hierarchical memory tracking for precise resource management:

```mermaid
classDiagram
class ScopedMemoryTracker {
<<interface>>
+boolean isClosed()
+void close()
+MemoryTracker getScopedMemoryTracker()
}
class DefaultScopedMemoryTracker {
-MemoryTracker delegate
-long trackedNative
-long trackedHeap
-boolean isClosed
+void allocateNative(long bytes)
+void releaseNative(long bytes)
+void allocateHeap(long bytes)
+void releaseHeap(long bytes)
+void close()
}
class HighWaterMarkMemoryPool {
-LongAccumulator highWaterMark
+void reserveHeap(long bytes)
+void reserveNative(long bytes)
+void releaseHeap(long bytes)
+void releaseNative(long bytes)
+long heapHighWaterMark()
+void reset()
}
ScopedMemoryTracker <|.. DefaultScopedMemoryTracker
DefaultScopedMemoryTracker --> MemoryTracker
HighWaterMarkMemoryPool --> MemoryTracker
```

**Diagram sources**
- [DefaultScopedMemoryTracker.java](file://community/common/src/main/java/org/neo4j/memory/DefaultScopedMemoryTracker.java#L27-L115)
- [HighWaterMarkMemoryPool.java](file://community/common/src/main/java/org/neo4j/memory/HighWaterMarkMemoryPool.java#L27-L50)

### Memory Pressure Detection

The system monitors memory usage patterns to detect and respond to pressure:

| Metric | Purpose | Threshold |
|--------|---------|-----------|
| Native Memory Usage | Track off-heap consumption | Configurable limit |
| Heap High Water Mark | Monitor GC pressure | Runtime adaptive |
| Free Page Count | Cache availability | 5-50% of total pages |
| Allocation Rate | Detect memory leaks | Sliding window average |

**Section sources**
- [DefaultScopedMemoryTracker.java](file://community/common/src/main/java/org/neo4j/memory/DefaultScopedMemoryTracker.java#L27-L115)
- [HighWaterMarkMemoryPool.java](file://community/common/src/main/java/org/neo4j/memory/HighWaterMarkMemoryPool.java#L27-L50)

## Garbage Collection Optimization

### Off-Heap Memory Management

Neo4j minimizes garbage collection pressure through careful memory management:

```mermaid
graph LR
subgraph "Traditional Java Memory"
A[Object Allocation]
B[GC Pressure]
C[Stop-the-World Pauses]
end
subgraph "Neo4j Memory Model"
D[Off-Heap Allocation]
E[Manual Cleanup]
F[Minimal GC Impact]
end
A --> B
B --> C
D --> E
E --> F
G[UnsafeUtil] -.-> D
H[Direct ByteBuffers] -.-> D
I[MemoryAllocators] -.-> D
```

### Explicit Buffer Cleanup

The system ensures proper cleanup of native resources:

```mermaid
sequenceDiagram
participant App as Application
participant SC as ScopedBuffer
participant MT as MemoryTracker
participant UB as UnsafeUtil
App->>SC : new NativeScopedBuffer(size)
SC->>MT : allocateNative(size)
SC->>UB : allocateMemory(size)
UB-->>SC : address
SC-->>App : buffer
Note over App,UB : Buffer operations
App->>SC : close()
SC->>UB : free(address, size)
SC->>MT : releaseNative(size)
MT-->>SC : confirmation
SC-->>App : cleanup complete
```

**Diagram sources**
- [ByteBuffers.java](file://community/io/src/main/java/org/neo4j/io/memory/ByteBuffers.java#L36-L95)
- [UnsafeUtil.java](file://community/unsafe/src/main/java/org/neo4j/internal/unsafe/UnsafeUtil.java#L638-L729)

**Section sources**
- [ByteBuffers.java](file://community/io/src/main/java/org/neo4j/io/memory/ByteBuffers.java#L36-L95)
- [UnsafeUtil.java](file://community/unsafe/src/main/java/org/neo4j/internal/unsafe/UnsafeUtil.java#L638-L729)

## Performance Considerations

### NUMA Awareness

The memory management system considers NUMA topology for optimal performance:

- **Memory Allocation Policy**: Allocates memory near CPU cores
- **Page Migration**: Moves pages to preferred NUMA nodes
- **Affinity Management**: Pins threads to specific NUMA domains

### Memory Locality Optimization

| Technique | Benefit | Implementation |
|-----------|---------|----------------|
| Cache Line Alignment | Reduced false sharing | 64-byte boundaries |
| Prefetching | Hidden latency | Hardware prefetch hints |
| Batch Operations | Improved throughput | Vectorized I/O operations |
| Memory Barriers | Correct ordering | VarHandle fences |

### I/O Optimization

The system optimizes I/O operations through buffering and batching:

```mermaid
graph TB
subgraph "I/O Buffering"
IB[IOBuffer]
CB[Configurable Buffer]
VB[Victim Buffer]
end
subgraph "Batch Operations"
VO[Vectored I/O]
BO[Bulk Operations]
CO[Caching Optimizations]
end
subgraph "Memory Management"
MA[Memory Allocator]
MT[Memory Tracker]
PO[Pool Management]
end
IB --> VO
CB --> BO
VB --> CO
VO --> MA
BO --> MT
CO --> PO
```

**Section sources**
- [ConfigurableIOBuffer.java](file://community/configuration/src/main/java/org/neo4j/configuration/pagecache/ConfigurableIOBuffer.java#L31-L86)

## Common Issues and Solutions

### Memory Leaks

Common causes and prevention strategies:

| Issue | Cause | Solution |
|-------|-------|----------|
| Unclosed Buffers | Missing close() calls | Use try-with-resources |
| Reference Retention | Stale page references | Proper eviction policies |
| Native Memory Exhaustion | Large page caches | Configurable limits |
| Buffer Overflows | Incorrect sizing | Bounds checking |

### Address Space Exhaustion

Mitigation strategies for large deployments:

```mermaid
flowchart TD
A[Address Space Monitoring] --> B{Usage > Limit?}
B --> |No| C[Continue Normal Operation]
B --> |Yes| D[Trigger Eviction]
D --> E[Free Non-Critical Pages]
E --> F{Still Exceeded?}
F --> |No| G[Reduce Cache Size]
F --> |Yes| H[Log Warning]
G --> I[Adjust Configuration]
H --> J[Emergency Cleanup]
I --> C
J --> C
```

### Performance Degradation

Symptoms and resolution approaches:

- **High GC Pause Times**: Reduce off-heap allocation rate
- **Page Fault Spikes**: Increase cache size or improve prefetching
- **Memory Fragmentation**: Use memory compaction strategies
- **NUMA Imbalance**: Adjust thread affinity and memory placement

**Section sources**
- [MuninnPageCache.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/MuninnPageCache.java#L127-L148)

## Best Practices

### Configuration Guidelines

Recommended settings for different deployment scenarios:

| Scenario | Cache Size | Buffer Size | Eviction Policy |
|----------|------------|-------------|-----------------|
| Development | 128MB | 8KB | Aggressive |
| Production | 1GB+ | 64KB | Balanced |
| Large Scale | 8GB+ | 256KB | Conservative |

### Monitoring and Maintenance

Essential metrics to monitor:

- **Memory Utilization**: Native and heap usage trends
- **Page Fault Rate**: Cache hit ratio indicators
- **Eviction Frequency**: Replacement policy effectiveness
- **I/O Throughput**: Disk operation efficiency

### Resource Management

Proper resource lifecycle management:

```mermaid
sequenceDiagram
participant App as Application
participant PC as PageCache
participant PF as PagedFile
participant MT as MemoryTracker
App->>PC : map(path, pageSize)
PC->>MT : reserveNative(memory)
PC->>PF : new PagedFile()
PF-->>PC : file instance
PC-->>App : pagedFile
Note over App,MT : Normal operations
App->>PF : close()
PF->>MT : releaseNative(memory)
PF-->>App : closed
App->>PC : close()
PC->>MT : releaseNative(allMemory)
PC-->>App : cache closed
```

**Diagram sources**
- [MuninnPageCache.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/MuninnPageCache.java#L735-L744)

### Troubleshooting Memory Issues

Diagnostic approaches for memory-related problems:

1. **Enable Memory Tracing**: Use detailed logging for allocation patterns
2. **Monitor Resource Usage**: Track memory consumption over time
3. **Profile Allocation Patterns**: Identify hot paths and bottlenecks
4. **Test Under Load**: Validate behavior with realistic workloads

**Section sources**
- [MuninnPageCache.java](file://community/io/src/main/java/org/neo4j/io/pagecache/impl/muninn/MuninnPageCache.java#L735-L744)