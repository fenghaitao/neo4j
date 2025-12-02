# Building and Testing

<cite>
**Referenced Files in This Document**   
- [pom.xml](file://pom.xml)
- [neo4j-harness/pom.xml](file://community/neo4j-harness/pom.xml)
- [it-test-support/src/main/java/org/neo4j/test/extension/DbmsExtension.java](file://community/community-it/it-test-support/src/main/java/org/neo4j/test/extension/DbmsExtension.java)
- [it-test-support/src/main/java/org/neo4j/test/extension/DbmsSupportController.java](file://community/community-it/it-test-support/src/main/java/org/neo4j/test/extension/DbmsSupportController.java)
- [it-test-support/src/main/java/org/neo4j/test/extension/TestDirectoryExtension.java](file://community/testing/io-utils/src/main/java/org/neo4j/test/extension/testdirectory/TestDirectoryExtension.java)
- [testing/io-utils/src/main/java/org/neo4j/io/pagecache/stress/PageCacheStresser.java](file://community/testing/io-utils/src/main/java/org/neo4j/io/pagecache/stress/PageCacheStresser.java)
- [testing/io-utils/src/main/java/org/neo4j/io/pagecache/stress/PageCacheStressTest.java](file://community/testing/io-utils/src/main/java/org/neo4j/io/pagecache/stress/PageCacheStressTest.java)
- [kernel/src/main/java/org/neo4j/kernel/impl/util/DebugUtil.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/util/DebugUtil.java)
- [testing/random-values/src/main/java/org/neo4j/values/storable/RandomValues.java](file://community/testing/random-values/src/main/java/org/neo4j/values/storable/RandomValues.java)
- [CONTRIBUTING.md](file://CONTRIBUTING.md)
</cite>

## Table of Contents
1. [Maven Build System](#maven-build-system)
2. [Testing Infrastructure](#testing-infrastructure)
3. [Running Tests](#running-tests)
4. [Debugging Test Failures](#debugging-test-failures)
5. [Writing New Tests](#writing-new-tests)
6. [Performance Testing](#performance-testing)
7. [Common Build Issues](#common-build-issues)
8. [Continuous Integration](#continuous-integration)

## Maven Build System

The Neo4j codebase uses a comprehensive Maven-based build system with a multi-module structure. The root `pom.xml` file defines the overall project configuration, including Java version requirements, compiler settings, and dependency management. The build system is configured to use Java 17 as the target version, with specific JVM options for optimal performance and debugging capabilities.

The project follows a modular architecture with three main top-level modules: `annotations`, `community`, and `packaging`. The `community` module contains numerous submodules organized by functionality, such as `bolt`, `cypher`, `dbms`, and various testing utilities. This modular structure allows for independent development and testing of different components while maintaining a cohesive build process.

Dependency management is handled through Maven's standard mechanisms, with version properties defined in the parent POM for consistency across modules. The build system includes extensive plugin configurations for code quality, testing, and packaging. Key plugins include:
- `maven-surefire-plugin` for unit testing
- `maven-failsafe-plugin` for integration testing
- `maven-enforcer-plugin` for build consistency rules
- `maven-javadoc-plugin` for API documentation generation

The build system also includes specialized configurations for different build scenarios, such as parallel test execution and custom VM target versions. Profiles can be activated based on environment variables or file presence to customize the build process for different environments.

**Section sources**
- [pom.xml](file://pom.xml#L1-L800)
- [pom.xml](file://pom.xml#L717-L1017)

## Testing Infrastructure

Neo4j employs a sophisticated testing infrastructure built on JUnit 5, with custom extensions to support complex integration testing scenarios. The primary testing framework components are located in the `community/community-it/it-test-support` module, which provides reusable test extensions and utilities.

The core of the testing infrastructure is the `DbmsExtension` annotation, which integrates with JUnit 5's extension model to manage the lifecycle of Neo4j database instances during tests. This extension automatically handles database startup and shutdown, ensuring proper resource cleanup. It supports both permanent and ephemeral file system abstractions through complementary extensions like `ImpermanentDbmsExtension`.

Key components of the testing infrastructure include:
- `DbmsSupportController`: Manages the database management service lifecycle and provides methods for starting, stopping, and restarting databases
- `TestDirectoryExtension`: Manages temporary test directories and file system resources
- `Neo4jWithSocketSupportExtension`: Provides network connectivity for testing distributed scenarios

The infrastructure supports dependency injection into test classes, automatically injecting components like `DatabaseManagementService`, `GraphDatabaseService`, and `FileSystemAbstraction` when fields are annotated appropriately. This reduces boilerplate code and ensures consistent test setup.

**Section sources**
- [it-test-support/src/main/java/org/neo4j/test/extension/DbmsExtension.java](file://community/community-it/it-test-support/src/main/java/org/neo4j/test/extension/DbmsExtension.java#L33-L83)
- [it-test-support/src/main/java/org/neo4j/test/extension/DbmsSupportController.java](file://community/community-it/it-test-support/src/main/java/org/neo4j/test/extension/DbmsSupportController.java#L30-L336)
- [it-test-support/src/main/java/org/neo4j/test/extension/TestDirectoryExtension.java](file://community/testing/io-utils/src/main/java/org/neo4j/test/extension/testdirectory/TestDirectoryExtension.java)

## Running Tests

The Neo4j build system distinguishes between unit tests and integration tests using Maven's standard testing plugins. Unit tests are executed by the `maven-surefire-plugin` and are typically named with patterns like `*Test.java`. Integration tests are handled by the `maven-failsafe-plugin` and follow naming conventions such as `IT*.java`, `*IT.java`, or `*IntegrationTest.java`.

To run tests, use the standard Maven commands:
- `mvn test` to execute unit tests
- `mvn verify` to run both unit and integration tests
- `mvn install` to build and test the complete project

The build system supports parallel test execution through the `parallel.tests` property, which can be enabled to improve test performance on multi-core systems. Test execution can be customized using various system properties, such as `test.vm.heap.size` to control heap allocation for test JVMs.

For specific testing scenarios, the build includes profiles that can be activated to skip certain tests or modify test behavior. For example, some integration test modules include profiles to skip tests under certain conditions, as seen in various `pom.xml` files across the codebase.

**Section sources**
- [pom.xml](file://pom.xml#L325-L404)
- [pom.xml](file://pom.xml#L350-L354)
- [pom.xml](file://pom.xml#L377-L381)

## Debugging Test Failures

When debugging test failures in Neo4j, several tools and techniques are available. The build system configures JVM options to facilitate debugging, including heap dump generation on out-of-memory errors and detailed garbage collection logging. These settings are defined in the `test.runner.jvm.settings` property in the parent POM.

The codebase includes specialized utilities for debugging and performance analysis:
- `DebugUtil.Timer` provides a simple mechanism for measuring execution time at various points in code
- `FakeCpuClock` allows for deterministic testing of time-dependent functionality
- Various tracer classes enable detailed monitoring of system behavior during tests

When a test fails, the first step is to examine the generated test reports in the `target/surefire-reports` directory. These reports include detailed stack traces and execution information. For integration tests, additional diagnostic information may be available in log files generated by the embedded Neo4j instance.

The test infrastructure also supports conditional execution through the `Condition` interface, allowing tests to be skipped based on runtime environment factors. This helps prevent false failures due to missing dependencies or incompatible configurations.

**Section sources**
- [kernel/src/main/java/org/neo4j/kernel/impl/util/DebugUtil.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/util/DebugUtil.java#L264-L322)
- [kernel-test-utils/src/main/java/org/neo4j/test/FakeCpuClock.java](file://community/kernel-test-utils/src/main/java/org/neo4j/test/FakeCpuClock.java)
- [pom.xml](file://pom.xml#L93-L131)

## Writing New Tests

To write new tests for Neo4j, leverage the existing test infrastructure and follow established patterns. The primary approach for integration testing is to use the `@DbmsExtension` annotation on test classes or methods. This annotation automatically sets up a Neo4j database instance and injects required dependencies.

When creating new tests, consider the following guidelines:
1. Use descriptive test method names that clearly indicate the scenario being tested
2. Follow the arrange-act-assert pattern for test structure
3. Use appropriate test categories (unit, integration, performance) based on the scope
4. Clean up resources in `@AfterEach` or `@AfterAll` methods when necessary

For complex test scenarios, you can customize database configuration using the `configurationCallback` parameter of `@DbmsExtension`. This allows you to modify the `TestDatabaseManagementServiceBuilder` before database creation, enabling customization of settings, dependencies, and extensions.

The codebase also provides various test utility classes, such as `RandomValues` for generating test data and `PageCacheStresser` for performance testing, which can be reused in new tests to reduce duplication.

**Section sources**
- [it-test-support/src/main/java/org/neo4j/test/extension/DbmsExtension.java](file://community/community-it/it-test-support/src/main/java/org/neo4j/test/extension/DbmsExtension.java)
- [testing/random-values/src/main/java/org/neo4j/values/storable/RandomValues.java](file://community/testing/random-values/src/main/java/org/neo4j/values/storable/RandomValues.java)
- [CONTRIBUTING.md](file://CONTRIBUTING.md#L46-L48)

## Performance Testing

Neo4j includes specialized infrastructure for performance testing, particularly for low-level components like the page cache. The `PageCacheStresser` class provides a framework for testing the page cache under various workloads and conditions. This tool creates multiple threads that perform concurrent operations on mapped files, simulating real-world usage patterns.

Performance tests are typically implemented as JUnit tests that use the stress testing framework to measure throughput, latency, and resource utilization. The `PageCacheStressTest.Builder` class provides a fluent API for configuring stress test parameters such as the number of pages, threads, and cache size.

The build system includes JVM options specifically tuned for performance testing, including G1 garbage collection and heap dump generation on memory errors. These settings help identify performance bottlenecks and memory issues during testing.

For microbenchmarking, developers can use the `DebugUtil.Timer` class to measure execution time of specific code sections. This simple but effective tool allows for quick performance comparisons between different implementations.

**Section sources**
- [testing/io-utils/src/main/java/org/neo4j/io/pagecache/stress/PageCacheStresser.java](file://community/testing/io-utils/src/main/java/org/neo4j/io/pagecache/stress/PageCacheStresser.java#L57-L110)
- [testing/io-utils/src/main/java/org/neo4j/io/pagecache/stress/PageCacheStressTest.java](file://community/testing/io-utils/src/main/java/org/neo4j/io/pagecache/stress/PageCacheStressTest.java#L91-L130)
- [kernel/src/main/java/org/neo4j/kernel/impl/util/DebugUtil.java](file://community/kernel/src/main/java/org/neo4j/kernel/impl/util/DebugUtil.java)

## Common Build Issues

Several common build issues may arise when working with the Neo4j codebase:

1. **Dependency conflicts**: The build system includes strict enforcer rules to prevent dependency issues, particularly banning test-scoped dependencies in compile scope. If you encounter dependency conflicts, check the `maven-enforcer-plugin` configuration in the parent POM.

2. **Java version mismatches**: The build requires Java 17. Ensure your JAVA_HOME environment variable points to a compatible JDK installation.

3. **Test failures due to resource constraints**: The default test heap size is 2GB. On systems with limited memory, you may need to reduce this by setting the `test.vm.heap.size` property.

4. **Platform-specific issues**: Some tests may fail on certain operating systems due to file system or networking differences. The test infrastructure includes conditional execution to handle some of these cases.

5. **Build profile activation**: Certain build profiles are activated based on environment variables or file presence. If expected functionality is missing, check if the appropriate profile needs to be activated.

The CONTRIBUTING.md file provides additional guidance on the development workflow, including requirements for pull requests and code contributions.

**Section sources**
- [pom.xml](file://pom.xml#L77-L131)
- [pom.xml](file://pom.xml#L717-L721)
- [CONTRIBUTING.md](file://CONTRIBUTING.md)

## Continuous Integration

The Neo4j build system is designed to work seamlessly with continuous integration environments. The Maven configuration includes several features that support CI/CD workflows:

1. **Deterministic builds**: The build system produces consistent outputs across different environments, which is essential for reliable CI pipelines.

2. **Parallel test execution**: Support for concurrent test execution helps reduce build times in CI environments with multiple cores.

3. **Comprehensive reporting**: The build generates detailed test reports in standard formats that can be consumed by CI tools.

4. **License compliance**: The licensing plugins automatically generate NOTICE and LICENSES files, ensuring compliance with open source requirements.

5. **Code quality enforcement**: The enforcer plugin validates coding standards and project structure, preventing common mistakes.

The build can be customized for CI environments using system properties and profiles. For example, the `sequentialTests` property can be used to disable parallel execution if needed, and the `VM_TARGET_VERSION` environment variable can override the default Java version.

**Section sources**
- [pom.xml](file://pom.xml#L1013-L1017)
- [pom.xml](file://pom.xml#L975-L999)
- [pom.xml](file://pom.xml#L1002-L1011)