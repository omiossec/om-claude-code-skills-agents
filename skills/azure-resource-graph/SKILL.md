---
name: azure-resource-graph
description: Write and review Azure Resource Graph queries using KQL, with PowerShell execution including pagination, error handling, and parallel batch processing
argument-hint: "[check|fix]"
---

# Azure Resource Graph Skill

## When to use this skill

Use this skill when:
- Writing Azure Resource Graph KQL queries to explore and analyze Azure resources at scale
- Reviewing existing Azure Resource Graph queries for quality issues
- Asked to audit, refactor, or improve Azure Resource Graph queries
- Using PowerShell to execute Resource Graph queries.

---

## Query Language

Azure Resource Graph uses a **subset of KQL**. Not all KQL operators are supported.

### Supported Operators
`where` · `project` · `project-away` · `extend` · `summarize` · `join` · `union` · `distinct` · `count` · `top` · `sort` / `order` · `mv-expand` · `parse` · `limit` / `take`

### Key Tables
| Table | Covers |
|---|---|
| `Resources` | Default — most ARM resource types |
| `ResourceContainers` | Management groups, subscriptions, resource groups |
| `PolicyResources` | `Microsoft.PolicyInsights` |
| `SecurityResources` | `Microsoft.Security` |
| `NetworkResources` | `Microsoft.Network` |
| `AuthorizationResources` | `Microsoft.Authorization` |
| `AdvisorResources` | `Microsoft.Advisor` |
| `HealthResources` | `Microsoft.ResourceHealth` |

Always name the source table explicitly — do not rely on `Resources` being the implicit default.

---

## join

### Supported Flavors
`innerunique` (default) · `inner` · `leftouter` · `fullouter`

### Hard Limits
- Maximum **3 `join` + `union` operations combined** per query.
- The same table **cannot appear as the right-side table more than once**.
- Custom join strategies (e.g., broadcast) are not supported.

### Patterns

**Cross-table join — enrich resources with subscription name:**
```kusto
Resources
| where type == 'microsoft.keyvault/vaults'
| project name, location, resourceGroup, subscriptionId
| join kind=leftouter (
    ResourceContainers
    | where type == 'microsoft.resources/subscriptions'
    | project subscriptionName=name, subscriptionId
) on subscriptionId
| project name, location, resourceGroup, subscriptionName
```
> Always include the join key in `project` when projecting before or after a join.

**Multi-table join (stay within the 3-operation limit):**
```kusto
Resources
| where type == 'microsoft.compute/virtualmachines'
| project vmName=name, subscriptionId, resourceGroup, vmId=id
| join kind=leftouter (
    SecurityResources
    | where type == 'microsoft.security/assessments'
    | project vmId=tostring(properties.resourceDetails.Id), severity=tostring(properties.metadata.severity)
) on vmId
| project vmName, resourceGroup, severity
```

---

## summarize

`summarize` is **simplified** in Resource Graph — it operates on the first result page only. Do not rely on it for full-dataset aggregations without also scoping with `where`.

**Common aggregation patterns:**
```kusto
// Count VMs by power state
Resources
| where type == 'microsoft.compute/virtualmachines'
| summarize count() by tostring(properties.extended.instanceView.powerState.code)

// Count resources by type and location
Resources
| summarize resourceCount=count() by type, location
| sort by resourceCount desc

// Count with condition
Resources
| summarize
    total=count(),
    encrypted=countif(tostring(properties.encryption.status) == 'enabled')
  by type
```

**Combine summarize with join** (count before joining to stay within limits):
```kusto
Resources
| summarize vmCount=count() by subscriptionId
| join kind=leftouter (
    ResourceContainers
    | where type == 'microsoft.resources/subscriptions'
    | project subscriptionName=name, subscriptionId
) on subscriptionId
| project subscriptionName, vmCount
| sort by vmCount desc
```

---

## PowerShell

### Basic Query with Error Handling
```powershell
function Invoke-ResourceGraphQuery {
    param(
        [Parameter(Mandatory)]
        [string]$Query,
        [string[]]$Subscriptions
    )

    $params = @{ Query = $Query }
    if ($Subscriptions) { $params.Subscription = $Subscriptions }

    try {
        Search-AzGraph @params -ErrorAction Stop
    }
    catch [Microsoft.Azure.Commands.ResourceManager.Cmdlets.Utilities.Graph.GraphErrorException] {
        Write-Error "Resource Graph query error: $($_.Exception.Message)"
    }
    catch {
        Write-Error "Unexpected error: $($_.Exception.Message)"
    }
}
```

### Pagination with Skip Token
Resource Graph returns a maximum of 1,000 rows per call. Use `-SkipToken` to retrieve all pages.

 See [skip-token.md](references/skip-token.md) for full example reference.



### Parallel Batch Processing
Use `ForEach-Object -Parallel` to process large result sets concurrently. Set `-ThrottleLimit` to respect API throttling.

 See [paralle.md](references/paralle.md) for full example reference.


---

## Best Practices

- **Filter early** — use `where` before `join` and `project` to reduce rows before expensive operations.
- **Always `project`** — select only the columns you need; avoids large payloads and speeds up queries.
- **Escape dots** in property names: `properties['odata.type']`; escape `$` in PowerShell with a backtick.
- **Handle 429** — if throttled, honor the `Retry-After` header before retrying.
- **Scope queries** — pass an explicit subscription list rather than querying all accessible subscriptions when possible.
- **`summarize` scope** — remember it only operates on the first result page; pre-filter with `where` to make results meaningful.
- **`mv-expand` limit** — maximum 3 per query; RowLimit capped at 2,000 (default 128).
