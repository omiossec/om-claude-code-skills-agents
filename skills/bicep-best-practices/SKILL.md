--- 
name: bicep-best-practices
description: conventions and guidelines for building bicep templates
argument-hint: "[check|fix]" 
--- 

# Bicep Best Practices

## When to use this skill

Use this skill when:
- Writing new Bicep templates or modules
- Reviewing existing Bicep code for quality issues
- Asked to audit, refactor, or improve Bicep files
- Someone asks "is this good Bicep?", "review my Bicep", or "write Bicep for X"

---

## 1. Parameters

**Rules to follow:**
- Use clear, descriptive names — avoid abbreviations that aren't universally understood
- Only use parameters for values that **change between deployments**; use variables or hard-coded values for stable settings
- Provide safe, low-cost **default values** (e.g., `Standard` SKU not `Premium`) so test deployments don't incur unexpected costs
- Use `@allowed` **sparingly** — overly restrictive allowed lists break valid deployments and go stale as Azure adds new SKUs
- Always add `@description()` decorators that explain what the value is for and any constraints
- Set `@minLength` / `@maxLength` on parameters that feed into resource names to catch errors early
- Place parameter declarations at the **top of the file**

```bicep
@description('Short name of the application, used to construct resource names.')
@minLength(2)
@maxLength(10)
param appName string

@description('Environment name. Drives SKU selection and naming.')
@allowed(['dev', 'test', 'prod'])
param environment string = 'dev'

@description('App Service SKU. Use B1 for non-prod to control cost.')
param appServiceSku string = 'B1'
```

---

## 2. Variables

**Rules to follow:**
- Do **not** declare a type for variables — Bicep infers it from the resolved value
- Extract **complex expressions** out of resource properties and into variables — keeps resource blocks clean and readable
- Use variables to avoid duplicating the same expression in multiple places
- You can use any Bicep function inside a variable declaration

```bicep
// Good — logic lives in a variable, resource property stays clean
var appServicePlanName = '${appName}-${environment}-plan'
var isProduction = environment == 'prod'
var skuTier = isProduction ? 'Standard' : 'Basic'
```

---

## 3. Naming Conventions

**Rules to follow:**
- Use **lower camelCase** for all symbolic names (parameters, variables, resources, modules, outputs): `storageAccount`, `appServicePlan`
- Use `uniqueString()` to generate stable, deterministic uniqueness — pass `resourceGroup().id` to scope it to the resource group
- Build resource names using **template expressions** that combine a meaningful prefix with `uniqueString()`:

```bicep
// Ensures the name is meaningful AND unique
var storageAccountName = 'st${appName}${environment}${uniqueString(resourceGroup().id)}'
```

- Add a prefix to `uniqueString()` output for resources (e.g., storage accounts) that **cannot start with a number**
- Do **not** include `Name` in symbolic names — the symbolic name represents the resource, not its ARM name:

```bicep
// Bad
resource storageAccountName 'Microsoft.Storage/storageAccounts@2023-05-01' = { ... }

// Good
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = { ... }
```

- Do **not** distinguish parameters from variables by appending a type suffix (no `paramAppName` or `varSku`)
- Use a resource-type **prefix or suffix** in the ARM resource *name* to make resources identifiable in the portal: `st` for storage, `plan` for App Service plans, `app` for web apps, `kv` for Key Vault, etc.

---

## 4. Resource Definitions

**Rules to follow:**
- Always use a **recent, stable API version** for each resource type. Check the Azure docs or use `az bicep decompile` to identify the latest version
- Access other resources via their **symbolic name** instead of `reference()` or `resourceId()` — this is safer and creates implicit dependencies automatically:

```bicep
// Bad — brittle, loses implicit dependency
var storageId = resourceId('Microsoft.Storage/storageAccounts', storageAccountName)

// Good — Bicep tracks the dependency
var storageId = storageAccount.id
```

- Prefer **implicit dependencies** (reading a property from another resource's symbolic name) over explicit `dependsOn` — only use `dependsOn` when there is no property relationship to express the dependency
- Use the **`existing` keyword** to reference resources not deployed in this file instead of hard-coding IDs:

```bicep
resource existingKeyVault 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: keyVaultName
}
```

- Use resource **properties as outputs** rather than reconstructing values (e.g., use `app.properties.defaultHostName` not `'${appName}.azurewebsites.net'`)

---

## 5. Child Resources

**Rules to follow:**
- Avoid nesting more than **two levels deep** — deep nesting hurts readability
- Use the **`parent` property** or inline nesting rather than constructing child resource names manually — Bicep handles the name concatenation for you:

```bicep
// Good — use parent property
resource blobContainer 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = {
  parent: blobService
  name: 'data'
  properties: {}
}
```

---

## 6. Outputs

**Rules to follow:**
- Mark sensitive outputs with `@secure()` to prevent values from appearing in deployment logs or the portal:

```bicep
@secure()
output storageKey string = storageAccount.listKeys().keys[0].value
```

- Prefer using the **`existing` keyword** in the consuming template to look up resource properties rather than passing them as outputs — you always get fresh data
- Do output resource **IDs, names, and URLs** that other templates or pipelines need — but never output secrets unless absolutely required and decorated with `@secure()`

---

## 7. Modularization

**Rules to follow:**
- Break large templates into **focused modules** — one module per logical infrastructure component (networking, storage, compute, identity)
- Each module should accept **parameters** for everything that varies — never hard-code environment-specific values inside a module
- Modules should expose useful **outputs** (IDs, names, hostnames) so the root template can wire them together
- Keep the root `main.bicep` as an orchestrator that calls modules; avoid putting resource definitions directly in it for large deployments

```bicep
// main.bicep
module network './modules/network.bicep' = {
  name: 'networkDeploy'
  params: {
    location: location
    vnetAddressPrefix: '10.0.0.0/16'
  }
}

module appService './modules/appService.bicep' = {
  name: 'appServiceDeploy'
  params: {
    location: location
    subnetId: network.outputs.appSubnetId
  }
}
```

---

## 8. Loops and Conditions

**Rules to follow:**
- Use `for` expressions to deploy multiple similar resources instead of duplicating resource blocks:

```bicep
param storageAccountNames array = ['data', 'logs', 'backup']

resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-05-01' = [for name in storageAccountNames: {
  name: 'st${name}${uniqueString(resourceGroup().id)}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}]
```

- Use `if` conditions to conditionally deploy resources based on parameters — avoids maintaining separate templates per environment:

```bicep
resource appInsights 'Microsoft.Insights/components@2020-02-02' = if (environment == 'prod') {
  name: 'appi-${appName}-prod'
  location: location
  kind: 'web'
  properties: { Application_Type: 'web' }
}
```

---

## 9. Testing and CI/CD

**Rules to follow:**
- Run `bicep build <file>.bicep` (linting) before committing — catches syntax errors and many best-practice violations
- Use `New-AzResourceGroupDeployment -Whatif` to preview changes before applying them
- Integrate Bicep deployment into CI/CD pipelines with automated linting (`bicep build`) and validation (`az deployment group validate`) steps
- Use the **Bicep linter** rules in `bicepconfig.json` to enforce team conventions automatically:

```json
{
  "analyzers": {
    "core": {
      "enabled": true,
      "rules": {
        "no-unused-params": { "level": "error" },
        "no-unused-vars": { "level": "error" },
        "prefer-interpolation": { "level": "warning" },
        "secure-parameter-default": { "level": "error" }
      }
    }
  }
}
```

---

## 10. Documentation

**Rules to follow:**
- Use `@description()` on every **parameter and output** — this is surfaced in Azure Portal and tooling
- Use `//` inline comments to explain **non-obvious logic**, not what the code plainly says
- Include a comment block at the top of each module explaining its **purpose, inputs, and outputs**
- Maintain a README for complex deployments describing prerequisites, parameter guidance, and deployment order

```bicep
// =====================================================================
// Module: appService.bicep
// Purpose: Deploys an App Service Plan and Web App with managed identity
// Inputs:  location, appName, environment, subnetId
// Outputs: appId, appHostname
// =====================================================================
```

---

## 11. Quick Review Checklist

When reviewing or generating Bicep code, verify each of these:

- [ ] Parameters are at the top of the file with `@description()` decorators
- [ ] No `@allowed` list that is likely to go stale
- [ ] Default values are safe for non-production environments
- [ ] Variables extract complex expressions from resource bodies
- [ ] Symbolic names use lower camelCase and do not contain `Name`
- [ ] Resource names use `uniqueString()` combined with meaningful prefixes
- [ ] API versions are recent and stable
- [ ] No use of `reference()` or `resourceId()` where symbolic names could be used
- [ ] `dependsOn` is only used when no property-level relationship exists
- [ ] Sensitive outputs are decorated with `@secure()`
- [ ] Child resources use `parent` property — no manual name construction
- [ ] Large templates are split into modules
- [ ] `for` loops replace duplicated resource blocks
- [ ] `bicepconfig.json` linter rules are enabled
- [ ] Comments explain *why*, not *what*
