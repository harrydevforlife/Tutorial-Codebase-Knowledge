# Chapter 3: Validation

In the previous chapter, [Executor](02_executor_.md), we met the "Project Manager" that takes your `Query` struct and orchestrates getting the data. We saw that one of its first jobs is to check your request. But why?

Imagine you're building a house. You wouldn't want the construction crew to start building based on faulty blueprints or unclear instructions, right? That would lead to wasted effort, potential safety hazards, and a house that might not even stand up!

Similarly, when you ask `metricsview` for data, you want to be sure your request is sensible and possible *before* it tries to run a potentially complex and time-consuming query on your database. This checking process is called **Validation**.

## What is Validation?

Validation in `metricsview` is like having a careful building inspector review your plans *before* construction starts. It's a set of checks designed to catch potential problems early. It ensures that:

1.  **Your Metrics View Definition is Sound:** The underlying "blueprint" for your data (the `MetricsViewSpec` definition, often in a YAML file) makes sense.
2.  **Your Query is Well-Formed:** The specific question you're asking (your [Query Definition (`Query` struct)](01_query_definition___query__struct__.md)) is valid according to the blueprint and the rules of the database.

The main goal is to prevent errors, ensure the results you get are correct, and avoid running queries that are guaranteed to fail.

## What Does `metricsview` Validate?

The "inspector" checks several things for both the blueprint (metrics view definition) and the construction plan (your query):

**1. Validating the Metrics View Definition (The Blueprint)**

This usually happens when `metricsview` first loads or understands your metrics view definition. It checks things like:

*   **Does the main table exist?** (`table: my_data` in your definition) - Does the table `my_data` actually exist in the database you specified?
*   **Does the time dimension exist and have the right type?** (`timeseries: event_timestamp`) - If you specified a time column, does `event_timestamp` exist in `my_data`, and is it actually a date or timestamp column?
*   **Do columns in security rules exist?** - If you set up rules like "only show data where `user_country` is 'Canada'", does the `user_country` column actually exist?
*   **Database-specific rules:** Some databases have unique rules. For example, ClickHouse might not allow a dimension named `revenue` if there's also a *column* named `revenue` in the table and the dimension uses a custom `expression`. Validation checks for these dialect-specific quirks ([Dialect Abstraction](07_dialect_abstraction_.md)).

Let's look conceptually at how `metricsview` might check the time dimension using information from the database:

```go
// Simplified concept from executor_validate.go

// mv is the MetricsView definition
// olap is the database connection
func checkTimeDimension(mv *MetricsViewSpec, olap drivers.OLAPStore) error {
    if mv.TimeDimension == "" {
        return nil // No time dimension specified, nothing to check
    }

    // Ask the database about the table structure
    tableInfo, err := olap.InformationSchema().Lookup(..., mv.Table)
    if err != nil {
        return fmt.Errorf("table %q not found", mv.Table)
    }

    // Check if the time dimension column exists in the table info
    found := false
    correctType := false
    for _, column := range tableInfo.Schema.Fields {
        if strings.ToLower(column.Name) == strings.ToLower(mv.TimeDimension) {
            found = true
            // Check if the column type is TIMESTAMP or DATE
            if column.Type.Code == runtimev1.Type_CODE_TIMESTAMP || column.Type.Code == runtimev1.Type_CODE_DATE {
                correctType = true
            }
            break
        }
    }

    if !found {
        return fmt.Errorf("time dimension %q not found in table %q", mv.TimeDimension, mv.Table)
    }
    if !correctType {
        return fmt.Errorf("time dimension %q is not a timestamp/date column", mv.TimeDimension)
    }
    return nil // Looks good!
}
```
This code (conceptually) asks the database for information about the table specified in the metrics view. It then checks if the column named as the `TimeDimension` exists and if its data type is suitable for time-based analysis.

**2. Validating the Query (The Construction Plan)**

When you submit a specific [Query Definition (`Query` struct)](01_query_definition___query__struct__.md) to the [Executor](02_executor_.md), another round of validation occurs. This checks:

*   **Do requested dimensions/measures exist?** - If your query asks for `SUM(revenue)` grouped by `country`, does the metrics view definition actually *have* a measure called `total_revenue` (or similar mapping) and a dimension called `country`?
*   **Are filters valid?** - Are the conditions in `Where` and `Having` clauses logical and correctly formed? (More on this in [Chapter 5: Expression Handling](05_expression_handling_.md)).
*   **Does the query respect security?** - Are you allowed to see the dimensions and measures you asked for based on the security rules?
*   **Are there incompatible options?** - As mentioned in Chapter 1, you can't ask for raw rows (`Rows: true`) *and* specific dimension groupings (`Dimensions: [...]`) in the same query. The `Query.Validate()` method checks for some of these basic inconsistencies.

```go
// Simplified concept of Query validation inside the Executor

func (e *Executor) Query(ctx context.Context, qry *Query, ...) (*drivers.Result, error) {
    // --- Basic Query struct validation (from Chapter 1) ---
    err := qry.Validate() // Checks for incompatible options like Rows + Dimensions
    if err != nil {
        return nil, fmt.Errorf("basic query validation failed: %w", err)
    }

    // --- More detailed validation against the Metrics View ---
    // Check if all requested dimensions exist in e.metricsView.Dimensions
    for _, requestedDim := range qry.Dimensions {
        found := false
        for _, definedDim := range e.metricsView.Dimensions {
            if requestedDim.Name == definedDim.Name {
                // Also check if user has access via e.security
                if !e.security.CanAccessDimension(requestedDim.Name) {
                     return nil, fmt.Errorf("access denied for dimension: %s", requestedDim.Name)
                }
                found = true
                break
            }
        }
        if !found {
            return nil, fmt.Errorf("dimension %q not defined in metrics view %q", requestedDim.Name, e.metricsView.Name)
        }
    }

    // Check if all requested measures exist in e.metricsView.Measures (similar logic)
    // ... check measures ...

    // ... other checks ...

    // If all checks pass, proceed to rewriting, AST building, etc.
    // ... build AST ...
    // ... generate SQL ...
    // ... execute query ...

    return result, nil
}
```
This conceptual code shows the [Executor](02_executor_.md) checking if each dimension requested in the `Query` actually exists in the `metricsView` definition it loaded, and also checking security rules (`e.security`). If a requested dimension isn't found or isn't allowed, it returns an error immediately, saving time and resources.

## How Does Validation Fit In?

Validation happens early in the process, primarily within the [Executor](02_executor_.md) *before* it starts building the complex database query ([Abstract Syntax Tree (AST)](04_abstract_syntax_tree__ast__.md)) or talking to the database extensively.

Here's a simplified view of where validation fits:

```mermaid
sequenceDiagram
    participant User
    participant Ex as Executor
    participant Val as Validation Logic
    participant DBInfo as Database Schema Info
    participant ASTBuilder as AST Builder

    User->>+Ex: Submit Query(myQuery)
    Ex->>+Val: Validate Query & Metrics View Def
    Note over Val, DBInfo: Checks like: Table exists? Columns exist? Dim/Measure defined? Security OK? Dialect rules?
    Val->>+DBInfo: Get Table Schema
    DBInfo-->>-Val: Schema Details
    alt Validation Fails
        Val-->>-Ex: Validation Error!
        Ex-->>-User: Return Error (e.g., "Dimension 'xyz' not found")
    else Validation Passes
        Val-->>-Ex: Validation OK
        Ex->>+ASTBuilder: Build AST based on validated query
        ASTBuilder-->>-Ex: AST created
        Note over Ex: Continue with SQL generation...
    end

```

1.  You submit your `Query` to the `Executor`.
2.  The `Executor` calls its internal `Validation Logic` (like functions in `executor_validate.go`).
3.  The validation logic might need to check the `Database Schema Info` (e.g., column types).
4.  If any check fails, an error is returned immediately to you.
5.  If all checks pass, the `Executor` proceeds to the next steps, like building the [Abstract Syntax Tree (AST)](04_abstract_syntax_tree__ast__.md).

## Conclusion

Validation is the essential "safety check" in `metricsview`. It acts like an inspector for both your data blueprint (metrics view definition) and your specific request (query). By catching errors, inconsistencies, and potential problems *before* executing the query, validation saves time, prevents confusing results, and ensures that `metricsview` operates reliably. It checks for things like missing columns, incorrect types, undefined dimensions/measures, and violations of security or database rules.

Now that we know our request is valid, how does the [Executor](02_executor_.md) translate it into a structured plan that can eventually become a database query? That's where the concept of the Abstract Syntax Tree comes in.

Let's explore this internal representation in the next chapter: [Chapter 4: Abstract Syntax Tree (AST)](04_abstract_syntax_tree__ast__.md).

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)