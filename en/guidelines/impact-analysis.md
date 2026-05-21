# Impact Analysis Guidelines

## Goal

Use the cortex-product-graph MCP to identify not only the diff of the PR but also **the places that should have been changed but were not**.

## Investigation Steps

### 1. Identify the Changed Nodes

Search cortex-product-graph for the files and functions modified in the PR and capture their `qualifiedName`.

```text
mcp__cortex-product-graph__search_product_graph_nodes(query: "<changed function name>")
```

### 2. Graph Traversal (Structural Connections)

For each changed node, follow edges in both forward and backward directions and enumerate the nodes that may be affected.

```text
mcp__cortex-product-graph__trace_product_graph_connections(
  start_node: "<qualifiedName>",
  direction: "both",
  max_depth: 3
)
```

### 3. Semantic Search (Functional Similarity)

Describe the business context of the change (the content of `@graph-business`, or the intent of the change) in natural language and search for nodes that implement similar functionality. This surfaces code that handles the same pattern or the same concept even when it is not directly connected on the graph.

```text
mcp__cortex-product-graph__search_product_graph_nodes(
  query: "<business-level description of the change>",
  search_mode: "semantic"
)
```

Example uses:
- Changed the validation logic of a function → discover other functions performing similar validation.
- Changed an error-handling pattern → discover other places using the same pattern.
- Changed the schema of a BQ table → discover functions that use the same table from a different context.
- Changed the shape of an API response → discover frontend code that consumes the same data.

### 4. Detecting Missed Edits

From the results of the graph traversal and the semantic search, treat anything not present in the PR's changed files as a "missed-edit candidate".

Raise a finding when the result matches one of the following patterns:

| Pattern | Description | Example |
|---------|-------------|---------|
| **Untransmitted type or interface change** | A type definition changed, but a function consuming that type was not updated. | Added a field to `KpiSummary` → `formatKpiSummary()` was not updated. |
| **`via` field drift** | A BQ / Firestore column was added or renamed, but the `via:` of `@graph-connects` was not updated. | Added a column to a table → the repository's `via:` is still stale. |
| **One-sided `shares_topic` change** | The publisher of a Pub/Sub topic was changed but the subscriber was not. | Changed the message shape on the publisher → the subscriber still expects the old shape. |
| **`documented_by` drift** | The documentation that corresponds to the code change was not updated. | New feature added → the matching document under `docs/` was not updated. |
| **Unpropagated Repository → UseCase → Handler chain** | A change to the return type of a lower layer was not reflected in an upper layer. | Repository return type changed → the usecase still uses the old type. |
| **Tests not updated** | The test file for the changed function is not included in the PR. | `calculateBugRate()` was changed → `calculateBugRate.test.ts` was not. |
| **Similar implementations not updated** | Code matching the same pattern found via semantic search was not modified. | Fixed the per-team aggregation logic → a similar aggregation in another stack still uses the old logic. |

### 5. When Not to Raise a Finding

- Indirect connections more than four hops away from the changed node.
- Utility functions annotated with `@graph-connects none` (no external connections).
- Nodes that live in a different stack and a different deploy unit (the impact exists but does not need to be fixed in the same PR).
- PRs that only update documentation.
- Nodes whose semantic-search similarity is low (distance of 0.4 or higher).
