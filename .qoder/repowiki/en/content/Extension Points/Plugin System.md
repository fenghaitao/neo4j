# Plugin System

<cite>
**Referenced Files in This Document**
- [ExtensionFactory.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/ExtensionFactory.java)
- [AbstractExtensions.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/AbstractExtensions.java)
- [GlobalExtensions.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/GlobalExtensions.java)
- [DatabaseExtensions.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/DatabaseExtensions.java)
- [ExtensionType.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/ExtensionType.java)
- [ExtensionContext.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/context/ExtensionContext.java)
- [BaseExtensionContext.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/context/BaseExtensionContext.java)
- [GlobalExtensionContext.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/context/GlobalExtensionContext.java)
- [DatabaseExtensionContext.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/context/DatabaseExtensionContext.java)
- [UserDataCollectorExtensionFactory.java](file://community/udc/src/main/java/org/neo4j/udc/UserDataCollectorExtensionFactory.java)
- [DataCollectorExtensionFactory.java](file://community/data-collector/src/main/java/org/neo4j/internal/collector/extension/DataCollectorExtensionFactory.java)
- [StorageSystemProvider.java](file://community/cloud/src/main/java/org/neo4j/cloud/storage/StorageSystemProvider.java)
- [StorageSystemProviderFactory.java](file://community/cloud/src/main/java/org/neo4j/cloud/storage/StorageSystemProviderFactory.java)
- [VectorEncoding.java](file://community/genai-plugin/src/main/java/org/neo4j/genai/vector/VectorEncoding.java)
- [Services.java](file://community/common/src/main/java/org/neo4j/service/Services.java)
- [ProcedureClassLoader.java](file://community/procedure/src/main/java/org/neo4j/procedure/impl/ProcedureClassLoader.java)
- [LifecycleAdapter.java](file://community/common/src/main/java/org/neo4j/kernel/lifecycle/LifecycleAdapter.java)
- [DatabaseEventListeners.java](file://community/kernel/src/main/java/org/neo4j/kernel/monitoring/DatabaseEventListeners.java)
- [DatabaseEventListener.java](file://community/graphdb-api/src/main/java/org/neo4j/graphdb/event/DatabaseEventListener.java)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Core Components](#core-components)
4. [Extension Factory System](#extension-factory-system)
5. [Storage System Providers](#storage-system-providers)
6. [Vector Encoding Implementation](#vector-encoding-implementation)
7. [Plugin Discovery and Configuration](#plugin-discovery-and-configuration)
8. [Lifecycle Management](#lifecycle-management)
9. [Common Issues and Solutions](#common-issues-and-solutions)
10. [Best Practices](#best-practices)
11. [Troubleshooting Guide](#troubleshooting-guide)

## Introduction

Neo4j's plugin system provides a powerful and extensible framework for adding custom functionality to the database engine. The system is built around the concept of extension factories that create and manage lifecycle-aware components, enabling developers to integrate custom logic, storage providers, and specialized functionality seamlessly with the core database operations.

The plugin system supports two primary extension types: **GLOBAL** extensions that operate at the database management service level, and **DATABASE** extensions that are scoped to individual databases. This dual-layer architecture allows for both system-wide and database-specific functionality while maintaining clear separation of concerns.

## Architecture Overview

The Neo4j plugin system follows a layered architecture that separates concerns between extension discovery, instantiation, lifecycle management, and integration with the core database components.

```mermaid
graph TB
subgraph "Plugin System Architecture"
A[Extension Factories] --> B[Extension Registry]
B --> C[Extension Manager]
C --> D[Extension Instances]
E[Service Discovery] --> A
F[Class Loading] --> E
G[Global Extensions] --> H[Global Extension Context]
I[Database Extensions] --> J[Database Extension Context]
K[Dependency Injection] --> D
L[Lifecycle Management] --> D
M[Database Events] --> N[Event Listeners]
O[System Events] --> P[System Listeners]
end
subgraph "Integration Points"
D --> Q[Kernel Extensions]
D --> R[Storage Systems]
D --> S[Procedures & Functions]
end
```

**Diagram sources**
- [ExtensionFactory.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/ExtensionFactory.java#L29-L83)
- [AbstractExtensions.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/AbstractExtensions.java#L36-L127)
- [Services.java](file://community/common/src/main/java/org/neo4j/service/Services.java#L36-L133)

## Core Components

### Extension Factory Base Classes

The foundation of the plugin system consists of several key abstract classes that define the contract for creating and managing extensions.

#### ExtensionFactory Abstract Class

The [`ExtensionFactory`](file://community/kernel/src/main/java/org/neo4j/kernel/extension/ExtensionFactory.java#L29-L83) serves as the base class for all extension factories, providing the essential infrastructure for creating extension instances.

```mermaid
classDiagram
class ExtensionFactory {
-ExtensionType extensionType
-String name
+ExtensionFactory(String name)
+ExtensionFactory(ExtensionType type, String name)
+Lifecycle newInstance(ExtensionContext context, DEPENDENCIES dependencies)*
+String getName()
+ExtensionType getExtensionType()
}
class UserDataCollectorExtensionFactory {
+UserDataCollectorExtensionFactory()
+Lifecycle newInstance(ExtensionContext context, Dependencies dependencies)
+interface Dependencies
}
class DataCollectorExtensionFactory {
+DataCollectorExtensionFactory()
+Lifecycle newInstance(ExtensionContext context, Dependencies dependencies)
+interface Dependencies
}
ExtensionFactory <|-- UserDataCollectorExtensionFactory
ExtensionFactory <|-- DataCollectorExtensionFactory
```

**Diagram sources**
- [ExtensionFactory.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/ExtensionFactory.java#L29-L83)
- [UserDataCollectorExtensionFactory.java](file://community/udc/src/main/java/org/neo4j/udc/UserDataCollectorExtensionFactory.java#L33-L63)
- [DataCollectorExtensionFactory.java](file://community/data-collector/src/main/java/org/neo4j/internal/collector/extension/DataCollectorExtensionFactory.java#L35-L62)

#### Extension Types

The system defines two distinct extension types that determine the scope and lifecycle of the extensions:

- **GLOBAL Extensions**: Operate at the database management service level, providing system-wide functionality
- **DATABASE Extensions**: Scoped to individual databases, allowing database-specific behavior

**Section sources**
- [ExtensionType.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/ExtensionType.java#L25-L34)
- [ExtensionFactory.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/ExtensionFactory.java#L34-L40)

### Extension Context System

The extension context provides the environment and dependencies required for extension initialization and operation.

```mermaid
classDiagram
class ExtensionContext {
<<interface>>
+DbmsInfo dbmsInfo()
+DependencySatisfier dependencySatisfier()
+Path directory()
}
class BaseExtensionContext {
-Path contextDirectory
-DbmsInfo dbmsInfo
-DependencySatisfier satisfier
+BaseExtensionContext(Path directory, DbmsInfo info, DependencySatisfier satisfier)
+DbmsInfo dbmsInfo()
+DependencySatisfier dependencySatisfier()
+Path directory()
}
class GlobalExtensionContext {
+GlobalExtensionContext(Neo4jLayout storeLayout, DbmsInfo dbmsInfo, DependencySatisfier satisfier)
}
class DatabaseExtensionContext {
+DatabaseExtensionContext(DatabaseLayout databaseLayout, DbmsInfo info, DependencySatisfier satisfier)
}
ExtensionContext <|.. BaseExtensionContext
BaseExtensionContext <|-- GlobalExtensionContext
BaseExtensionContext <|-- DatabaseExtensionContext
```

**Diagram sources**
- [ExtensionContext.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/context/ExtensionContext.java#L29-L41)
- [BaseExtensionContext.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/context/BaseExtensionContext.java#L26-L52)
- [GlobalExtensionContext.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/context/GlobalExtensionContext.java#L26-L30)
- [DatabaseExtensionContext.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/context/DatabaseExtensionContext.java#L26-L31)

**Section sources**
- [ExtensionContext.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/context/ExtensionContext.java#L29-L41)
- [BaseExtensionContext.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/context/BaseExtensionContext.java#L26-L52)

## Extension Factory System

### Implementation Patterns

The extension factory system follows consistent patterns for creating and managing extensions across different subsystems.

#### Basic Extension Factory Implementation

Here's how a typical extension factory is implemented:

```mermaid
sequenceDiagram
participant Client as "Extension Manager"
participant Factory as "ExtensionFactory"
participant Context as "ExtensionContext"
participant Dependencies as "Dependency Resolver"
participant Extension as "Extension Instance"
Client->>Factory : newInstance(context, dependencies)
Factory->>Dependencies : resolve dependencies
Dependencies-->>Factory : dependency objects
Factory->>Extension : new Extension(dependencies)
Extension-->>Factory : extension instance
Factory-->>Client : lifecycle object
Note over Client,Extension : Extension is now managed by lifecycle system
```

**Diagram sources**
- [AbstractExtensions.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/AbstractExtensions.java#L58-L70)
- [ExtensionFactory.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/ExtensionFactory.java#L55-L56)

#### Dependency Resolution

The system uses sophisticated dependency resolution to inject required services and components into extension instances.

**Section sources**
- [AbstractExtensions.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/AbstractExtensions.java#L109-L126)
- [ExtensionFactory.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/ExtensionFactory.java#L55-L56)

### Extension Lifecycle Management

Extensions follow a standardized lifecycle that ensures proper initialization, operation, and cleanup.

```mermaid
stateDiagram-v2
[*] --> Created : Factory.newInstance()
Created --> Initialized : init()
Initialized --> Started : start()
Started --> Stopped : stop()
Stopped --> Started : start()
Started --> Shutdown : shutdown()
Stopped --> Shutdown : shutdown()
Shutdown --> [*]
note right of Initialized : Dependencies resolved
note right of Started : Extension active
note right of Shutdown : Resources released
```

**Diagram sources**
- [LifecycleAdapter.java](file://community/common/src/main/java/org/neo4j/kernel/lifecycle/LifecycleAdapter.java#L41-L89)

**Section sources**
- [AbstractExtensions.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/AbstractExtensions.java#L58-L88)
- [LifecycleAdapter.java](file://community/common/src/main/java/org/neo4j/kernel/lifecycle/LifecycleAdapter.java#L41-L89)

## Storage System Providers

### Cloud Storage Integration

Neo4j's storage system provider framework enables seamless integration with cloud storage solutions, allowing databases to be stored remotely while maintaining local filesystem semantics.

#### StorageSystemProvider Architecture

```mermaid
classDiagram
class StorageSystemProvider {
-MutableMap~URI,StorageSystem~ systems
#String scheme
#ChunkChannelSupplier tempSupplier
#Config config
#InternalLogProvider logProvider
#MemoryTracker memoryTracker
+StorageSystem getStorageSystem(URI uri)
+SeekableByteChannel newByteChannel(StoragePath path, Set~OpenOption~ options)
+InputStream newInputStream(Path path, OpenOption... options)
+OutputStream newOutputStream(Path path, OpenOption... options)
#abstract SeekableByteChannel openAsByteChannel(StoragePath path, Set~OpenOption~ options)
#abstract OutputStream openAsOutputStream(StoragePath fileName, Set~OpenOption~ options)
#abstract InputStream openAsInputStream(StoragePath fileName)
#abstract StorageSystem create(URI storageUri)
#abstract StorageLocation resolve(URI uri)
}
class StorageSystemProviderFactory {
<<abstract>>
#String scheme
+String scheme()
+boolean matches(String scheme)
+StorageSystemProvider createStorageSystemProvider(...)
#abstract String storageSystemProviderClass()
}
class SchemeFileSystemAbstraction {
-FileSystemAbstraction fs
-Collection~StorageSystemProviderFactory~ factories
-Config config
-InternalLogProvider logProvider
-MemoryTracker memoryTracker
+SchemeFileSystemAbstraction(FileSystemAbstraction fs)
+StorageSystem getStorageSystem(URI uri)
}
StorageSystemProvider <|-- CloudStorageProvider
StorageSystemProviderFactory <|-- CloudStorageFactory
SchemeFileSystemAbstraction --> StorageSystemProviderFactory
StorageSystemProviderFactory --> StorageSystemProvider
```

**Diagram sources**
- [StorageSystemProvider.java](file://community/cloud/src/main/java/org/neo4j/cloud/storage/StorageSystemProvider.java#L60-L298)
- [StorageSystemProviderFactory.java](file://community/cloud/src/main/java/org/neo4j/cloud/storage/StorageSystemProviderFactory.java#L42-L131)

#### Implementation Example: Cloud Storage Provider

The cloud storage provider demonstrates how to implement a storage system that integrates with remote storage services while maintaining local filesystem compatibility.

**Section sources**
- [StorageSystemProvider.java](file://community/cloud/src/main/java/org/neo4j/cloud/storage/StorageSystemProvider.java#L60-L298)
- [StorageSystemProviderFactory.java](file://community/cloud/src/main/java/org/neo4j/cloud/storage/StorageSystemProviderFactory.java#L42-L131)

## Vector Encoding Implementation

### GenAI Plugin Architecture

The vector encoding system showcases advanced plugin capabilities, providing AI-powered vector embeddings for graph data processing.

#### Vector Encoding System

```mermaid
classDiagram
class VectorEncoding {
-ImmutableList~Provider~ PROVIDERS
-Map~String,VectorEncodingCallCountersMonitor~ MONITORS
+GraphDatabaseService graphDatabaseService
+Value encode(String resource, String providerName, AnyValue configuration)
+Stream~InternalBatchRow~ encodeBatch(String[] resources, String providerName, AnyValue configuration)
+Stream~ProviderRow~ listEncodingProviders()
+static Provider getProvider(String name)
+interface Provider
}
class Provider {
<<interface>>
+Class parameterDeclarations()
+String name()
+Encoder configure(MapValue configuration)
+Encoder configure(PARAMETERS configuration)
}
class Encoder {
<<interface>>
+float[] encode(String resource)
+Stream~BatchRow~ encode(String[] resources, int[] nullIndexes)
}
class VectorEncodingCallCountersMonitor {
<<interface>>
+void encodeFunctionCalled(String provider)
+void encodeBatchProcedureCalled(String provider)
}
VectorEncoding --> Provider
Provider --> Encoder
VectorEncoding --> VectorEncodingCallCountersMonitor
```

**Diagram sources**
- [VectorEncoding.java](file://community/genai-plugin/src/main/java/org/neo4j/genai/vector/VectorEncoding.java#L62-L343)

#### Provider Implementation Pattern

The vector encoding system uses a service provider pattern to support multiple AI providers:

```mermaid
sequenceDiagram
participant Client as "Application"
participant VectorEncoding as "VectorEncoding"
participant Provider as "AI Provider"
participant Encoder as "Encoder"
Client->>VectorEncoding : encode(resource, provider, config)
VectorEncoding->>VectorEncoding : getProvider(providerName)
VectorEncoding->>Provider : configure(configuration)
Provider->>Encoder : new Encoder()
Encoder-->>Provider : encoder instance
Provider-->>VectorEncoding : configured encoder
VectorEncoding->>Encoder : encode(resource)
Encoder-->>VectorEncoding : float[] vector
VectorEncoding-->>Client : vector value
```

**Diagram sources**
- [VectorEncoding.java](file://community/genai-plugin/src/main/java/org/neo4j/genai/vector/VectorEncoding.java#L144-L176)

**Section sources**
- [VectorEncoding.java](file://community/genai-plugin/src/main/java/org/neo4j/genai/vector/VectorEncoding.java#L62-L343)

## Plugin Discovery and Configuration

### Service Discovery Mechanism

Neo4j uses Java's ServiceLoader mechanism combined with annotation processing to discover and register plugins automatically.

#### Plugin Discovery Process

```mermaid
flowchart TD
A[Classpath Scanning] --> B[ServiceLoader Discovery]
B --> C[META-INF/services Generation]
C --> D[Plugin Registration]
D --> E[Dependency Resolution]
E --> F[Extension Instantiation]
G[Annotation Processing] --> H[Service Configuration]
H --> C
I[Plugin Validation] --> J[Compatibility Check]
J --> K[Error Reporting]
F --> L[Extension Lifecycle]
L --> M[Runtime Management]
```

**Diagram sources**
- [Services.java](file://community/common/src/main/java/org/neo4j/service/Services.java#L36-L133)
- [ProcedureClassLoader.java](file://community/procedure/src/main/java/org/neo4j/procedure/impl/ProcedureClassLoader.java#L176-L250)

#### Configuration Options

The plugin system supports various configuration mechanisms:

| Configuration Type | Scope | Purpose | Example |
|-------------------|-------|---------|---------|
| META-INF/services | Global | Automatic discovery | ExtensionFactory implementations |
| Classpath scanning | Runtime | Dynamic loading | Plugin JAR files |
| Dependency injection | Per-extension | Service resolution | Database services, logging |
| Lifecycle management | Per-instance | State coordination | Initialization, shutdown |

**Section sources**
- [Services.java](file://community/common/src/main/java/org/neo4j/service/Services.java#L36-L133)
- [ProcedureClassLoader.java](file://community/procedure/src/main/java/org/neo4j/procedure/impl/ProcedureClassLoader.java#L176-L250)

### Classpath Management

The system handles complex classpath scenarios to prevent conflicts and ensure proper isolation.

#### ClassLoader Hierarchy

```mermaid
graph TD
A[Bootstrap ClassLoader] --> B[Extension ClassLoader]
B --> C[Plugin JAR 1]
B --> D[Plugin JAR 2]
B --> E[Plugin JAR N]
F[Application ClassLoader] --> G[Neo4j Core]
G --> B
H[System ClassLoader] --> I[Java Runtime]
I --> A
```

**Diagram sources**
- [ProcedureClassLoader.java](file://community/procedure/src/main/java/org/neo4j/procedure/impl/ProcedureClassLoader.java#L176-L250)

**Section sources**
- [ProcedureClassLoader.java](file://community/procedure/src/main/java/org/neo4j/procedure/impl/ProcedureClassLoader.java#L176-L250)

## Lifecycle Management

### Database Event Integration

The plugin system integrates deeply with Neo4j's database lifecycle events, enabling extensions to respond to database state changes.

#### Database Event Lifecycle

```mermaid
sequenceDiagram
participant DBMS as "Database Management Service"
participant EventMgr as "Event Manager"
participant Ext1 as "Extension 1"
participant Ext2 as "Extension 2"
DBMS->>EventMgr : databaseCreate(namedDbId)
EventMgr->>Ext1 : databaseCreate(event)
EventMgr->>Ext2 : databaseCreate(event)
DBMS->>EventMgr : databaseStart(namedDbId)
EventMgr->>Ext1 : databaseStart(event)
EventMgr->>Ext2 : databaseStart(event)
DBMS->>EventMgr : databaseShutdown(namedDbId)
EventMgr->>Ext1 : databaseShutdown(event)
EventMgr->>Ext2 : databaseShutdown(event)
DBMS->>EventMgr : databaseDrop(namedDbId)
EventMgr->>Ext1 : databaseDrop(event)
EventMgr->>Ext2 : databaseDrop(event)
```

**Diagram sources**
- [DatabaseEventListeners.java](file://community/kernel/src/main/java/org/neo4j/kernel/monitoring/DatabaseEventListeners.java#L34-L107)
- [DatabaseEventListener.java](file://community/graphdb-api/src/main/java/org/neo4j/graphdb/event/DatabaseEventListener.java#L32-L61)

#### Event Listener Implementation

Extensions can register themselves as database event listeners to participate in the database lifecycle:

**Section sources**
- [DatabaseEventListeners.java](file://community/kernel/src/main/java/org/neo4j/kernel/monitoring/DatabaseEventListeners.java#L34-L107)
- [DatabaseEventListener.java](file://community/graphdb-api/src/main/java/org/neo4j/graphdb/event/DatabaseEventListener.java#L32-L61)

### Resource Management

The system provides comprehensive resource management capabilities to ensure proper cleanup and prevent resource leaks.

#### Resource Lifecycle

```mermaid
stateDiagram-v2
[*] --> Allocated : new Resource()
Allocated --> Active : useResource()
Active --> Released : release()
Released --> Closed : close()
Closed --> [*]
Active --> Error : exception
Error --> Released : cleanup()
```

**Diagram sources**
- [AbstractExtensions.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/AbstractExtensions.java#L58-L88)

**Section sources**
- [AbstractExtensions.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/AbstractExtensions.java#L58-L88)

## Common Issues and Solutions

### Classpath Conflicts

**Problem**: Multiple versions of the same library in the classpath causing linkage errors.

**Solution**: Use isolated classloaders for each plugin and implement proper dependency versioning.

**Example**: The [`ProcedureClassLoader`](file://community/procedure/src/main/java/org/neo4j/procedure/impl/ProcedureClassLoader.java#L176-L250) handles classpath conflicts by isolating plugin classes.

### Version Compatibility

**Problem**: Plugins compiled against different Neo4j versions causing compatibility issues.

**Solution**: Implement version checking and provide compatibility layers.

**Example**: The [`Services`](file://community/common/src/main/java/org/neo4j/service/Services.java#L36-L133) utility provides version-aware service loading.

### Proper Shutdown Procedures

**Problem**: Extensions not properly releasing resources during shutdown.

**Solution**: Implement proper lifecycle management with cleanup hooks.

**Example**: The [`VectorEncoding.MonitorsCleaner`](file://community/genai-plugin/src/main/java/org/neo4j/genai/vector/VectorEncoding.java#L306-L342) extension cleans up monitors on database shutdown.

**Section sources**
- [ProcedureClassLoader.java](file://community/procedure/src/main/java/org/neo4j/procedure/impl/ProcedureClassLoader.java#L176-L250)
- [Services.java](file://community/common/src/main/java/org/neo4j/service/Services.java#L36-L133)
- [VectorEncoding.java](file://community/genai-plugin/src/main/java/org/neo4j/genai/vector/VectorEncoding.java#L306-L342)

## Best Practices

### Thread Safety in Extensions

Extensions should be designed with thread safety in mind, especially when sharing state across multiple database operations.

**Guidelines**:
- Use immutable objects where possible
- Implement proper synchronization for shared state
- Avoid static mutable state
- Use thread-local storage for request-specific data

### Resource Management

Proper resource management is crucial for preventing memory leaks and ensuring system stability.

**Best Practices**:
- Implement proper cleanup in shutdown methods
- Use try-with-resources for automatic resource management
- Register cleanup callbacks for asynchronous operations
- Monitor resource usage and implement limits

### Configuration Management

Effective configuration management ensures plugins work correctly across different environments.

**Recommendations**:
- Use type-safe configuration objects
- Provide sensible defaults
- Validate configuration at startup
- Support configuration hot-reloading where appropriate

## Troubleshooting Guide

### Plugin Loading Issues

**Symptoms**: Plugin classes not found or service providers not discovered.

**Diagnosis Steps**:
1. Check META-INF/services files are properly generated
2. Verify plugin JAR files are in the correct directory
3. Review classpath configuration
4. Check for compilation errors in plugin code

**Resolution**:
- Ensure proper annotation processing
- Verify service provider registration
- Check classpath ordering

### Lifecycle Management Problems

**Symptoms**: Extensions not initializing or shutting down correctly.

**Diagnosis**:
1. Verify extension implements Lifecycle interface
2. Check dependency resolution
3. Review extension factory implementation
4. Examine lifecycle event handling

**Common Solutions**:
- Implement proper dependency injection
- Add error handling to lifecycle methods
- Ensure proper resource cleanup

### Performance Issues

**Symptoms**: Slow plugin initialization or degraded database performance.

**Investigation**:
1. Profile plugin initialization time
2. Monitor resource usage
3. Check for blocking operations
4. Review thread usage patterns

**Optimization Strategies**:
- Implement lazy initialization
- Use connection pooling
- Optimize resource allocation
- Reduce synchronous operations

**Section sources**
- [ProcedureClassLoader.java](file://community/procedure/src/main/java/org/neo4j/procedure/impl/ProcedureClassLoader.java#L176-L250)
- [AbstractExtensions.java](file://community/kernel/src/main/java/org/neo4j/kernel/extension/AbstractExtensions.java#L58-L88)