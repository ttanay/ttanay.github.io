+++
title = "Query Processing In Clickhouse"
date = "2024-03-24"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["clickhouse", "study",]
+++

ClickHouse is easily one of the fastest OLAP databases on the face of this planet. I have had the fortune of working on it at different stints. 
This is my attempt to best summarize my understanding of it. 

This post is latest as on [835afe1f342](https://github.com/ClickHouse/ClickHouse/commit/835afe1f342). 

## Introduction
ClickHouse is a columnar, vectorized OLAP database written in C++. 
We will take an example read query, and look at how the query is processed. 

Consider this SQL as the example query:
```sql
-- Schema
CREATE TABLE purchases
(
    `dt` DateTime,
    `customer_id` UInt32,
    `total_spent` Float32
)
ENGINE = MergeTree
ORDER BY dt;

-- Read Query
SELECT
    floor(total_spent) AS s,
    count(*) AS n,
    bar(n, 0, 350000, 50)
FROM purchases
GROUP BY s
ORDER BY s ASC
```

![overview](/query_processing_clickhouse_overview.png)

When we run the SELECT query, at a high-level the following happens:
1. The ClickHouse server receives the query and calls [`executeQuery`](https://github.com/ClickHouse/ClickHouse/blob/835afe1f342ebbb42a39d55d19a4f95df2691978/src/Interpreters/executeQuery.h#L66-L71) which executes the query. 
2. The query string is parsed by the ClickHouse Parser to create an Abstract Syntax Tree(AST) representation of the query. 
3. An Interpreter is created to interpret this AST by doing the following:
    1. Creates a logical query plan by rewriting the AST for binding symbols with objects from the catalog, applying settings, and making logical query optimizations.
    2. Creates the physical query plan by applying optimizations to the logical query plan.  
4. An execution graph is created from this physical query plan. 
5. An executor executes the execution graph in multiple threads(depending on the settings). 

Query processing can be broken down broadly into the following stages:
1. Parser 
2. Building Query Plan 
3. Building Query Pipeline
4. Executing the query.

Let's look at the data structures used. We will then see how their are built in future posts. 

## AST 
The AST data structure is a tree. 
Each type of AST subclasses [`IAST`](https://github.com/ClickHouse/ClickHouse/blob/835afe1f342ebbb42a39d55d19a4f95df2691978/src/Parsers/IAST.h) and describes data specific to the grammar. Eg: [`ASTSelectQuery`](https://github.com/ClickHouse/ClickHouse/blob/835afe1f342ebbb42a39d55d19a4f95df2691978/src/Parsers/ASTSelectQuery.h) for a Select Query.  
```cpp
class IAST : public std::enabled_shared_from_this<IAST> , public TypePromotion<IAST> 
{
public:
    ASTs children; 
};
```
The AST for our query looks something like:
```
┌─explain─────────────────────────────────────┐
│ SelectWithUnionQuery (children 1)           │
│  ExpressionList (children 1)                │
│   SelectQuery (children 4)                  │
│    ExpressionList (children 3)              │
│     Function floor (alias s) (children 1)   │
│      ExpressionList (children 1)            │
│       Identifier total_spent                │
│     Function count (alias n) (children 1)   │
│      ExpressionList (children 1)            │
│       Asterisk                              │
│     Function bar (children 1)               │
│      ExpressionList (children 4)            │
│       Identifier n                          │
│       Literal UInt64_0                      │
│       Literal UInt64_350000                 │
│       Literal UInt64_50                     │
│    TablesInSelectQuery (children 1)         │
│     TablesInSelectQueryElement (children 1) │
│      TableExpression (children 1)           │
│       TableIdentifier purchases             │
│    ExpressionList (children 1)              │
│     Identifier s                            │
│    ExpressionList (children 1)              │
│     OrderByElement (children 1)             │
│      Identifier s                           │
└─────────────────────────────────────────────┘
```


## Logical Query Plan 
The logical query plan is built by performing the following:
1. Rewriting the AST by applying logical query optimizations. Eg: Predicate Pushdown, Constant Folding, etc.  
2. Applying AST-level settings like [`count_distinct_implementation`](https://clickhouse.com/docs/en/operations/settings/settings#count_distinct_implementation).
3. Binding all symbols/identifiers to their corresponding objects from the database catalog.

The logical query plan - [`QueryPlan`](https://github.com/ClickHouse/ClickHouse/blob/835afe1f342ebbb42a39d55d19a4f95df2691978/src/Processors/QueryPlan/QueryPlan.h) is a tree with each node being a [`IQueryPlanStep`](https://github.com/ClickHouse/ClickHouse/blob/835afe1f342ebbb42a39d55d19a4f95df2691978/src/Processors/QueryPlan/IQueryPlanStep.h). Each clause is in the query plan is an object of `IQueryPlanStep`. Eg: [`JoinStep`](https://github.com/ClickHouse/ClickHouse/blob/835afe1f342ebbb42a39d55d19a4f95df2691978/src/Processors/QueryPlan/JoinStep.h),[`SortingStep`](https://github.com/ClickHouse/ClickHouse/blob/835afe1f342ebbb42a39d55d19a4f95df2691978/src/Processors/QueryPlan/SortingStep.h).
```cpp
class QueryPlan 
{
    struct Node
    {
        QueryPlanStepPtr step; 
        std::vector<Node *> children = {};
    };
    // Optimizes the logical query plan
    void optimize(const QueryPlanOptimizationSettings & optimization_settings); 
    // Builds the physical query plan 
    QueryPipelineBuilderPtr buildQueryPipeline(
        const QueryPlanOptimizationSettings & optimization_settings,
        const BuildQueryPipelineSettings & build_pipeline_settings);
}
```
The nodes in the query plan are also connected with data streams. 
```cpp
// Logical data stream containing description of the stream
class DataStream {};
using DataStreams = std::vector<DataStream>; 
class IQueryPlanStep
{
public:
    /// Add processors from current step to QueryPipeline.
    virtual QueryPipelineBuilderPtr updatePipeline(QueryPipelineBuilders pipelines, const BuildQueryPipelineSettings & settings) = 0;
protected:
    DataStreams input_streams; 
    std::optional<DataStream> output_stream;
}
```

The logical query plan for our query looks something like this: 
```
┌─explain──────────────────────────────────────────────────────┐
│ Expression ((Projection + Before ORDER BY [lifted up part])) │
│   Sorting (Sorting for ORDER BY)                             │
│     Expression (Before ORDER BY)                             │
│       Aggregating                                            │
│         Expression (Before GROUP BY)                         │
│           ReadFromPreparedSource (Read from NullSource)      │
└──────────────────────────────────────────────────────────────┘
```

## Physical Query Plan
The physical query plan is called Query Pipeline in ClickHouse.
To translate the logical query plan into a Query Pipeline, `QueryPlan::buildQueryPipeline` is called.
This performs optimizations on the logical query plan. 
After optimizations are done, each step is translated into one or more sets of processors using `IQueryPlanStep::updatePipeline` in an inorder traversal of the tree. 

The query pipeline is a tree with each node being a `IProcessor`. Each processor represents a stage in the query execution. They are closely linked to the actual execution. Eg: Depending on the specifics, `SortingStep` is transformed to `PartialSortingTransform`, `MergeSortedTransform` and `MergeSortingTransform` in the query pipeline.

```cpp
class QueryPipeline {
private: 
    std::shared_ptr<Processors> processors;

    InputPort * input = nullptr; 
    OutputPort * output = nullptr; 
}; 

class IProcessor
{
protected:
    InputPorts inputs; 
    OutputPorts outputs; 

public:
    enum class Status 
    {
        NeedData,
        PortFull, 
        Finished, 
        Ready, 
        Async, 
        ExpandPipeline,
    };

    virtual Status prepare(const PortNumbers & updated_input_ports, const PortNumbers & updated_output_ports);
    virtual void work(); 
    virtual int schedule(); 
    virtual Processors expandPipeline(); 
}
```
`IProcessor::Status` is used to signal whether data should be pushed to a processors' port or not. This is used while executing the query. We will look at this in detail in a future post about the query execution. 
The query pipeline for our query looks something like: 
```
┌─explain────────────────────────────────────┐
│ (Expression)                               │
│ ExpressionTransform                        │
│   (Sorting)                                │
│   MergingSortedTransform 16 _ 1            │
│     MergeSortingTransform _ 16             │
│       LimitsCheckingTransform _ 16         │
│         PartialSortingTransform _ 16       │
│           (Expression)                     │
│           ExpressionTransform _ 16         │
│             (Aggregating)                  │
│             Resize 1 _ 16                  │
│               AggregatingTransform         │
│                 (Expression)               │
│                 ExpressionTransform        │
│                   (ReadFromPreparedSource) │
│                   NullSource 0 _ 1         │
└────────────────────────────────────────────┘
```
Now that we have looked at the data structures involved in query processing, we will look at how these data structures are built and used in the next post. 
