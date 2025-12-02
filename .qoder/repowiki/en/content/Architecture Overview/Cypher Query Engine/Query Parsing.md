# Query Parsing

<cite>
**Referenced Files in This Document**
- [CachingPreParser.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/CachingPreParser.scala)
- [InputQuery.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/InputQuery.scala)
- [CypherParsing.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/CypherParsing.scala)
- [ExecutionEngine.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/ExecutionEngine.scala)
- [PreParsedStatement.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/PreParsedStatement.scala)
- [CypherQueryCaches.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/cache/CypherQueryCaches.scala)
- [CompilationPhases.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/phases/CompilationPhases.scala)
- [QueryCache.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/QueryCache.scala)
- [MasterCompiler.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/MasterCompiler.scala)
- [CypherPlanner.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/planning/CypherPlanner.scala)
- [CompilerLibrary.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/CompilerLibrary.scala)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Core Components](#core-components)
4. [Parsing Pipeline Implementation](#parsing-pipeline-implementation)
5. [Caching Mechanisms](#caching-mechanisms)
6. [Domain Model](#domain-model)
7. [Query Parameterization](#query-parameterization)
8. [Performance Optimization](#performance-optimization)
9. [Error Handling](#error-handling)
10. [Best Practices](#best-practices)
11. [Troubleshooting Guide](#troubleshooting-guide)

## Introduction

The Cypher query parsing component in Neo4j represents a sophisticated system designed to efficiently convert raw Cypher text queries into structured representations suitable for execution. This component serves as the critical first stage in the query processing pipeline, transforming human-readable Cypher statements into optimized execution plans.

The parsing system implements a multi-layered approach combining lexical analysis, syntactic parsing, semantic analysis, and optimization techniques to deliver high-performance query processing. At its core lies the CachingPreParser, which optimizes repeated parsing operations through intelligent caching strategies, while maintaining thread safety and memory efficiency.

## Architecture Overview

The query parsing architecture follows a layered design pattern with clear separation of concerns:

```mermaid
graph TB
subgraph "Query Input Layer"
A[Raw Cypher Text] --> B[PreParser]
B --> C[PreParsedQuery]
end
subgraph "Parsing Pipeline"
C --> D[AST Generation]
D --> E[Semantic Analysis]
E --> F[Logical Planning]
end
subgraph "Optimization Layer"
F --> G[Query Caching]
G --> H[Execution Planning]
end
subgraph "Cache Management"
I[PreParser Cache] --> J[AST Cache]
J --> K[Logical Plan Cache]
K --> L[Executable Query Cache]
end
G -.-> I
H -.-> M[Runtime Execution]
```

**Diagram sources**
- [CachingPreParser.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/CachingPreParser.scala#L61-L121)
- [CypherParsing.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/CypherParsing.scala#L56-L99)
- [CypherQueryCaches.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/cache/CypherQueryCaches.scala#L461-L591)

## Core Components

### CachingPreParser

The CachingPreParser serves as the primary entry point for query preprocessing, implementing intelligent caching to optimize repeated parsing operations:

```mermaid
classDiagram
class CachingPreParser {
+configuration : CypherConfiguration
+preParserCache : LFUCache[String, PreParsedQuery]
+clearCache() : Long
+insertIntoCache(queryText : String, preParsedQuery : PreParsedQuery) : Unit
+preParseQuery(queryText : String, notificationLogger : InternalNotificationLogger, profile : Boolean, couldContainSensitiveFields : Boolean, targetsComposite : Boolean) : PreParsedQuery
}
class PreParser {
+configuration : CypherConfiguration
+preParse(queryText : String, notificationLogger : InternalNotificationLogger) : PreParsedQuery
}
class PreParsedQuery {
+statement : String
+rawStatement : String
+options : QueryOptions
+additionalNotifications : Seq[InternalNotification]
+cacheKey : InputQuery.CacheKey
+rawPreparserOptions : String
}
CachingPreParser --|> PreParser
CachingPreParser --> PreParsedQuery
```

**Diagram sources**
- [CachingPreParser.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/CachingPreParser.scala#L61-L121)
- [InputQuery.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/InputQuery.scala#L73-L95)

### Execution Engine Integration

The ExecutionEngine orchestrates the entire query processing workflow:

```mermaid
sequenceDiagram
participant Client
participant ExecutionEngine
participant CachingPreParser
participant CypherParsing
participant MasterCompiler
Client->>ExecutionEngine : execute(query, params, context)
ExecutionEngine->>CachingPreParser : preParseQuery(query, logger)
CachingPreParser->>CachingPreParser : check cache
alt Cache Miss
CachingPreParser->>CachingPreParser : parse query
CachingPreParser->>CachingPreParser : insert into cache
end
CachingPreParser-->>ExecutionEngine : PreParsedQuery
ExecutionEngine->>CypherParsing : parseQuery(...)
CypherParsing-->>ExecutionEngine : BaseState
ExecutionEngine->>MasterCompiler : compile(...)
MasterCompiler-->>ExecutionEngine : ExecutableQuery
ExecutionEngine-->>Client : QueryExecution
```

**Diagram sources**
- [ExecutionEngine.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/ExecutionEngine.scala#L181-L200)
- [CachingPreParser.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/CachingPreParser.scala#L98-L120)

**Section sources**
- [CachingPreParser.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/CachingPreParser.scala#L61-L121)
- [ExecutionEngine.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/ExecutionEngine.scala#L66-L100)

## Parsing Pipeline Implementation

### Frontend Compilation Phases

The parsing pipeline consists of multiple distinct phases, each responsible for specific aspects of query processing:

```mermaid
flowchart TD
A[Raw Query Text] --> B[Parse Phase]
B --> C[Syntax Validation]
C --> D[AST Construction]
D --> E[Semantic Analysis]
E --> F[AST Rewriting]
F --> G[Logical Planning]
G --> H[Physical Planning]
H --> I[Execution Planning]
subgraph "Parse Phase"
B1[Lexical Analysis] --> B2[Syntactic Parsing]
B2 --> B3[Error Recovery]
end
subgraph "Semantic Analysis"
E1[Type Checking] --> E2[Scope Resolution]
E2 --> E3[Validation]
end
B --> B1
E --> E1
```

**Diagram sources**
- [CompilationPhases.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/phases/CompilationPhases.scala#L75-L175)

### AST Generation and Transformation

The Abstract Syntax Tree (AST) generation process transforms parsed queries into structured representations:

```mermaid
graph LR
subgraph "AST Nodes"
A[Statement] --> B[Clause]
B --> C[Expression]
C --> D[Pattern]
D --> E[Property]
end
subgraph "Transformation Pipeline"
F[AST Rewriting] --> G[Semantic Analysis]
G --> H[Optimization]
H --> I[Code Generation]
end
A --> F
```

**Section sources**
- [CompilationPhases.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/phases/CompilationPhases.scala#L75-L142)

## Caching Mechanisms

### Multi-Level Cache Architecture

The caching system implements a sophisticated multi-level architecture designed for optimal performance:

```mermaid
graph TB
subgraph "Cache Hierarchy"
A[PreParser Cache] --> B[AST Cache]
B --> C[Logical Plan Cache]
C --> D[Execution Plan Cache]
D --> E[Executable Query Cache]
end
subgraph "Cache Types"
F[LFU Cache] --> G[Soft Cache]
G --> H[Stale Detection]
H --> I[Eviction Policy]
end
A -.-> F
B -.-> F
C -.-> F
D -.-> F
E -.-> F
```

**Diagram sources**
- [CypherQueryCaches.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/cache/CypherQueryCaches.scala#L461-L591)

### Cache Key Generation

Cache keys are carefully constructed to ensure proper cache coherency:

| Cache Level | Key Components | Purpose |
|-------------|---------------|---------|
| PreParser | Raw query text | Prevents redundant parsing |
| AST | Parsed query + parameter types | Ensures semantic correctness |
| Logical Plan | Statement + query options + parameters | Supports parameterized queries |
| Execution Plan | Logical plan + runtime context | Optimizes physical execution |
| Executable Query | Complete compilation state | Enables direct execution |

**Section sources**
- [CypherQueryCaches.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/cache/CypherQueryCaches.scala#L216-L226)
- [QueryCache.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/QueryCache.scala#L642-L693)

## Domain Model

### InputQuery Hierarchy

The domain model defines a clear hierarchy of query representations:

```mermaid
classDiagram
class InputQuery {
<<trait>>
+options : QueryOptions
+description : String
+cacheKey : CacheKey
+notifications : Seq[InternalNotification]
}
class PreParsedQuery {
+statement : String
+rawStatement : String
+options : QueryOptions
+additionalNotifications : Seq[InternalNotification]
+cacheKey : CacheKey
+cacheKeyWithRawStatement : CacheKey
+rawPreparserOptions : String
}
class FullyParsedQuery {
+state : BaseState
+options : QueryOptions
+description : String
+cacheKey : CacheKey
}
class QueryOptions {
+offset : InputPosition
+queryOptions : CypherQueryOptions
+recompilationLimitReached : Boolean
+materializedEntitiesMode : Boolean
+cacheKey : String
+executionPlanCacheKey : String
+logicalPlanCacheKey : String
}
InputQuery <|-- PreParsedQuery
InputQuery <|-- FullyParsedQuery
InputQuery --> QueryOptions
```

**Diagram sources**
- [InputQuery.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/InputQuery.scala#L35-L182)

### Query Options Management

Query options control various aspects of query execution and optimization:

| Option Category | Configuration | Impact |
|----------------|---------------|---------|
| Execution Mode | `EXPLAIN`, `PROFILE`, `NORMAL` | Controls query execution behavior |
| Planner Selection | `cost`, `idp`, `dp` | Determines optimization strategy |
| Runtime Engine | `interpreted`, `slotted`, `compiled` | Affects execution performance |
| Expression Engine | `interpreted`, `compiled`, `onlyWhenHot` | Controls code generation |
| Parameter Handling | `extractLiterals`, `obfuscateLiterals` | Security and performance tuning |

**Section sources**
- [InputQuery.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/InputQuery.scala#L117-L172)

## Query Parameterization

### Parameter Binding Process

The parameterization system ensures type-safe query execution with proper validation:

```mermaid
sequenceDiagram
participant Client
participant ExecutionEngine
participant ParameterValidator
participant QueryCompiler
Client->>ExecutionEngine : execute(query, params, context)
ExecutionEngine->>ParameterValidator : validateParameters(queryParams, givenParams)
ParameterValidator->>ParameterValidator : checkRequiredParams()
ParameterValidator->>ParameterValidator : validateTypes()
ParameterValidator-->>ExecutionEngine : validation result
ExecutionEngine->>QueryCompiler : compileWithParameters(params)
QueryCompiler-->>ExecutionEngine : ExecutableQuery
ExecutionEngine-->>Client : QueryExecution
```

**Diagram sources**
- [ExecutionEngine.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/ExecutionEngine.scala#L485-L497)

### Parameter Type Extraction

The system automatically extracts parameter types for optimization:

```mermaid
flowchart TD
A[Query Parameters] --> B[Type Inference]
B --> C[Parameter Type Map]
C --> D[Cache Key Generation]
D --> E[Query Optimization]
subgraph "Type Extraction Strategies"
F[Always Extract] --> G[Extract When No Parameters]
G --> H[Never Extract]
end
B --> F
```

**Section sources**
- [ExecutionEngine.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/ExecutionEngine.scala#L485-L497)

## Performance Optimization

### Parsing Overhead Reduction

Several strategies minimize parsing overhead for frequently executed queries:

1. **Intelligent Caching**: Multi-level cache hierarchy prevents redundant parsing
2. **Lazy Evaluation**: AST construction occurs only when needed
3. **Parameter Type Optimization**: Efficient parameter type extraction reduces compilation time
4. **Soft Cache Support**: Memory-efficient cache eviction policies

### Cache Utilization Patterns

| Pattern | Use Case | Benefits |
|---------|----------|----------|
| Hot Query Caching | Frequently executed queries | Eliminates parsing overhead |
| Parameterized Queries | Dynamic parameter values | Maintains cache effectiveness |
| Soft Cache Eviction | Memory pressure scenarios | Prevents OOM conditions |
| Stale Detection | Schema change scenarios | Ensures query correctness |

**Section sources**
- [CypherQueryCaches.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/cache/CypherQueryCaches.scala#L486-L510)

## Error Handling

### Syntax Error Management

The parsing system implements comprehensive error handling:

```mermaid
flowchart TD
A[Parsing Error] --> B{Error Type}
B --> |Lexical| C[Token Manager Error]
B --> |Syntactic| D[Parser Error]
B --> |Semantic| E[Semantic Analysis Error]
C --> F[Error Positioning]
D --> F
E --> F
F --> G[Error Reporting]
G --> H[User Notification]
G --> I[Stack Trace Generation]
```

### Common Parsing Issues

| Issue Category | Symptoms | Solutions |
|---------------|----------|-----------|
| Syntax Errors | Unexpected token messages | Validate query structure |
| Parameter Binding | Type mismatch errors | Check parameter types |
| Cache Misses | Performance degradation | Optimize cache configuration |
| Memory Issues | OutOfMemoryError | Adjust cache sizes |

**Section sources**
- [CachingPreParser.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/CachingPreParser.scala#L131-L141)

## Best Practices

### Query Design Guidelines

1. **Use Parameterized Queries**: Always use parameters instead of embedding literals
2. **Optimize Cache Usage**: Structure queries to maximize cache hit rates
3. **Minimize Query Complexity**: Break complex queries into simpler components
4. **Monitor Cache Performance**: Track cache hit ratios and adjust accordingly

### Performance Tuning Recommendations

1. **Configure Appropriate Cache Sizes**: Balance memory usage with performance benefits
2. **Enable Soft Caching**: Use soft cache eviction for memory-constrained environments
3. **Monitor Stale Cache Detection**: Ensure proper staleness detection for schema changes
4. **Profile Query Execution**: Use EXPLAIN and PROFILE modes for optimization

### Memory Management

1. **Monitor Cache Growth**: Track cache sizes and growth patterns
2. **Implement Cache Warming**: Preload frequently used queries
3. **Configure Eviction Policies**: Set appropriate eviction strategies
4. **Handle Memory Pressure**: Implement graceful degradation under memory constraints

## Troubleshooting Guide

### Common Issues and Solutions

#### High Parsing Overhead

**Symptoms**: Slow query execution, high CPU usage during parsing
**Causes**: 
- Frequent cache misses
- Large query variations
- Inefficient parameterization

**Solutions**:
- Review query patterns for commonality
- Optimize parameter usage
- Increase cache sizes
- Implement query batching

#### Cache Inefficiency

**Symptoms**: Low cache hit ratios, poor performance scaling
**Causes**:
- Poor cache key design
- Excessive query variation
- Memory pressure

**Solutions**:
- Standardize query patterns
- Use consistent parameter naming
- Monitor cache statistics
- Adjust cache eviction policies

#### Memory Issues

**Symptoms**: OutOfMemoryError, slow garbage collection
**Causes**:
- Cache sizes too large
- Memory leaks in cache entries
- Stale cache accumulation

**Solutions**:
- Reduce cache sizes
- Enable soft caching
- Monitor cache growth
- Implement cache cleanup

### Diagnostic Tools

The system provides several diagnostic capabilities:

1. **Cache Statistics**: Monitor hit ratios and cache sizes
2. **Query Tracing**: Track query execution phases
3. **Performance Metrics**: Measure parsing and compilation times
4. **Error Logging**: Detailed error reporting and stack traces

**Section sources**
- [CypherQueryCaches.scala](file://community/cypher/cypher/src/main/scala/org/neo4j/cypher/internal/cache/CypherQueryCaches.scala#L598-L630)