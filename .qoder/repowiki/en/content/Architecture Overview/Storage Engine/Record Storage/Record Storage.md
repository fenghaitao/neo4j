# Record Storage

<cite>
**Referenced Files in This Document**
- [RecordStorageEngine.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageEngine.java)
- [RecordStorageReader.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageReader.java)
- [NodeRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/NodeRecord.java)
- [PropertyRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/PropertyRecord.java)
- [RelationshipRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/RelationshipRecord.java)
- [DynamicRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/DynamicRecord.java)
- [NodeRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/NodeRecordFormat.java)
- [PropertyRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/PropertyRecordFormat.java)
- [RelationshipRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/RelationshipRecordFormat.java)
- [DynamicRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/DynamicRecordFormat.java)
- [Record.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/Record.java)
- [AbstractDynamicStore.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/AbstractDynamicStore.java)
- [RecordStore.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/RecordStore.java)
- [RecordAccess.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordAccess.java)
- [RecordFormats.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/RecordFormats.java)
- [RecordDatabaseLayout.java](file://community/layout/src/main/java/org/neo4j/io/layout/recordstorage/RecordDatabaseLayout.java)
- [RecordDatabaseFile.java](file://community/layout/src/main/java/org/neo4j/io/layout/recordstorage/RecordDatabaseFile.java)
- [RecordStorageMigrator.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/storemigration/RecordStorageMigrator.java)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [System Architecture](#system-architecture)
3. [Core Components](#core-components)
4. [Record Formats and Layout](#record-formats-and-layout)
5. [Dynamic Record Chains](#dynamic-record-chains)
6. [Property Storage Strategies](#property-storage-strategies)
7. [Page Cache Integration](#page-cache-integration)
8. [Storage Engine Operations](#storage-engine-operations)
9. [Schema Evolution and Migration](#schema-evolution-and-migration)
10. [Performance Optimization](#performance-optimization)
11. [Common Issues and Solutions](#common-issues-and-solutions)
12. [Conclusion](#conclusion)

## Introduction

The Record Storage subsystem of Neo4j's Storage Engine provides a sophisticated, efficient mechanism for representing nodes, relationships, and properties on disk. This system implements a fixed-length record structure with intelligent overflow handling for variable-length data, enabling high-performance graph operations while maintaining data integrity and supporting schema evolution.

The subsystem operates on the principle of separating concerns between different types of data: fixed-size records for frequently accessed information (nodes, relationships) and dynamic records for variable-length data (large properties, extensive label lists). This design enables optimal memory usage patterns and efficient disk I/O operations.

## System Architecture

The Record Storage subsystem follows a layered architecture that separates storage concerns from business logic:

```mermaid
graph TB
subgraph "Application Layer"
SE[StorageEngine]
SR[StorageReader]
SC[StorageCursors]
end
subgraph "Record Storage Layer"
RSE[RecordStorageEngine]
RSA[RecordStorageAccess]
RC[RecordCursor]
end
subgraph "Record Format Layer"
NF[NodeRecordFormat]
RF[RelationshipRecordFormat]
PF[PropertyRecordFormat]
DF[DynamicRecordFormat]
end
subgraph "Storage Layer"
NS[NodeStore]
RS[RelationshipStore]
PS[PropertyStore]
DS[DynamicStore]
end
subgraph "Page Cache Layer"
PC[PageCache]
PCU[PageCursor]
end
subgraph "File System"
FS[File System]
FNS[Node Store File]
FRS[Relationship Store File]
FPS[Property Store File]
end
SE --> RSE
SR --> RSA
SC --> RC
RSE --> NF
RSE --> RF
RSE --> PF
RSE --> DF
NF --> NS
RF --> RS
PF --> PS
DF --> DS
NS --> PC
RS --> PC
PS --> PC
DS --> PC
PC --> PCU
PCU --> FS
FS --> FNS
FS --> FRS
FS --> FPS
```

**Diagram sources**
- [RecordStorageEngine.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageEngine.java#L142-L200)
- [RecordStorageReader.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageReader.java#L60-L87)

**Section sources**
- [RecordStorageEngine.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageEngine.java#L142-L200)
- [RecordStorageReader.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageReader.java#L57-L87)

## Core Components

### RecordStorageEngine

The central orchestrator of the record storage system, managing all storage operations and coordinating between different store components.

```mermaid
classDiagram
class RecordStorageEngine {
-NeoStores neoStores
-RecordDatabaseLayout databaseLayout
-Config config
-TokenHolders tokenHolders
-SchemaCache schemaCache
-CountsStore countsStore
-RelationshipGroupDegreesStore groupDegreesStore
+newReader() RecordStorageReader
+createCommands() StorageCommand[]
+apply() void
+checkpoint() void
+init() void
+start() void
+stop() void
+shutdown() void
}
class NeoStores {
+getNodeStore() NodeStore
+getRelationshipStore() RelationshipStore
+getPropertyStore() PropertyStore
+getSchemaStore() SchemaStore
+getMetaDataStore() MetaDataStore
}
class RecordStorageReader {
-TokenHolders tokenHolders
-NodeStore nodeStore
-RelationshipStore relationshipStore
-PropertyStore propertyStore
-CountsStore counts
-SchemaCache schemaCache
+allocateNodeCursor() RecordNodeCursor
+allocatePropertyCursor() StoragePropertyCursor
+allNodeScan() AllNodeScan
+nodesGetCount() long
}
RecordStorageEngine --> NeoStores
RecordStorageEngine --> RecordStorageReader
```

**Diagram sources**
- [RecordStorageEngine.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageEngine.java#L142-L200)
- [RecordStorageReader.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageReader.java#L60-L87)

### Domain Model Components

The system defines several key record types that represent different aspects of graph data:

```mermaid
classDiagram
class AbstractBaseRecord {
<<abstract>>
-long id
-long secondaryUnitId
-boolean requiresSecondaryUnit
-boolean inUse
-boolean created
-boolean useFixedReferences
+getId() long
+inUse() boolean
+isCreated() boolean
+setInUse(boolean) void
+clear() void
}
class NodeRecord {
-long nextRel
-long labels
-DynamicRecord[] dynamicLabelRecords
-boolean isLight
-boolean dense
+getNextRel() long
+setLabelField(long, List) void
+isDense() boolean
+getDynamicLabelRecords() DynamicRecord[]
}
class RelationshipRecord {
-long firstNode
-long secondNode
-int type
-long firstPrevRel
-long firstNextRel
-long secondPrevRel
-long secondNextRel
-boolean firstInFirstChain
-boolean firstInSecondChain
+getFirstNode() long
+getSecondNode() long
+getType() int
+setLinks(long, long, int) void
+setFirstInChain(boolean, long) void
}
class PropertyRecord {
-long nextProp
-long prevProp
-long[] blocks
-PropertyBlock[] blockRecords
-int blocksCursor
-int blockRecordsCursor
-boolean blocksLoaded
-long entityId
-byte entityType
-DynamicRecord[] deletedRecords
+addPropertyBlock(PropertyBlock) void
+removePropertyBlock(int) PropertyBlock
+propertyBlocks() Iterable~PropertyBlock~
+ensureBlocksLoaded() void
+size() int
}
class DynamicRecord {
-byte[] data
-long nextBlock
-int type
-boolean startRecord
+getData() byte[]
+setStartRecord(boolean) void
+isStartRecord() boolean
+getType() PropertyType
}
AbstractBaseRecord <|-- NodeRecord
AbstractBaseRecord <|-- RelationshipRecord
AbstractBaseRecord <|-- PropertyRecord
AbstractBaseRecord <|-- DynamicRecord
```

**Diagram sources**
- [NodeRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/NodeRecord.java#L32-L42)
- [RelationshipRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/RelationshipRecord.java#L29-L42)
- [PropertyRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/PropertyRecord.java#L46-L82)
- [DynamicRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/DynamicRecord.java#L33-L51)

**Section sources**
- [NodeRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/NodeRecord.java#L32-L165)
- [RelationshipRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/RelationshipRecord.java#L29-L289)
- [PropertyRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/PropertyRecord.java#L46-L426)
- [DynamicRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/DynamicRecord.java#L33-L79)

## Record Formats and Layout

### Fixed-Length Record Structure

Each record type implements a specific format optimized for its data characteristics:

| Record Type | Size | Purpose | Key Fields |
|-------------|------|---------|------------|
| NodeRecord | 15 bytes | Node metadata | nextRel, nextProp, labels, dense flag |
| RelationshipRecord | 34 bytes | Relationship metadata | firstNode, secondNode, type, chains |
| PropertyRecord | 41 bytes | Property containers | prevProp, nextProp, property blocks |
| DynamicRecord | Variable | Variable data | data payload, nextBlock |

### Bit-Level Encoding

The system employs sophisticated bit-level encoding to maximize storage efficiency:

```mermaid
flowchart TD
A[Record Header] --> B{In Use Flag}
B --> |1| C[Load Record Data]
B --> |0| D[Skip to Next Offset]
C --> E{Record Type}
E --> |Node| F[Parse Node Fields]
E --> |Relationship| G[Parse Relationship Fields]
E --> |Property| H[Parse Property Blocks]
E --> |Dynamic| I[Parse Dynamic Payload]
F --> J[Extract High Bits]
G --> K[Extract Chain Links]
H --> L[Extract Property Types]
I --> M[Extract Data Length]
J --> N[Combine with Low Bits]
K --> O[Combine with Low Bits]
L --> P[Parse Block Values]
M --> Q[Read Data Bytes]
N --> R[Complete Record]
O --> R
P --> R
Q --> R
```

**Diagram sources**
- [NodeRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/NodeRecordFormat.java#L47-L81)
- [RelationshipRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/RelationshipRecordFormat.java#L55-L116)

### Secondary Unit Support

For records that exceed their fixed size, the system supports secondary units:

```mermaid
sequenceDiagram
participant App as Application
participant RS as RecordStorage
participant RF as RecordFormat
participant PC as PageCache
App->>RS : Request Large Property
RS->>RF : Check Record Capacity
RF->>RF : Calculate Required Units
alt Requires Secondary Unit
RF->>PC : Allocate Secondary Unit
PC-->>RF : Secondary Unit ID
RF->>RF : Set Secondary Unit Flags
end
RF-->>RS : Complete Record
RS-->>App : Record with Secondary Unit
```

**Diagram sources**
- [Record.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/Record.java#L38-L65)

**Section sources**
- [NodeRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/NodeRecordFormat.java#L29-L116)
- [RelationshipRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/RelationshipRecordFormat.java#L29-L186)
- [PropertyRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/PropertyRecordFormat.java#L32-L171)
- [DynamicRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/DynamicRecordFormat.java#L31-L164)

## Dynamic Record Chains

### Variable-Length Data Handling

Large properties and extensive label lists are stored using dynamic record chains:

```mermaid
graph LR
subgraph "Dynamic Record Chain"
DR1[DynamicRecord 1<br/>Start Record<br/>Type: STRING<br/>Data: "Hello World..."]
DR2[DynamicRecord 2<br/>Linked Record<br/>Type: STRING<br/>Data: "... continuation"]
DR3[DynamicRecord 3<br/>Linked Record<br/>Type: STRING<br/>Data: "... more data"]
DR1 --> |nextBlock| DR2
DR2 --> |nextBlock| DR3
DR3 --> |nextBlock| NULL[NULL Reference]
end
subgraph "Property Block"
PB[PropertyBlock<br/>Key: name<br/>Type: STRING<br/>Value: Reference to DR1]
end
PB -.-> DR1
```

**Diagram sources**
- [DynamicRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/DynamicRecord.java#L33-L79)
- [AbstractDynamicStore.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/AbstractDynamicStore.java#L117-L209)

### Chain Management

The system maintains integrity through careful chain management:

```mermaid
flowchart TD
A[Read Property] --> B{Has Dynamic Records?}
B --> |No| C[Use Inline Data]
B --> |Yes| D[Follow Chain]
D --> E{Record In Use?}
E --> |No| F[Log Inconsistent Chain]
E --> |Yes| G{More Records?}
G --> |Yes| H[Read Next Record]
G --> |No| I[Complete Chain]
H --> E
F --> J[Continue with Available Data]
I --> K[Return Complete Value]
J --> K
C --> K
```

**Diagram sources**
- [AbstractDynamicStore.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/AbstractDynamicStore.java#L176-L209)

**Section sources**
- [DynamicRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/DynamicRecord.java#L33-L79)
- [AbstractDynamicStore.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/AbstractDynamicStore.java#L117-L313)

## Property Storage Strategies

### Inlining vs Overflow

The system intelligently chooses between inline storage and dynamic overflow:

| Property Type | Inline Size | Storage Method | Use Case |
|---------------|-------------|----------------|----------|
| BOOLEAN | 8 bytes | Inline | Small boolean values |
| LONG | 8 bytes | Inline | Integer values |
| FLOAT | 8 bytes | Inline | Floating-point numbers |
| STRING | 23 bytes | Inline | Short strings |
| STRING | >23 bytes | Dynamic | Long strings |
| ARRAY | 120 bytes | Inline | Small arrays |
| ARRAY | >120 bytes | Dynamic | Large arrays |

### Property Block Organization

Property records contain multiple property blocks organized for efficient access:

```mermaid
classDiagram
class PropertyRecord {
-long nextProp
-long prevProp
-long[] blocks
-PropertyBlock[] blockRecords
-int blocksCursor
-int blockRecordsCursor
-boolean blocksLoaded
-long entityId
-byte entityType
+addPropertyBlock(PropertyBlock) void
+removePropertyBlock(int) PropertyBlock
+propertyBlocks() Iterable~PropertyBlock~
+ensureBlocksLoaded() void
+size() int
}
class PropertyBlock {
-long[] valueBlocks
-int keyIndexId
-int propertyBlockId
+getValueBlocks() long[]
+getKeyIndexId() int
+getSize() int
+setValueBlocks(long[]) void
}
class PropertyType {
<<enumeration>>
BOOLEAN
BYTE
SHORT
INT
LONG
FLOAT
DOUBLE
STRING
ARRAY
POINT
DATE
TIME
LOCAL_TIME
DATE_TIME
LOCAL_DATE_TIME
ZONED_DATE_TIME
DURATION
GEOMETRY
+calculateNumberOfBlocksUsed(long) int
+getPayloadSize() int
+getPayloadSizeLongs() int
}
PropertyRecord --> PropertyBlock
PropertyBlock --> PropertyType
```

**Diagram sources**
- [PropertyRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/PropertyRecord.java#L46-L82)

**Section sources**
- [PropertyRecord.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/record/PropertyRecord.java#L46-L426)
- [PropertyRecordFormat.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/format/standard/PropertyRecordFormat.java#L32-L171)

## Page Cache Integration

### Efficient Memory Management

The record storage system integrates seamlessly with Neo4j's page cache for optimal performance:

```mermaid
sequenceDiagram
participant App as Application
participant RS as RecordStorage
participant PC as PageCache
participant FS as FileSystem
App->>RS : Request Node Record
RS->>PC : Pin Page
PC->>FS : Read Page (if not cached)
FS-->>PC : Page Data
PC-->>RS : Page Reference
RS->>RS : Parse Record from Page
RS-->>App : Node Record
Note over RS,PC : Page remains pinned until explicitly released
App->>RS : Modify Node Record
RS->>PC : Mark Page Dirty
PC-->>RS : Acknowledge
RS-->>App : Success
App->>RS : Release Page
RS->>PC : Unpin Page
PC->>PC : Eviction Policy Check
```

**Diagram sources**
- [RecordStore.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/RecordStore.java#L33-L176)

### Cursor-Based Access

The system provides cursor-based access for efficient record traversal:

```mermaid
classDiagram
class RecordCursor {
<<interface>>
+next() boolean
+get() AbstractBaseRecord
+close() void
}
class NodeCursor {
+node() NodeRecord
+labels() Iterable~Integer~
+hasLabel(int) boolean
+properties() PropertyCursor
}
class PropertyCursor {
+propertyKey() int
+value() Value
+next() boolean
}
class RelationshipCursor {
+relationship() RelationshipRecord
+type() int
+startNode() long
+endNode() long
+properties() PropertyCursor
}
RecordCursor <|-- NodeCursor
RecordCursor <|-- PropertyCursor
RecordCursor <|-- RelationshipCursor
```

**Section sources**
- [RecordStore.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/store/RecordStore.java#L33-L176)

## Storage Engine Operations

### CRUD Operations

The RecordStorageEngine supports comprehensive CRUD operations:

```mermaid
flowchart TD
A[Storage Operation] --> B{Operation Type}
B --> |Create| C[Generate ID]
B --> |Read| D[Locate Record]
B --> |Update| E[Modify Record]
B --> |Delete| F[Mark In Use False]
C --> G[Allocate Space]
D --> H[Page Cache Lookup]
E --> I[Update Record]
F --> J[Free Resources]
G --> K[Write to Store]
H --> L{Page Found?}
I --> K
J --> K
L --> |Yes| M[Parse from Cache]
L --> |No| N[Read from Disk]
M --> O[Apply Changes]
N --> O
O --> P[Flush to Disk]
K --> Q[Success]
P --> Q
```

**Diagram sources**
- [RecordStorageEngine.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageEngine.java#L461-L528)

### Transaction Support

The system provides ACID guarantees through transactional operations:

```mermaid
sequenceDiagram
participant TX as Transaction
participant RSE as RecordStorageEngine
participant RA as RecordAccess
participant LS as LogService
participant CS as CheckpointService
TX->>RSE : Begin Transaction
RSE->>RA : Create RecordAccess
RA-->>RSE : Access Instance
TX->>RSE : Create Node
RSE->>RA : Create Record
RA-->>RSE : Node Record
TX->>RSE : Add Properties
RSE->>RA : Modify Record
RA-->>RSE : Modified Record
TX->>RSE : Commit Transaction
RSE->>LS : Write Log Entries
LS-->>RSE : Log Success
RSE->>CS : Trigger Checkpoint
CS-->>RSE : Checkpoint Complete
RSE-->>TX : Commit Success
```

**Section sources**
- [RecordStorageEngine.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordStorageEngine.java#L461-L528)
- [RecordAccess.java](file://community/record-storage-engine/src/main/java/org/neo4j/internal/recordstorage/RecordAccess.java#L32-L110)

## Schema Evolution and Migration

### Format Versioning

The system supports multiple record format versions with backward compatibility:

```mermaid
graph TD
A[Schema Evolution] --> B{Migration Required?}
B --> |Yes| C[Detect Old Format]
B --> |No| D[Use Current Format]
C --> E[Determine Migration Strategy]
E --> F{Migration Type}
F --> |Online| G[Live Migration]
F --> |Offline| H[Batch Migration]
G --> I[Incremental Format Updates]
H --> J[Full Store Rebuild]
I --> K[Validate Compatibility]
J --> K
D --> K
K --> L[Update Store Headers]
L --> M[Complete Migration]
```

**Diagram sources**
- [RecordStorageMigrator.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/storemigration/RecordStorageMigrator.java#L149-L875)

### Capability-Based Migration

The system uses capability detection for intelligent migration decisions:

| Capability | Description | Migration Impact |
|------------|-------------|------------------|
| SECONDARY_RECORD_UNITS | Support for secondary units | May require format conversion |
| LITTLE_ENDIAN | Little-endian format support | Binary compatibility check |
| MULTI_VERSIONED | MVCC support | Transaction model change |
| RELATIONSHIP_TYPE_3BYTES | Extended relationship types | Schema definition update |

**Section sources**
- [RecordStorageMigrator.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/storemigration/RecordStorageMigrator.java#L149-L875)

## Performance Optimization

### Storage Efficiency Techniques

The system employs several optimization strategies:

1. **Bit Packing**: Compact representation of boolean flags and small integers
2. **Inline Storage**: Small values stored directly in record headers
3. **Compression**: Variable-length encoding for large numbers
4. **Caching**: Intelligent page caching with prefetching
5. **Batch Operations**: Bulk processing for improved throughput

### Memory Layout Optimization

```mermaid
graph LR
subgraph "Record Layout Optimization"
A[Fixed Header] --> B[Variable Data]
B --> C[Padding]
A1[In Use Flag] --> A2[High Bits]
A2 --> A3[Record Type]
B1[Property Blocks] --> B2[Dynamic References]
B2 --> B3[Label Lists]
C1[Page Alignment] --> C2[Cache Efficiency]
end
subgraph "Access Patterns"
D[Sequential Access] --> E[Random Access]
E --> F[Scattered Access]
D1[Node Scans] --> D2[Relationship Traversal]
E1[Point Queries] --> E2[Index Lookups]
F1[Join Operations] --> F2[Graph Algorithms]
end
```

### Performance Monitoring

The system provides comprehensive performance metrics:

| Metric | Description | Optimization Target |
|--------|-------------|-------------------|
| Record Cache Hit Rate | Percentage of cache hits | >95% |
| Average Record Size | Mean size of stored records | Minimize fragmentation |
| Dynamic Chain Length | Average chain depth | <3 records |
| Page Fault Rate | Frequency of disk reads | Optimize access patterns |

## Common Issues and Solutions

### Record Fragmentation

**Problem**: Over time, records become fragmented due to frequent updates and deletions.

**Solution**: The system implements automatic compaction during checkpoints and idle periods.

```mermaid
flowchart TD
A[Fragmentation Detected] --> B[Identify Affected Areas]
B --> C[Plan Compaction]
C --> D[Acquire Locks]
D --> E[Read Affected Records]
E --> F[Compact Data]
F --> G[Write New Layout]
G --> H[Update References]
H --> I[Release Locks]
I --> J[Verify Integrity]
J --> K[Complete Compaction]
```

### Storage Overhead

**Problem**: Excessive storage overhead for small properties.

**Solution**: Automatic inlining of small values and intelligent threshold management.

### Schema Evolution Challenges

**Problem**: Maintaining compatibility during schema changes.

**Solution**: Capability-based migration with fallback strategies and validation.

### Performance Degradation

**Problem**: Slow performance due to inefficient access patterns.

**Solution**: Adaptive caching, prefetching, and query optimization.

**Section sources**
- [RecordStorageMigrator.java](file://community/record-storage-engine/src/main/java/org/neo4j/kernel/impl/storemigration/RecordStorageMigrator.java#L149-L875)

## Conclusion

The Record Storage subsystem of Neo4j's Storage Engine represents a sophisticated approach to graph data persistence. Through its combination of fixed-length records for frequently accessed data and dynamic record chains for variable-length content, it achieves an optimal balance between performance and flexibility.

Key strengths of the system include:

- **Efficient Storage**: Bit-level packing and intelligent inlining minimize storage overhead
- **Scalable Design**: Dynamic record chains handle arbitrarily large data while maintaining performance
- **Schema Evolution**: Capability-based migration ensures smooth upgrades and compatibility
- **Performance Optimization**: Advanced caching, prefetching, and batch operations maximize throughput
- **Reliability**: Comprehensive error handling and validation ensure data integrity

The modular architecture separates concerns effectively, making the system maintainable and extensible. The integration with Neo4j's page cache and transaction system provides enterprise-grade reliability and performance.

For developers working with Neo4j, understanding these storage mechanisms provides valuable insights into optimizing graph applications and troubleshooting performance issues. The system's design principles of efficiency, scalability, and reliability serve as excellent examples for building high-performance data storage systems.