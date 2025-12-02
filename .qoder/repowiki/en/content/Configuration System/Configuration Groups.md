# Configuration Groups

<cite>
**Referenced Files in This Document**   
- [GroupSetting.java](file://community/configuration/src/main/java/org/neo4j/configuration/GroupSetting.java)
- [GroupSettingHelper.java](file://community/configuration/src/main/java/org/neo4j/configuration/GroupSettingHelper.java)
- [BoltConnector.java](file://community/configuration/src/main/java/org/neo4j/configuration/connectors/BoltConnector.java)
- [HttpConnector.java](file://community/configuration/src/main/java/org/neo4j/configuration/connectors/HttpConnector.java)
- [SslPolicyConfig.java](file://community/configuration/src/main/java/org/neo4j/configuration/ssl/SslPolicyConfig.java)
- [CrsConfig.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/index/schema/config/CrsConfig.java)
- [Config.java](file://community/configuration/src/main/java/org/neo4j/configuration/Config.java)
- [LocalConfig.java](file://community/configuration/src/main/java/org/neo4j/configuration/LocalConfig.java)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Domain Model](#domain-model)
3. [Core Components](#core-components)
4. [Implementation Examples](#implementation-examples)
5. [Configuration Loading and UI Presentation](#configuration-loading-and-ui-presentation)
6. [Common Issues and Solutions](#common-issues-and-solutions)
7. [Creating Custom Configuration Groups](#creating-custom-configuration-groups)
8. [Conclusion](#conclusion)

## Introduction
Configuration groups in Neo4j provide a structured approach to organizing related settings into logical units such as database, connectors, or storage. This organizational pattern enhances configuration management by grouping related parameters under a common prefix and name, enabling better maintainability and clarity. The implementation leverages the `GroupSetting` interface as a container for related settings and utilizes `GroupSettingHelper` for utility operations. This document explores the domain model, implementation examples, and practical considerations for working with configuration groups in Neo4j.

## Domain Model
The domain model for configuration groups in Neo4j centers around the `GroupSetting` interface, which serves as the foundation for organizing related settings. Each group is identified by a unique name and shares a common prefix, allowing for hierarchical organization of configuration parameters. The `GroupSettingHelper` class provides utility methods for creating settings within a group, simplifying the process of defining configuration options. This model supports both simple and complex hierarchical settings, enabling developers to create flexible and scalable configuration structures.

```mermaid
classDiagram
class GroupSetting {
<<interface>>
+name() String
+getPrefix() String
}
class GroupSettingHelper {
+getBuilder(prefix, name, suffix, parser, defaultValue) SettingBuilder
+getBuilder(prefix, name, parser, defaultValue) SettingBuilder
}
class SslPolicyConfig {
+enabled Setting<Boolean>
+base_directory Setting<Path>
+revoked_dir Setting<Path>
+trust_all Setting<Boolean>
+trust_expired Setting<Boolean>
+client_auth Setting<ClientAuth>
+tls_versions Setting<List<String>>
+ciphers Setting<List<String>>
+verify_hostname Setting<Boolean>
+private_key Setting<Path>
+private_key_password Setting<SecureString>
+public_certificate Setting<Path>
+trusted_dir Setting<Path>
+name() String
+getPrefix() String
}
class CrsConfig {
+min Setting<List<Double>>
+max Setting<List<Double>>
+crs CoordinateReferenceSystem
+name() String
+getPrefix() String
}
GroupSetting <|-- SslPolicyConfig
GroupSetting <|-- CrsConfig
GroupSettingHelper ..> SettingImpl
```

**Diagram sources**
- [GroupSetting.java](file://community/configuration/src/main/java/org/neo4j/configuration/GroupSetting.java)
- [GroupSettingHelper.java](file://community/configuration/src/main/java/org/neo4j/configuration/GroupSettingHelper.java)
- [SslPolicyConfig.java](file://community/configuration/src/main/java/org/neo4j/configuration/ssl/SslPolicyConfig.java)
- [CrsConfig.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/index/schema/config/CrsConfig.java)

**Section sources**
- [GroupSetting.java](file://community/configuration/src/main/java/org/neo4j/configuration/GroupSetting.java)
- [GroupSettingHelper.java](file://community/configuration/src/main/java/org/neo4j/configuration/GroupSettingHelper.java)

## Core Components
The core components of the configuration groups system in Neo4j include the `GroupSetting` interface, which defines the contract for configuration groups, and the `GroupSettingHelper` class, which provides utility methods for creating settings within a group. The `Config` class manages the lifecycle of configuration instances and provides methods for retrieving groups of settings. The `LocalConfig` class extends the functionality of `Config` by adding support for local configuration overrides and listeners.

```mermaid
classDiagram
class Config {
+getGroups(group) Map<String, T>
+getGroupsFromInheritance(parentClass) Map<Class<U>, Map<String, U>>
+get(setting) T
+getDefault(setting) T
+getStartupValue(setting) T
+getValueSource(setting) ValueSource
+getObserver(setting) SettingObserver
+setDynamic(setting, value, scope) void
}
class LocalConfig {
+getGroups(group) Map<String, T>
+getGroupsFromInheritance(parentClass) Map<Class<U>, Map<String, U>>
+get(setting) T
+getDefault(setting) T
+getStartupValue(setting) T
+getValueSource(setting) ValueSource
+getObserver(setting) SettingObserver
+setDynamic(setting, value, scope) void
}
Config <|-- LocalConfig
```

**Diagram sources**
- [Config.java](file://community/configuration/src/main/java/org/neo4j/configuration/Config.java)
- [LocalConfig.java](file://community/configuration/src/main/java/org/neo4j/configuration/LocalConfig.java)

**Section sources**
- [Config.java](file://community/configuration/src/main/java/org/neo4j/configuration/Config.java)
- [LocalConfig.java](file://community/configuration/src/main/java/org/neo4j/configuration/LocalConfig.java)

## Implementation Examples
The implementation of configuration groups in Neo4j is exemplified by the `BoltConnector` and `HttpConnector` classes, which group their respective settings under common prefixes. The `BoltConnector` class defines settings such as `enabled`, `listen_address`, and `advertised_address`, while the `HttpConnector` class defines similar settings for HTTP connectivity. These examples demonstrate how configuration groups can be used to organize related settings in a logical and maintainable manner.

```mermaid
classDiagram
class BoltConnector {
+DEFAULT_PORT int
+NAME String
+INTERNAL_NAME String
+enabled Setting<Boolean>
+server_bolt_telemetry_enabled Setting<Boolean>
+encryption_level Setting<EncryptionLevel>
+listen_address Setting<SocketAddress>
+additional_listen_addresses Setting<Set<SocketAddress>>
+advertised_address Setting<SocketAddress>
}
class HttpConnector {
+DEFAULT_PORT int
+NAME String
+enabled Setting<Boolean>
+listen_address Setting<SocketAddress>
+advertised_address Setting<SocketAddress>
}
class HttpsConnector {
+DEFAULT_PORT int
+NAME String
+enabled Setting<Boolean>
+listen_address Setting<SocketAddress>
+advertised_address Setting<SocketAddress>
}
BoltConnector ..> Setting
HttpConnector ..> Setting
HttpsConnector ..> Setting
```

**Diagram sources**
- [BoltConnector.java](file://community/configuration/src/main/java/org/neo4j/configuration/connectors/BoltConnector.java)
- [HttpConnector.java](file://community/configuration/src/main/java/org/neo4j/configuration/connectors/HttpConnector.java)
- [HttpsConnector.java](file://community/configuration/src/main/java/org/neo4j/configuration/connectors/HttpsConnector.java)

**Section sources**
- [BoltConnector.java](file://community/configuration/src/main/java/org/neo4j/configuration/connectors/BoltConnector.java)
- [HttpConnector.java](file://community/configuration/src/main/java/org/neo4j/configuration/connectors/HttpConnector.java)

## Configuration Loading and UI Presentation
Configuration groups in Neo4j are loaded through the `Config` class, which manages the lifecycle of configuration instances and provides methods for retrieving groups of settings. The `getGroups` method allows for the retrieval of all instances of a specific group, while the `getGroupsFromInheritance` method enables the retrieval of groups based on inheritance relationships. This functionality supports both programmatic access to configuration data and integration with user interfaces for configuration management.

```mermaid
sequenceDiagram
participant User as "User Interface"
participant Config as "Config"
participant GroupSetting as "GroupSetting"
User->>Config : getGroups(SslPolicyConfig.class)
Config->>Config : Retrieve all SslPolicyConfig instances
Config-->>User : Map<String, SslPolicyConfig>
User->>Config : getGroupsFromInheritance(GroupSetting.class)
Config->>Config : Retrieve all GroupSetting instances
Config-->>User : Map<Class<U>, Map<String, U>>
```

**Diagram sources**
- [Config.java](file://community/configuration/src/main/java/org/neo4j/configuration/Config.java)
- [SslPolicyConfig.java](file://community/configuration/src/main/java/org/neo4j/configuration/ssl/SslPolicyConfig.java)

**Section sources**
- [Config.java](file://community/configuration/src/main/java/org/neo4j/configuration/Config.java)

## Common Issues and Solutions
Common issues when working with configuration groups in Neo4j include naming conflicts and circular dependencies. Naming conflicts can occur when multiple groups define settings with the same name, leading to ambiguity in configuration resolution. Circular dependencies can arise when groups reference each other in a way that creates an infinite loop. These issues can be mitigated by using unique prefixes for each group and carefully managing dependencies between groups.

```mermaid
flowchart TD
Start([Start]) --> CheckNamingConflict["Check for naming conflicts"]
CheckNamingConflict --> NamingConflict{"Naming conflict?"}
NamingConflict --> |Yes| ResolveConflict["Resolve conflict using unique prefixes"]
NamingConflict --> |No| CheckCircularDependency["Check for circular dependencies"]
CheckCircularDependency --> CircularDependency{"Circular dependency?"}
CircularDependency --> |Yes| ResolveDependency["Resolve dependency using dependency injection"]
CircularDependency --> |No| End([End])
ResolveConflict --> End
ResolveDependency --> End
```

**Diagram sources**
- [Config.java](file://community/configuration/src/main/java/org/neo4j/configuration/Config.java)

**Section sources**
- [Config.java](file://community/configuration/src/main/java/org/neo4j/configuration/Config.java)

## Creating Custom Configuration Groups
Creating custom configuration groups in Neo4j involves implementing the `GroupSetting` interface and defining settings using the `GroupSettingHelper` class. Developers can create groups for specific domains such as database, connectors, or storage, and organize related settings under a common prefix. This approach promotes modularity and reusability, enabling the creation of flexible and scalable configuration structures.

```mermaid
classDiagram
class CustomGroup {
+setting1 Setting<T>
+setting2 Setting<U>
+name() String
+getPrefix() String
}
GroupSetting <|-- CustomGroup
CustomGroup ..> GroupSettingHelper
```

**Diagram sources**
- [GroupSetting.java](file://community/configuration/src/main/java/org/neo4j/configuration/GroupSetting.java)
- [GroupSettingHelper.java](file://community/configuration/src/main/java/org/neo4j/configuration/GroupSettingHelper.java)

**Section sources**
- [GroupSetting.java](file://community/configuration/src/main/java/org/neo4j/configuration/GroupSetting.java)
- [GroupSettingHelper.java](file://community/configuration/src/main/java/org/neo4j/configuration/GroupSettingHelper.java)

## Conclusion
Configuration groups in Neo4j provide a powerful mechanism for organizing related settings into logical units, enhancing configuration management and maintainability. By leveraging the `GroupSetting` interface and `GroupSettingHelper` class, developers can create flexible and scalable configuration structures that support both simple and complex hierarchical settings. This document has explored the domain model, implementation examples, and practical considerations for working with configuration groups, providing a comprehensive guide for both beginners and experienced developers.