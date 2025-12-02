# Query Planning

<cite>
**Referenced Files in This Document**   
- [LogicalPlan.scala](file://community/cypher/cypher-logical-plans/src/main/scala/org/neo4j/cypher/internal/logical/plans/LogicalPlan.scala)
- [IDPQueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/idp/IDPQueryGraphSolver.scala)
- [QueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/QueryGraphSolver.scala)
- [QueryGraph.scala](file://community/cypher/ir/src/main/scala/org/neo4j/cypher/internal/ir/QueryGraph.scala)
- [CardinalityCostModel.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/CardinalityCostModel.scala)
- [PhysicalPlanner.scala](file://community/cypher/physical-planning/src/main/scala/org/neo4j/cypher/internal/physicalplanning/PhysicalPlanner.scala)
- [QueryPlanner.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/QueryPlanner.scala)
- [NFA.scala](file://community/cypher/cypher-logical-plans/src/main/scala/org/neo4j/cypher/internal/logical/plans/NFA.scala)
- [LogicalPlanGenerator.scala](file://community/cypher/logical-plan-generator/src/main/scala/org/neo4j/cypher/internal/logical/generator/LogicalPlanGenerator.scala)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Query Planning Architecture](#query-planning-architecture)
3. [QueryGraph and Declarative Query Representation](#querygraph-and-declarative-query-representation)
4. [Logical Planning Process](#logical-planning-process)
5. [IDPQueryGraphSolver and Optimal Plan Selection](#idpquerygraphsolver-and-optimal-plan-selection)
6. [Domain Model of Logical Plans](#domain-model-of-logical-plans)
7. [Cost-Based Optimization and Cardinality Estimation](#cost-based-optimization-and-cardinality-estimation)
8. [Physical Planning and Execution Runtime](#physical-planning-and-execution-runtime)
9. [Query Pattern Translation Examples](#query-pattern-translation-examples)
10. [Common Planning Issues and Performance Considerations](#common-planning-issues-and-performance-considerations)
11. [Query Optimization Best Practices](#query-optimization-best-practices)
12. [Conclusion](#conclusion)

## Introduction

The Cypher query planning component is responsible for transforming parsed Cypher queries into executable logical plans through a sophisticated optimization process. This document provides a comprehensive analysis of the query planning architecture, focusing on the logical planning process that converts declarative query patterns into optimal execution strategies. The planner employs a cost-based optimization approach, utilizing the QueryGraph data structure to represent query patterns and the IDPQueryGraphSolver to find optimal execution plans through dynamic programming techniques. The system integrates with the execution runtime to deliver efficient query processing while handling complex graph patterns, subqueries, and various MATCH clauses.

## Query Planning Architecture

The Cypher query planning architecture follows a multi-stage compilation process that transforms high-level Cypher queries into executable physical plans. The architecture is organized into distinct components that handle different aspects of the planning process, from initial query analysis to final plan execution.

```mermaid
graph TB
subgraph "Query Compilation Pipeline"
Parser[Query Parser] --> AST[Abstract Syntax Tree]
AST --> IR[Intermediate Representation]
IR --> LogicalPlanner[Logical Planner]
LogicalPlanner --> CostOptimizer[Cost-Based Optimizer]
CostOptimizer --> PhysicalPlanner[Physical Planner]
PhysicalPlanner --> ExecutionRuntime[Execution Runtime]
end
subgraph "Optimization Components"
QueryGraph[QueryGraph] --> IDPSolver[IDPQueryGraphSolver]
IDPSolver --> CostModel[CardinalityCostModel]
CostModel --> Statistics[GraphStatistics]
end
subgraph "Plan Representation"
LogicalPlan[LogicalPlan Hierarchy]
PhysicalPlan[PhysicalPlan]
end
Parser --> QueryGraph
LogicalPlanner --> QueryGraph
CostOptimizer --> IDPSolver
PhysicalPlanner --> LogicalPlan
ExecutionRuntime --> PhysicalPlan
style QueryCompilationPipeline fill:#f9f,stroke:#333
style OptimizationComponents fill:#bbf,stroke:#333
style PlanRepresentation fill:#f96,stroke:#333
```

**Diagram sources**
- [QueryPlanner.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/QueryPlanner.scala)
- [IDPQueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/idp/IDPQueryGraphSolver.scala)
- [CardinalityCostModel.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/CardinalityCostModel.scala)

**Section sources**
- [QueryPlanner.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/QueryPlanner.scala)
- [PhysicalPlanner.scala](file://community/cypher/physical-planning/src/main/scala/org/neo4j/cypher/internal/physicalplanning/PhysicalPlanner.scala)

## QueryGraph and Declarative Query Representation

The QueryGraph is a fundamental data structure that represents the declarative query patterns extracted from Cypher queries. It serves as an intermediate representation that captures the essential elements of a query, including patterns, predicates, and relationships, without specifying the execution strategy. The QueryGraph acts as the input to the planning process, enabling the planner to analyze and optimize query patterns independently of their execution.

```mermaid
classDiagram
class QueryGraph {
+patternNodes : Set[String]
+patternRelationships : Set[PatternRelationship]
+argumentIds : Set[LogicalVariable]
+selections : Set[Expression]
+shortestRelationshipPatterns : Set[ShortestRelationshipPattern]
+optionalMatches : Set[QueryGraph]
+subQueries : Set[IRExpression]
+connectedComponents : Seq[QueryGraph]
+contains : (other : QueryGraph) Boolean
+addSelections : (expressions : Set[Expression]) QueryGraph
+addPattern : (pattern : PatternRelationship) QueryGraph
}
class PatternRelationship {
+name : String
+left : String
+right : String
+types : Seq[RelTypeName]
+direction : SemanticDirection
+length : PatternLength
+properties : Option[Expression]
}
class ShortestRelationshipPattern {
+left : String
+right : String
+relationship : String
+single : Boolean
+optional : Boolean
+predicates : Set[Expression]
}
class IRExpression {
+query : PlannerQuery
+computedScopeDependencies : Option[Set[LogicalVariable]]
}
QueryGraph --> PatternRelationship : "contains"
QueryGraph --> ShortestRelationshipPattern : "contains"
QueryGraph --> IRExpression : "contains"
```

**Diagram sources**
- [QueryGraph.scala](file://community/cypher/ir/src/main/scala/org/neo4j/cypher/internal/ir/QueryGraph.scala)
- [LogicalPlan.scala](file://community/cypher/cypher-logical-plans/src/main/scala/org/neo4j/cypher/internal/logical/plans/LogicalPlan.scala)

**Section sources**
- [QueryGraph.scala](file://community/cypher/ir/src/main/scala/org/neo4j/cypher/internal/ir/QueryGraph.scala)

## Logical Planning Process

The logical planning process transforms the declarative QueryGraph representation into executable logical plans through a systematic approach that considers various execution strategies and selects the optimal one based on cost estimates. The process begins with the decomposition of the query into connected components, which are planned independently before being connected through appropriate join operations.

```mermaid
flowchart TD
Start([Start Planning]) --> Decompose["Decompose QueryGraph into\nConnected Components"]
Decompose --> CheckEmpty{"Empty Components?"}
CheckEmpty --> |Yes| PlanEmpty["Plan Empty Components\nusing Argument Scan"]
CheckEmpty --> |No| PlanComponents["Plan Each Component\nusing SingleComponentPlanner"]
PlanEmpty --> Connect["Connect Components\nusing Join Strategies"]
PlanComponents --> Connect
Connect --> Optional["Handle Optional Matches\nusing Outer Joins"]
Optional --> Subqueries["Plan Subqueries\nusing ExistsSubqueryPlanner"]
Subqueries --> Shortest["Plan Shortest Path Patterns\nusing StatefulShortestPath"]
Shortest --> Order["Apply Ordering Requirements\nusing SortPlanner"]
Order --> Complete([Complete Logical Plan])
```

**Diagram sources**
- [IDPQueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/idp/IDPQueryGraphSolver.scala)
- [QueryPlanner.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/QueryPlanner.scala)

**Section sources**
- [IDPQueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/idp/IDPQueryGraphSolver.scala)

## IDPQueryGraphSolver and Optimal Plan Selection

The IDPQueryGraphSolver is the core component responsible for finding optimal execution strategies through an iterative dynamic programming approach. Based on the paper "Iterative Dynamic Programming: A New Class of Query Optimization Algorithms" by Donald Kossmann and Konrad Stocker, this solver employs a systematic method to explore the space of possible execution plans and select the most efficient one based on cost estimates.

```mermaid
sequenceDiagram
participant Planner as "QueryPlanner"
participant Solver as "IDPQueryGraphSolver"
participant Component as "SingleComponentPlanner"
participant Connector as "ComponentConnector"
participant CostModel as "CardinalityCostModel"
Planner->>Solver : plan(queryGraph, interestingOrder, context)
Solver->>Solver : decompose into connectedComponents
alt Empty Components
Solver->>Solver : planEmptyComponent(queryGraph, context, kit)
else Components Exist
loop For Each Component
Solver->>Component : planComponent(qg, context, kit, interestingOrder)
Component-->>Solver : BestPlans
end
end
Solver->>Connector : connectComponentsAndSolveOptionalMatch(plannedComponents, queryGraph, interestingOrder, context, kit)
Connector->>CostModel : evaluate plan costs
CostModel-->>Connector : cost estimates
Connector-->>Solver : BestPlans
Solver-->>Planner : BestPlans
```

**Diagram sources**
- [IDPQueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/idp/IDPQueryGraphSolver.scala)
- [CardinalityCostModel.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/CardinalityCostModel.scala)

**Section sources**
- [IDPQueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/idp/IDPQueryGraphSolver.scala)

## Domain Model of Logical Plans

The domain model of logical plans is built around a hierarchy of plan nodes that represent different operations in the query execution process. The LogicalPlan trait serves as the base class for all plan nodes, with various specialized implementations for different operations such as scans, joins, aggregations, and pattern matching.

```mermaid
classDiagram
class LogicalPlan {
<<abstract>>
+id : Id
+lhs : Option[LogicalPlan]
+rhs : Option[LogicalPlan]
+localAvailableSymbols : Set[LogicalVariable]
+availableSymbols : Set[LogicalVariable]
+distinctness : Distinctness
+isLeaf : Boolean
+readOnly : Boolean
+isUpdatingPlan : Boolean
+flatten : Seq[LogicalPlan]
+indexUsage : Seq[IndexUsage]
}
class LogicalLeafPlan {
<<abstract>>
+argumentIds : Set[LogicalVariable]
+usedVariables : Set[LogicalVariable]
+addArgumentIds : Set[LogicalVariable]
+withoutArgumentIds : Set[LogicalVariable]
+removeArgumentIds : LogicalLeafPlan
}
class LogicalUnaryPlan {
<<abstract>>
+source : LogicalPlan
+withLhs : LogicalPlan
}
class LogicalBinaryPlan {
<<abstract>>
+left : LogicalPlan
+right : LogicalPlan
+withLhs : LogicalPlan
+withRhs : LogicalPlan
}
class AllNodesScan {
+idName : LogicalVariable
+argumentIds : Set[LogicalVariable]
}
class NodeByLabelScan {
+idName : LogicalVariable
+label : LabelName
+argumentIds : Set[LogicalVariable]
+indexOrder : IndexOrder
}
class Expand {
+source : LogicalPlan
+from : LogicalVariable
+dir : SemanticDirection
+types : Seq[RelTypeName]
+to : LogicalVariable
+rel : LogicalVariable
+mode : ExpansionMode
}
class Selection {
+source : LogicalPlan
+predicate : Expression
}
class Projection {
+source : LogicalPlan
+projectExpressions : Map[LogicalVariable, Expression]
}
class Aggregation {
+source : LogicalPlan
+groupingExpressions : Map[LogicalVariable, Expression]
+aggregationExpressions : Map[LogicalVariable, Expression]
}
class Apply {
+left : LogicalPlan
+right : LogicalPlan
}
class ValueHashJoin {
+left : LogicalPlan
+right : LogicalPlan
+predicate : Expression
}
LogicalPlan <|-- LogicalLeafPlan
LogicalPlan <|-- LogicalUnaryPlan
LogicalPlan <|-- LogicalBinaryPlan
LogicalLeafPlan <|-- AllNodesScan
LogicalLeafPlan <|-- NodeByLabelScan
LogicalUnaryPlan <|-- Expand
LogicalUnaryPlan <|-- Selection
LogicalUnaryPlan <|-- Projection
LogicalUnaryPlan <|-- Aggregation
LogicalBinaryPlan <|-- Apply
LogicalBinaryPlan <|-- ValueHashJoin
```

**Diagram sources**
- [LogicalPlan.scala](file://community/cypher/cypher-logical-plans/src/main/scala/org/neo4j/cypher/internal/logical/plans/LogicalPlan.scala)

**Section sources**
- [LogicalPlan.scala](file://community/cypher/cypher-logical-plans/src/main/scala/org/neo4j/cypher/internal/logical/plans/LogicalPlan.scala)

## Cost-Based Optimization and Cardinality Estimation

The cost-based optimization process relies on accurate cardinality estimation to compare different execution plans and select the most efficient one. The CardinalityCostModel evaluates the cost of each plan based on estimated row counts, operation complexity, and data access patterns, using graph statistics to inform its estimates.

```mermaid
flowchart TD
Start([Start Cost Calculation]) --> GetBatchSize["Determine Execution Model\nand Batch Size"]
GetBatchSize --> CalculateEffective["Calculate Effective\nCardinalities"]
CalculateEffective --> LhsCost["Calculate LHS Cost\nRecursively"]
CalculateEffective --> RhsCost["Calculate RHS Cost\nRecursively"]
LhsCost --> Combine["Combine Costs Based on\nPlan Type"]
RhsCost --> Combine
Combine --> PlanType{"Plan Type?"}
PlanType --> |CartesianProduct| Cartesian["Cost = lhsCost +\n(lhsCardinality * rhsExecutions * rhsCost)"]
PlanType --> |ApplyPlan| Apply["Cost = lhsCost +\n(lhsCardinality * rhsCost)"]
PlanType --> |HashJoin| Hash["Cost = lhsCost + rhsCost +\n(lhsCardinality * PROBE_BUILD_COST) +\n(rhsCardinality * PROBE_SEARCH_COST)"]
PlanType --> |Other| Default["Cost = inputCardinality *\ncostPerRow + lhsCost + rhsCost"]
Cartesian --> Monitor["Report Plan Cost and\nEffective Cardinality"]
Apply --> Monitor
Hash --> Monitor
Default --> Monitor
Monitor --> Complete([Return Total Cost])
```

**Diagram sources**
- [CardinalityCostModel.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/CardinalityCostModel.scala)

**Section sources**
- [CardinalityCostModel.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/CardinalityCostModel.scala)

## Physical Planning and Execution Runtime

The physical planning phase transforms the optimized logical plan into a physical execution plan that can be executed by the runtime engine. This process involves slot allocation, expression variable allocation, and pipeline breaking decisions that optimize memory usage and execution efficiency.

```mermaid
sequenceDiagram
participant LogicalPlanner as "Logical Planner"
participant PhysicalPlanner as "PhysicalPlanner"
participant ExpressionAllocator as "ExpressionVariableAllocation"
participant SlotAllocator as "SlotAllocation"
participant Rewriter as "SlottedRewriter"
participant Runtime as "Execution Runtime"
LogicalPlanner->>PhysicalPlanner : plan(tokenContext, logicalPlan, semanticTable, breakingPolicy, config, anonymousVariableNameGenerator, cancellationChecker)
PhysicalPlanner->>ExpressionAllocator : allocate(logicalPlan)
ExpressionAllocator-->>PhysicalPlanner : Result(logicalPlan, nExpressionSlots, availableExpressionVars)
PhysicalPlanner->>PhysicalPlanner : slottedParameters(logicalPlan)
PhysicalPlanner->>PhysicalPlanner : computeLiveVariables(logicalPlan, breakingPolicy)
PhysicalPlanner->>SlotAllocator : allocateSlots(withSlottedParameters, semanticTable, breakingPolicy, availableExpressionVars, config, anonymousVariableNameGenerator, liveVariables, cancellationChecker, allocatePipelinedSlots)
SlotAllocator-->>PhysicalPlanner : slotMetaData
PhysicalPlanner->>Rewriter : apply(withSlottedParameters, slotMetaData.slotConfigurations, slotMetaData.trailPlans)
Rewriter-->>PhysicalPlanner : finalLogicalPlan
PhysicalPlanner-->>LogicalPlanner : PhysicalPlan(finalLogicalPlan, nExpressionSlots, slotConfigurations, argumentSizes, applyPlans, trailPlans, nestedPlanArgumentConfigurations, availableExpressionVariables, parameterMapping)
LogicalPlanner->>Runtime : execute(PhysicalPlan)
```

**Diagram sources**
- [PhysicalPlanner.scala](file://community/cypher/physical-planning/src/main/scala/org/neo4j/cypher/internal/physicalplanning/PhysicalPlanner.scala)

**Section sources**
- [PhysicalPlanner.scala](file://community/cypher/physical-planning/src/main/scala/org/neo4j/cypher/internal/physicalplanning/PhysicalPlanner.scala)

## Query Pattern Translation Examples

This section illustrates how different Cypher query patterns are translated into logical operators during the planning process. Each example demonstrates the transformation from a declarative query pattern to its corresponding logical plan representation.

### MATCH Clause Translation

The MATCH clause is translated into a series of Expand operations that traverse the graph according to the specified pattern. The planner considers various strategies, including index seeks, label scans, and relationship scans, to optimize the traversal.

```mermaid
flowchart TD
MatchQuery["MATCH (p:Person)-[:ACTED_IN]->(m:Movie)\nWHERE m.year > 2000\nRETURN p.name, m.title"] --> Parse["Parse Query and\nBuild QueryGraph"]
Parse --> QueryGraph["QueryGraph:\n- patternNodes: {p, m}\n- patternRelationships: {(p)-[:ACTED_IN]->(m)}\n- selections: {m.year > 2000}\n- labels: {Person, Movie}"]
QueryGraph --> PlanComponents["Plan Connected Components"]
PlanComponents --> NodeScan["Plan Node Scans:\n- NodeByLabelScan(p, Person)\n- NodeByLabelScan(m, Movie)"]
NodeScan --> Expand["Plan Pattern Expansion:\n- Expand from p to m via :ACTED_IN"]
Expand --> Selection["Add Selection Filter:\n- Selection(m.year > 2000)"]
Selection --> Projection["Add Projection:\n- Projection(p.name, m.title)"]
Projection --> Result["Complete Plan"]
```

**Section sources**
- [LogicalPlan.scala](file://community/cypher/cypher-logical-plans/src/main/scala/org/neo4j/cypher/internal/logical/plans/LogicalPlan.scala)
- [IDPQueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/idp/IDPQueryGraphSolver.scala)

### OPTIONAL MATCH Translation

The OPTIONAL MATCH clause is translated into an outer join operation that preserves rows from the left side even when no matching patterns are found on the right side. The planner uses a LeftOuterHashJoin to implement this behavior efficiently.

```mermaid
flowchart TD
OptionalQuery["MATCH (p:Person)\nOPTIONAL MATCH (p)-[:DIRECTED]->(m:Movie)\nRETURN p.name, m.title"] --> Parse["Parse Query and\nBuild QueryGraph"]
Parse --> QueryGraph["QueryGraph:\n- patternNodes: {p}\n- optionalMatches: {QueryGraph with (p)-[:DIRECTED]->(m)}\n- labels: {Person}"]
QueryGraph --> PlanMain["Plan Main Query Component"]
PlanMain --> MainScan["NodeByLabelScan(p, Person)"]
PlanMain --> PlanOptional["Plan Optional Component"]
PlanOptional --> OptionalScan["NodeByLabelScan(m, Movie)"]
OptionalScan --> OptionalExpand["Expand from p to m via :DIRECTED"]
PlanOptional --> OptionalJoin["LeftOuterHashJoin(p, m)"]
MainScan --> OptionalJoin
OptionalJoin --> Projection["Projection(p.name, m.title)"]
Projection --> Result["Complete Plan"]
```

**Section sources**
- [LogicalPlan.scala](file://community/cypher/cypher-logical-plans/src/main/scala/org/neo4j/cypher/internal/logical/plans/LogicalPlan.scala)
- [IDPQueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/idp/IDPQueryGraphSolver.scala)

### Subquery Translation

Subqueries, particularly those used in EXISTS expressions, are planned using specialized strategies that consider the interaction between the outer query and the subquery. The planner uses an ExistsSubqueryPlanner to optimize these patterns.

```mermaid
flowchart TD
SubqueryQuery["MATCH (p:Person)\nWHERE EXISTS {\n MATCH (p)-[:ACTED_IN]->(:Movie {year: 2020})\n}\nRETURN p.name"] --> Parse["Parse Query and\nBuild QueryGraph"]
Parse --> QueryGraph["QueryGraph:\n- patternNodes: {p}\n- selections: {EXISTS subquery}\n- subQueries: {IRExpression with inner query}\n- labels: {Person}"]
QueryGraph --> PlanOuter["Plan Outer Query"]
PlanOuter --> OuterScan["NodeByLabelScan(p, Person)"]
OuterScan --> PlanExists["Plan EXISTS Subquery"]
PlanExists --> ExistsPlanner["ExistsSubqueryPlanner.planInnerOfExistsSubquery()"]
ExistsPlanner --> InnerQueryGraph["Inner QueryGraph:\n- patternNodes: {p, m}\n- patternRelationships: {(p)-[:ACTED_IN]->(m)}\n- selections: {m.year = 2020}\n- labels: {Movie}"]
InnerQueryGraph --> InnerPlan["Plan Inner Query\nusing IDPQueryGraphSolver"]
InnerPlan --> SemiApply["Wrap in SemiApply\nwith outer plan as left"]
SemiApply --> Projection["Projection(p.name)"]
Projection --> Result["Complete Plan"]
```

**Section sources**
- [IDPQueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/idp/IDPQueryGraphSolver.scala)
- [QueryPlanner.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/QueryPlanner.scala)

## Common Planning Issues and Performance Considerations

Several common issues can affect query planning performance and result in suboptimal execution strategies. Understanding these issues and their root causes is essential for effective query optimization and performance tuning.

### Suboptimal Plan Selection

Suboptimal plan selection occurs when the planner chooses an execution strategy that is not the most efficient for the given query and data distribution. This can happen due to inaccurate cardinality estimates, missing statistics, or limitations in the cost model.

```mermaid
flowchart TD
Issue["Suboptimal Plan Selection"] --> Causes["Root Causes"]
Causes --> Cardinality["Inaccurate Cardinality\nEstimation"]
Causes --> Statistics["Missing or Outdated\nGraph Statistics"]
Causes --> CostModel["Limitations in Cost Model\nor Heuristics"]
Causes --> Indexes["Missing or Inappropriate\nIndexes"]
Causes --> Complexity["Query Complexity Exceeding\nPlanner Capabilities"]
Solutions["Mitigation Strategies"] --> Cardinality --> Stats["Update Graph Statistics\nusing ANALYZE"]
Solutions --> Statistics --> Stats
Solutions --> CostModel --> Hints["Use Planner Hints\n(eg, USE INDEX)"]
Solutions --> Indexes --> CreateIndex["Create Appropriate Indexes"]
Solutions --> Complexity --> Simplify["Simplify Query Structure\nor Break into Parts"]
Monitoring["Monitoring and Detection"] --> Explain["Use EXPLAIN to Analyze\nExecution Plan"]
Monitoring --> Profile["Use PROFILE to Identify\nPerformance Bottlenecks"]
Monitoring --> Metrics["Monitor Query Performance\nMetrics and Statistics"]
```

**Section sources**
- [CardinalityCostModel.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/CardinalityCostModel.scala)
- [IDPQueryGraphSolver.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/idp/IDPQueryGraphSolver.scala)

### Cardinality Estimation Errors

Cardinality estimation errors are a primary cause of suboptimal plan selection. The planner relies on accurate row count estimates to compare different execution strategies, and errors in these estimates can lead to poor plan choices.

```mermaid
flowchart TD
EstimationError["Cardinality Estimation Errors"] --> Sources["Error Sources"]
Sources --> Independence["Assumption of Independence\nbetween Predicates"]
Sources --> Uniformity["Assumption of Uniform\nData Distribution"]
Sources --> Correlation["Failure to Account for\nData Correlations"]
Sources --> Sampling["Insufficient Sampling\nfor Statistics"]
Sources --> Updates["Stale Statistics due to\nData Modifications"]
Impact["Performance Impact"] --> NestedLoops["Excessive Nested Loop\nIterations"]
Impact --> Memory["Excessive Memory Usage\ndue to Large Hash Tables"]
Impact --> IOPS["Excessive I/O Operations\ndue to Full Scans"]
Impact --> Time["Increased Query Execution\nTime"]
Prevention["Prevention Strategies"] --> Analyze["Regular Use of ANALYZE\nCommand"]
Prevention --> Indexes["Create Indexes on\nFrequently Filtered Properties"]
Prevention --> Hints["Use Cardinality Hints\nfor Complex Queries"]
Prevention --> Monitoring["Monitor Estimation Accuracy\nin Query Profiles"]
Prevention --> Sampling["Increase Statistics\nSampling Rate"]
```

**Section sources**
- [CardinalityCostModel.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/CardinalityCostModel.scala)
- [QueryGraph.scala](file://community/cypher/ir/src/main/scala/org/neo4j/cypher/internal/ir/QueryGraph.scala)

## Query Optimization Best Practices

Effective query optimization requires understanding both the capabilities of the query planner and the characteristics of the underlying data. The following best practices can help ensure optimal query performance.

### Writing Planner-Friendly Queries

Writing queries that the planner can optimize effectively involves following certain patterns and avoiding common pitfalls that can hinder optimization.

```mermaid
flowchart TD
BestPractices["Query Optimization Best Practices"] --> Structure["Query Structure"]
Structure --> Specific["Be Specific with Labels\nand Relationship Types"]
Structure --> Early["Filter Early in the Query\nusing WHERE clauses"]
Structure --> Simple["Keep Queries Simple\nand Focused"]
Structure --> Parameters["Use Parameters Instead\nof Literal Values"]
Structure --> Limit["Use LIMIT to Reduce\nResult Set Size"]
Indexes["Index Utilization"] --> Create["Create Indexes on\nFrequently Queried Properties"]
Indexes --> Composite["Use Composite Indexes for\nMultiple Property Queries"]
Indexes --> Constraints["Use Unique Constraints for\nHigh-Cardinality Properties"]
Indexes --> Array["Consider Indexing Array\nProperties when Appropriate"]
Patterns["Pattern Matching"] --> Avoid["Avoid Cartesian Products\nwith Multiple MATCH clauses"]
Patterns --> Direction["Specify Relationship\nDirection when Possible"]
Patterns --> Optional["Use OPTIONAL MATCH Sparingly\nand with Filters"]
Patterns --> Shortest["Use shortestPath with\nAppropriate Constraints"]
Monitoring["Performance Monitoring"] --> Explain["Use EXPLAIN to Understand\nExecution Plans"]
Monitoring --> Profile["Use PROFILE to Identify\nBottlenecks"]
Monitoring --> Stats["Monitor Query Performance\nand Statistics"]
Monitoring --> Analyze["Regularly Run ANALYZE\nto Update Statistics"]
```

**Section sources**
- [LogicalPlan.scala](file://community/cypher/cypher-logical-plans/src/main/scala/org/neo4j/cypher/internal/logical/plans/LogicalPlan.scala)
- [CardinalityCostModel.scala](file://community/cypher/cypher-planner/src/main/scala/org/neo4j/cypher/internal/compiler/planner/logical/CardinalityCostModel.scala)

## Conclusion

The Cypher query planning component represents a sophisticated optimization system that transforms declarative graph queries into efficient execution plans. By leveraging the QueryGraph data structure to represent query patterns and the IDPQueryGraphSolver to find optimal execution strategies through dynamic programming, the planner effectively balances query complexity with performance requirements. The integration of cost-based optimization, cardinality estimation, and physical planning ensures that queries are executed efficiently across diverse data distributions and access patterns. Understanding the planner's behavior, including its handling of various query patterns and potential optimization challenges, is essential for developing high-performance graph applications. By following query optimization best practices and monitoring plan selection, developers can ensure that their queries leverage the full capabilities of the Neo4j query engine.