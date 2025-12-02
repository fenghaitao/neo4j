# Concurrency Control

<cite>
**Referenced Files in This Document**
- [WorkSync.java](file://community/concurrent/src/main/java/org/neo4j/util/concurrent/WorkSync.java)
- [ForsetiLockManager.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiLockManager.java)
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java)
- [SharedLock.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/SharedLock.java)
- [ExclusiveLock.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ExclusiveLock.java)
- [LockType.java](file://community/lock/src/main/java/org/neo4j/lock/LockType.java)
- [ResourceType.java](file://community/lock/src/main/java/org/neo4j/lock/ResourceType.java)
- [KernelTransactionImplementation.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/api/KernelTransactionImplementation.java)
- [RecordStorageLocks.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageLocks.java)
- [TransactionDependenciesResolver.java](file://community/cypher/runtime-util/src/main/java/org/neo4j/internal/kernel/api/helpers/TransactionDependenciesResolver.java)
- [LockAcquisitionTimeoutException.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/LockAcquisitionTimeoutException.java)
- [LockManager.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/LockManager.java)
- [EntityLocks.java](file://community/kernel-api/src/main/java/org/neo4j/internal/kernel/api/EntityLocks.java)
- [LockVerificationMonitor.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/LockVerificationMonitor.java)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [System Architecture Overview](#system-architecture-overview)
3. [Core Components](#core-components)
4. [Lock Types and Granularity](#lock-types-and-granularity)
5. [Transaction Execution Coordination](#transaction-execution-coordination)
6. [Deadlock Detection and Resolution](#deadlock-detection-and-resolution)
7. [Isolation Levels and Visibility Rules](#isolation-levels-and-visibility-rules)
8. [Performance Considerations](#performance-considerations)
9. [Common Issues and Solutions](#common-issues-and-solutions)
10. [Best Practices](#best-practices)

## Introduction

Neo4j's Concurrency Control system ensures safe access to graph data when multiple transactions operate concurrently. The system implements sophisticated locking mechanisms, deadlock detection, and isolation guarantees to maintain data consistency while maximizing throughput. At its core, the system coordinates access to graph entities (nodes, relationships, schema) through a hierarchical locking strategy that prevents conflicts while allowing optimal parallelism.

The concurrency control system operates on multiple levels:
- **Transaction Level**: Manages transaction lifecycle and coordination
- **Lock Management**: Provides fine-grained locking for graph entities
- **Deadlock Detection**: Prevents and resolves circular wait dependencies
- **Isolation Enforcement**: Maintains ACID properties through MVCC and visibility rules

## System Architecture Overview

The concurrency control system follows a layered architecture that separates concerns between transaction management, lock coordination, and deadlock prevention.

```mermaid
graph TB
subgraph "Transaction Layer"
TX[KernelTransaction]
TX_IMPL[KernelTransactionImplementation]
WORK_SYNC[WorkSync]
end
subgraph "Lock Management Layer"
LOCK_MGR[ForsetiLockManager]
CLIENT[ForsetiClient]
SHARED_LOCK[SharedLock]
EXCLUSIVE_LOCK[ExclusiveLock]
end
subgraph "Storage Layer"
STORAGE_LOCKS[RecordStorageLocks]
ENTITY_LOCKS[EntityLocks]
VERIFICATION[LockVerificationMonitor]
end
subgraph "Dependency Resolution"
DEPENDENCY_RESOLVER[TransactionDependenciesResolver]
WAIT_LIST[Wait Lists]
end
TX --> TX_IMPL
TX_IMPL --> WORK_SYNC
TX_IMPL --> LOCK_MGR
LOCK_MGR --> CLIENT
CLIENT --> SHARED_LOCK
CLIENT --> EXCLUSIVE_LOCK
TX_IMPL --> STORAGE_LOCKS
STORAGE_LOCKS --> ENTITY_LOCKS
ENTITY_LOCKS --> VERIFICATION
CLIENT --> DEPENDENCY_RESOLVER
DEPENDENCY_RESOLVER --> WAIT_LIST
```

**Diagram sources**
- [KernelTransactionImplementation.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/api/KernelTransactionImplementation.java#L188-L200)
- [ForsetiLockManager.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiLockManager.java#L106-L154)
- [WorkSync.java](file://community/concurrent/src/main/java/org/neo4j/util/concurrent/WorkSync.java#L48-L66)

## Core Components

### WorkSync: Coordinating Concurrent Operations

WorkSync serves as the primary mechanism for coordinating concurrent operations in a thread-safe manner. It transforms multiple concurrent requests into serialized batches, reducing contention and improving overall throughput.

```mermaid
classDiagram
class WorkSync {
-Material material
-AtomicReference~WorkUnit~ stack
-AtomicReference~Thread~ lock
+apply(W work) void
+applyAsync(W work) AsyncApply
-enqueueWork(W work) WorkUnit
-tryDoWork(WorkUnit unit, boolean block) Throwable
-grabBatch() WorkUnit
-doSynchronizedWork(WorkUnit batch) Throwable
}
class WorkUnit {
-W work
-Thread owner
-volatile WorkUnit next
-AtomicInteger state
+park() void
+isDone() boolean
+complete() void
+unpark() void
}
class AsyncApply {
+await() void
+tryComplete() boolean
}
WorkSync --> WorkUnit : manages
WorkSync --> AsyncApply : creates
```

**Diagram sources**
- [WorkSync.java](file://community/concurrent/src/main/java/org/neo4j/util/concurrent/WorkSync.java#L48-L279)

**Section sources**
- [WorkSync.java](file://community/concurrent/src/main/java/org/neo4j/util/concurrent/WorkSync.java#L1-L279)

### ForsetiLockManager: Advanced Lock Coordination

The ForsetiLockManager implements a sophisticated lock management system with integrated deadlock detection using the Dreadlocks algorithm. This system provides optimal scalability for high-concurrency scenarios.

```mermaid
classDiagram
class ForsetiLockManager {
-ConcurrentMap~Long, Lock~[] lockMaps
-ResourceType[] resourceTypes
-AtomicLong clientIds
-SystemNanoClock clock
-boolean verboseDeadlocks
+newClient() Client
+accept(Visitor out) void
-findMaxResourceId(ResourceType[] resourceTypes) int
}
class ForsetiClient {
-ConcurrentMap~Long, Lock~[] lockMaps
-HeapTrackingLongIntHashMap[] sharedLockCounts
-HeapTrackingLongIntHashMap[] exclusiveLockCounts
-AtomicLong activeLockCount
-Set~ForsetiClient~ waitList
-ExclusiveLock myExclusiveLock
-volatile boolean hasLocks
+acquireShared(LockTracer tracer, ResourceType resourceType, long... resourceIds) void
+acquireExclusive(LockTracer tracer, ResourceType resourceType, long... resourceIds) void
+releaseShared(ResourceType resourceType, long... resourceIds) void
+releaseExclusive(ResourceType resourceType, long... resourceIds) void
+holdsLock(long id, ResourceType resource, LockType lockType) boolean
+activeLocks() Collection~ActiveLock~
}
class Lock {
<<interface>>
+copyHolderWaitListsInto(Set waitList) void
+detectDeadlock(ForsetiClient client) ForsetiClient
+describeWaitList() String
+collectOwners(Set owners) void
+isOwnedBy(ForsetiClient client) boolean
+type() LockType
+transactionIds() LongSet
+isClosed() boolean
}
ForsetiLockManager --> ForsetiClient : creates
ForsetiClient --> Lock : manages
ForsetiLockManager --> Lock : contains
```

**Diagram sources**
- [ForsetiLockManager.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiLockManager.java#L106-L231)
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java#L75-L200)

**Section sources**
- [ForsetiLockManager.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiLockManager.java#L1-L231)
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java#L1-L200)

## Lock Types and Granularity

Neo4j implements a hierarchical locking system with different granularities to balance between concurrency and consistency.

### Lock Types

```mermaid
graph TD
LOCK_TYPES[Lock Types] --> SHARED[SHARED Lock]
LOCK_TYPES --> EXCLUSIVE[EXCLUSIVE Lock]
SHARED --> SHARED_DESC["• Allows multiple readers<br/>• Blocks writers<br/>• Used for read operations"]
EXCLUSIVE --> EXCLUSIVE_DESC["• Single writer access<br/>• Blocks all other locks<br/>• Used for write operations"]
SHARED -.-> UPGRADABLE[Can be upgraded to EXCLUSIVE]
EXCLUSIVE -.-> DOWNGRADABLE[Can be downgraded to SHARED]
```

**Diagram sources**
- [LockType.java](file://community/lock/src/main/java/org/neo4j/lock/LockType.java#L22-L36)

### Resource Types and Lock Hierarchy

The system organizes locks in a strict hierarchy to prevent deadlocks:

| Priority | Resource Type | Description | Lock Scope |
|----------|---------------|-------------|------------|
| 1 | LABEL | Schema locks for labels | Single label |
| 2 | RELATIONSHIP_TYPE | Schema locks for relationship types | Single relationship type |
| 3 | SCHEMA_NAME | Schema name locks | Hashed schema name |
| 4 | NODE_RELATIONSHIP_GROUP_DELETE | Node delete locks | Single node |
| 5 | NODE | Node data locks | Single node |
| 6 | DEGREES | Degree calculation locks | Single node |
| 7 | RELATIONSHIP_DELETE | Relationship delete locks | Single relationship |
| 8 | RELATIONSHIP_GROUP | Relationship group locks | Single node |
| 9 | RELATIONSHIP | Relationship data locks | Single relationship |

**Section sources**
- [ResourceType.java](file://community/lock/src/main/java/org/neo4j/lock/ResourceType.java#L1-L92)
- [RecordStorageLocks.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageLocks.java#L34-L70)

## Transaction Execution Coordination

### Transaction Lifecycle and Lock Acquisition

The transaction execution process involves careful coordination between WorkSync and LockService to ensure proper ordering and conflict resolution.

```mermaid
sequenceDiagram
participant Client as Client Thread
participant WorkSync as WorkSync
participant KTI as KernelTransaction
participant LockMgr as ForsetiLockManager
participant LockClient as ForsetiClient
Client->>WorkSync : apply(work)
WorkSync->>WorkSync : enqueueWork(work)
WorkSync->>WorkSync : tryLock(unit, block)
alt Lock Available
WorkSync->>WorkSync : grabBatch()
WorkSync->>WorkSync : doSynchronizedWork(batch)
WorkSync->>KTI : execute transaction
KTI->>LockMgr : newClient()
LockMgr->>LockClient : create client
LockClient->>LockClient : acquire locks
LockClient-->>WorkSync : locks acquired
WorkSync->>WorkSync : markAsDone(batch)
else Lock Contention
WorkSync->>WorkSync : park unit
WorkSync-->>Client : block until lock available
end
```

**Diagram sources**
- [WorkSync.java](file://community/concurrent/src/main/java/org/neo4j/util/concurrent/WorkSync.java#L78-L156)
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java#L175-L325)

### Lock Acquisition Patterns

The system implements several lock acquisition patterns to optimize performance and prevent deadlocks:

```mermaid
flowchart TD
START([Transaction Starts]) --> CHECK_LOCKS{Already Holding Locks?}
CHECK_LOCKS --> |Yes| INCREMENT_REF[Increment Local Reference Count]
CHECK_LOCKS --> |No| GLOBAL_LOCK[Attempt Global Lock Acquisition]
GLOBAL_LOCK --> LOCK_SUCCESS{Lock Acquired?}
LOCK_SUCCESS --> |Yes| UPDATE_LOCAL[Update Local Lock Tracking]
LOCK_SUCCESS --> |No| WAIT_FOR_LOCK[Wait for Lock Availability]
WAIT_FOR_LOCK --> DETECT_DEADLOCK[Detect Deadlock]
DETECT_DEADLOCK --> DEADLOCK_FOUND{Deadlock Detected?}
DEADLOCK_FOUND --> |Yes| THROW_EXCEPTION[Throw Deadlock Exception]
DEADLOCK_FOUND --> |No| BACKOFF_WAIT[Exponential Backoff]
BACKOFF_WAIT --> GLOBAL_LOCK
INCREMENT_REF --> SUCCESS([Continue Operation])
UPDATE_LOCAL --> SUCCESS
THROW_EXCEPTION --> END([Transaction Ends])
SUCCESS --> END
```

**Diagram sources**
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java#L302-L400)

**Section sources**
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java#L175-L503)
- [KernelTransactionImplementation.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/api/KernelTransactionImplementation.java#L591-L608)

## Deadlock Detection and Resolution

### Dreadlocks Algorithm Implementation

Neo4j uses the Dreadlocks algorithm for efficient deadlock detection with O(1) complexity. The algorithm maintains wait lists that propagate through the dependency graph.

```mermaid
graph TD
subgraph "Wait List Propagation"
A[Client A] --> B[Client B]
B --> C[Client C]
C --> A
A -.-> A_WAIT[A.waitList = [A]]
B -.-> B_WAIT[B.waitList = [B]]
C -.-> C_WAIT[C.waitList = [C]]
A -.-> UNION[Union Wait Lists]
B -.-> UNION
C -.-> UNION
UNION --> NEW_WAIT[New Wait List]
NEW_WAIT --> DETECT_CYCLE[Detect Cycle]
end
subgraph "Deadlock Verification"
DETECT_CYCLE --> REAL_DETECTION{Real Deadlock?}
REAL_DETECTION --> |Yes| THROW_DEADLOCK[Throw DeadlockDetectedException]
REAL_DETECTION --> |No| CONTINUE[Continue Waiting]
end
```

**Diagram sources**
- [ForsetiLockManager.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiLockManager.java#L62-L105)
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java#L813-L943)

### Deadlock Resolution Strategies

The system implements multiple strategies for resolving deadlocks:

1. **Priority-Based Resolution**: Transactions with higher sequence numbers are prioritized
2. **Timeout-Based Resolution**: Transactions exceeding lock acquisition timeout are terminated
3. **Random Selection**: When priorities are equal, one transaction is randomly selected for rollback

**Section sources**
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java#L813-L943)
- [LockAcquisitionTimeoutException.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/LockAcquisitionTimeoutException.java)

## Isolation Levels and Visibility Rules

### Multi-Version Concurrency Control (MVCC)

Neo4j implements MVCC to provide snapshot isolation without blocking readers. Each transaction sees a consistent snapshot of the database state.

```mermaid
graph TB
subgraph "Transaction Visibility"
TX1[Transaction 1<br/>Seq: 100] --> SNAPSHOT1[Snapshot View]
TX2[Transaction 2<br/>Seq: 101] --> SNAPSHOT2[Snapshot View]
TX3[Transaction 3<br/>Seq: 102] --> SNAPSHOT3[Snapshot View]
SNAPSHOT1 --> DATA1[Data Revision: 100]
SNAPSHOT2 --> DATA2[Data Revision: 101]
SNAPSHOT3 --> DATA3[Data Revision: 102]
end
subgraph "Visibility Boundaries"
OLDEST_VISIBLE[Oldest Visible: 100]
LAST_CLOSED[Last Closed: 102]
COMMITTING[Committing: 103]
OLDEST_VISIBLE -.-> SNAPSHOT1
LAST_CLOSED -.-> SNAPSHOT2
COMMITTING -.-> SNAPSHOT3
end
```

### Transaction State Tracking

The system maintains detailed transaction state to enforce isolation:

| State | Description | Visibility Rules |
|-------|-------------|------------------|
| Active | Transaction is running | Can see committed data and own uncommitted changes |
| Preparing | Transaction is preparing to commit | Cannot see uncommitted changes from other transactions |
| Committed | Transaction has been committed | Other transactions can see committed changes |
| Rolled Back | Transaction was rolled back | Other transactions cannot see rolled-back changes |

**Section sources**
- [KernelTransactionImplementation.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/api/KernelTransactionImplementation.java#L1560-L1563)

## Performance Considerations

### Lock Contention Optimization

The system implements several optimization strategies to minimize lock contention:

1. **Lock Ordering**: Resources are always acquired in predefined order to prevent deadlocks
2. **Lock Granularity**: Fine-grained locking reduces contention while maintaining consistency
3. **Lock Escalation**: Shared locks can be upgraded to exclusive locks atomically
4. **Lock Caching**: Local lock tracking reduces global lock map lookups

### Memory Management

The concurrency control system carefully manages memory allocation:

```mermaid
graph LR
subgraph "Memory Tracking"
MEM_TRACKER[Memory Tracker]
HEAP_ALLOC[Heap Allocation]
SCOPE_TRACK[Scoped Memory Pool]
MEM_TRACKER --> HEAP_ALLOC
HEAP_ALLOC --> SCOPE_TRACK
end
subgraph "Lock Memory"
SHARED_LOCKS[Shared Lock Counts]
EXCLUSIVE_LOCKS[Exclusive Lock Counts]
WAIT_LISTS[Wait Lists]
SHARED_LOCKS --> MEM_TRACKER
EXCLUSIVE_LOCKS --> MEM_TRACKER
WAIT_LISTS --> MEM_TRACKER
end
```

**Diagram sources**
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java#L145-L160)

**Section sources**
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java#L145-L172)

## Common Issues and Solutions

### Deadlock Scenarios

Common deadlock scenarios and their solutions:

| Scenario | Cause | Solution |
|----------|-------|----------|
| Circular Dependencies | Transactions acquire locks in different orders | Enforce strict lock ordering |
| Long-Running Transactions | Hold locks for extended periods | Implement timeouts and early lock release |
| High Contention | Many transactions competing for same resources | Optimize query patterns and reduce lock scope |
| Nested Transactions | Inner transactions hold locks unexpectedly | Proper cleanup in inner transaction handlers |

### Lock Timeout Handling

The system provides configurable timeout mechanisms:

```mermaid
flowchart TD
ACQUIRE_LOCK[Attempt Lock Acquisition] --> TIMEOUT_CHECK{Timeout Exceeded?}
TIMEOUT_CHECK --> |No| WAIT[Wait for Lock]
TIMEOUT_CHECK --> |Yes| THROW_TIMEOUT[Throw LockAcquisitionTimeoutException]
WAIT --> LOCK_GRANTED{Lock Granted?}
LOCK_GRANTED --> |Yes| SUCCESS[Continue Operation]
LOCK_GRANTED --> |No| TIMEOUT_CHECK
THROW_TIMEOUT --> LOG_ERROR[Log Timeout Error]
LOG_ERROR --> CLEANUP[Cleanup Resources]
CLEANUP --> FAIL_TRANSACTION[Fail Transaction]
```

**Diagram sources**
- [ForsetiClient.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/forseti/ForsetiClient.java#L854-L870)

### Long-Running Transaction Impacts

Long-running transactions can impact system performance:

1. **Visibility Boundaries**: Extend the oldest visible transaction boundary
2. **Lock Contention**: Hold locks for extended periods, blocking other transactions
3. **Memory Usage**: Accumulate transaction state and undo logs
4. **Checkpoint Impact**: Delay checkpoint operations due to active transactions

**Section sources**
- [LockAcquisitionTimeoutException.java](file://community/lock/src/main/java/org/neo4j/kernel/impl/locking/LockAcquisitionTimeoutException.java)
- [KernelTransactionImplementation.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/api/KernelTransactionImplementation.java#L743-L776)

## Best Practices

### Lock Acquisition Patterns

1. **Always Acquire Locks in Order**: Follow the predefined resource hierarchy
2. **Minimize Lock Hold Time**: Release locks as soon as possible
3. **Use Appropriate Lock Types**: Prefer shared locks for read operations
4. **Handle Deadlocks Gracefully**: Implement retry logic for deadlock scenarios

### Transaction Design Guidelines

1. **Keep Transactions Short**: Minimize the duration of transaction execution
2. **Batch Operations**: Use WorkSync to batch related operations
3. **Monitor Lock Metrics**: Track lock acquisition times and contention
4. **Configure Timeouts Appropriately**: Set reasonable timeouts for different operation types

### Performance Tuning

1. **Optimize Query Patterns**: Reduce the number of entities accessed in a transaction
2. **Use Indexes Effectively**: Leverage indexes to reduce lock scope
3. **Monitor System Metrics**: Track transaction throughput and lock contention
4. **Adjust Memory Settings**: Configure appropriate memory limits for transaction state

The concurrency control system in Neo4j provides a robust foundation for handling concurrent access to graph data while maintaining strong consistency guarantees. By understanding the underlying mechanisms and following established best practices, developers can build highly scalable applications that efficiently utilize Neo4j's concurrency capabilities.