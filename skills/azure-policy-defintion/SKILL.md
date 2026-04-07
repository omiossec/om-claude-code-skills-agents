--- 
name: azure-policy-defintion
description: conventions and guidelines for building Azure Policy definitions and initiatives
argument-hint: "[check|fix]" 
--- 

## When to use this skill

Use this skill when:
- Writing or reviewing Azure Policy definitions (custom policies or initiatives)
- Building a Policy as Code (PaC) pipeline
- Asked to audit, refactor, or improve existing policy structures

---

## 1. Policy vs. RBAC — Know the Difference

Use the right tool for the right job:

| Tool | Controls |
|---|---|
| **Azure Policy** | Prohibits **anyone and any service** from doing something regardless of identity |
| **Azure RBAC** | Prohibits **specific users and service principals** from doing something |

Policy governs **what resources can look like** (configuration, compliance, security posture).
RBAC governs **who can do what** (actions and data operations).

Do not use Policy as a substitute for RBAC, or vice versa.

---

## 2. Policy Effects

Effects determine what happens when a policy rule matches. Choose the most appropriate effect and always make it a parameter.

**Effect hierarchy (least to most impactful):**

`Disabled` → `Audit` / `AuditIfNotExists` → `Append` / `Modify` → `Deny` / `DeployIfNotExists` → `DenyAction`

**Always parameterize the effect** — use the standard parameter name `effect` with displayName `Effect`:

```json
"parameters": {
  "effect": {
    "type": "String",
    "metadata": {
      "displayName": "Effect",
      "description": "Enable or disable the execution of the policy."
    },
    "allowedValues": [ "Deny", "Audit", "Disabled" ],
    "defaultValue": "Audit"
  }
}
```

**Recommended effect combinations and defaults:**

| Scenario | Allowed Values | Default |
|---|---|---|
| Full control available | `Append`, `Deny`, `Audit`, `Disabled` | `Append` |
| Deny not possible | `Append`, `Audit`, `Disabled` | `Append` |
| Modify preferred | `Modify`, `Deny`, `Audit`, `Disabled` | `Modify` |
| Modify only | `Modify`, `Audit`, `Disabled` | `Modify` |
| Deny preferred | `Deny`, `Audit`, `Disabled` | `Audit` |
| Deny not possible | `Audit`, `Disabled` | `Audit` |
| Remediation available | `DeployIfNotExists`, `AuditIfNotExists`, `Disabled` | `AuditIfNotExists` |
| Audit only | `AuditIfNotExists`, `Disabled` | `AuditIfNotExists` |
| Delete protection | `DenyAction`, `Disabled` | `DenyAction` |

> **Warning:** `Modify` and `Append` effects can interfere with desired-state IaC tools like Terraform. If using Terraform, add `ignore_changes` blocks for policy-managed properties.

---

## 3. Custom Policy Definitions

**Before creating a custom policy — ask yourself:**
- Does a built-in policy already cover this need?
- Is a built-in policy close enough to fork or request an update?
- Have you checked the [Azure/Community-Policy](https://github.com/Azure/Community-Policy) repo?

If you still need a custom policy, follow these rules:

**Naming:**
- Use a **GUID** or a unique **company name prefix** — avoids collisions in multi-tenant environments and community contributions
- Never use generic names that could conflict with built-in policies

**Required structure — always include:**

```json
{
  "name": "<GUID or company-prefixed name>",
  "properties": {
    "displayName": "Human readable name shown in Portal",
    "description": "What this policy does and why it exists.",
    "metadata": {
      "version": "1.0.0",
      "category": "Security"
    },
    "mode": "All",
    "parameters": { ... },
    "policyRule": { ... }
  }
}
```

- `displayName` — required; shown in Portal
- `description` — highly recommended; explain intent and impact
- `metadata.version` — use **semantic versioning** (`major.minor.patch`)
- `metadata.category` — must match an existing built-in category (e.g., `Security`, `Compute`, `Storage`, `Network`)

**Excluded properties** — never set these manually (Azure manages them):
- `properties.policyType`
- `properties.metadata.createdOn`
- `properties.metadata.createdBy`
- `properties.metadata.updatedOn`
- `properties.metadata.updatedBy`

**For multi-tenant environments:** propagate the same definition to every tenant — do not maintain separate copies per tenant (WET anti-pattern). A single source of truth avoids copy/paste drift.

---

## 4. Policy Set Definitions (Initiatives)

Follow the same naming and structure rules as individual policy definitions.

**Keep initiative count low:**
- Limit to **no more than 5 initiatives** (including custom ones) per scope
- More initiatives create maintenance burden and make exemption management exponentially harder

**Surfacing effect parameters in initiatives:**
- Prefix the policy-level parameter name with an initiative-specific indicator to avoid collisions
- When using GUID-named policies, use `policyDefinitionReferenceId` as a short readable alias

```json
"policyDefinitionGroups": [...],
"parameters": {
  "myInitiative_storageHttpsEffect": {
    "type": "String",
    "defaultValue": "Audit"
  }
}
```

---

## 5. Management Group Placement

**Custom policy definitions:**
- Deploy at the **top Management Group** of each tenant (a single MG directly under "Tenant root group" with no siblings, or directly at Tenant root group)
- This ensures definitions are visible and reusable across all child MGs and subscriptions

## 7. Policy as Code (PaC)

A mature PaC pipeline must satisfy these properties:

| Property | Description |
|---|---|
| **Idempotent** | Running the pipeline multiple times produces the same result |
| **Desired State** | Automatically reverses any manual drift from the declared state |
| **DRY** | No copy/paste of definitions across files, repos, or tenants |
| **Team Co-existence** | Different teams can own different policy areas without conflicts |
| **CI/CD Integration** | Follows a branching strategy (e.g., GitHub Flow) with PR reviews |
| **Round-trip** | Existing live assignments can be exported and re-ingested |
| **Minimized Code** | Reduces hand-authored JSON/Bicep to the minimum necessary |


**Separate CI/CD from Operations:**
- CI/CD pipelines deploy and update policy resources
- Operational tasks (remediation runs, compliance report generation) should be **separate scripts** — not baked into the deployment pipeline

---

## 9. Phased Rollout Strategy

Never deploy a new policy directly in `Deny` or `DeployIfNotExists` mode in production.

**Recommended rollout sequence:**

1. **Assess** — run a compliance scan with `Audit` effect to understand current non-compliance baseline
2. **Identify** — determine which teams and resources fall outside compliance
3. **Communicate** — notify affected teams; set remediation deadlines
5. **Enforce** — switch effect to `Deny` or `DeployIfNotExists` after waiver deadlines pass
6. **Remediate** — trigger remediation tasks for `DeployIfNotExists` policies on existing resources

---

## 10. Operations and Maintenance

**Validate policy definitions before deploying:**
- Use `Confirm-PolicyDefinitionIsValid.ps1` (from EPAC) to catch structural errors
- Use `Out-FormattedPolicyDefinition.ps1` to normalize formatting before committing

---

## 11. Quick Review Checklist

When writing or reviewing Azure Policy artifacts, verify:

- [ ] Policy vs. RBAC — correct tool chosen for the requirement
- [ ] `effect` is a parameter named `effect` with standard allowed values and a safe default (`Audit` not `Deny`)
- [ ] Custom policy has `displayName`, `description`, `metadata.version`, and `metadata.category`
- [ ] `metadata.category` matches an existing built-in category
- [ ] No manually set system properties (`policyType`, `createdOn`, etc.)
- [ ] GUID or company prefix used for custom definition names
- [ ] Initiative count at any scope is ≤ 5
- [ ] Managed Identity configured for `Modify` / `DeployIfNotExists` policies
- [ ] New policies rolled out in `Audit` before switching to `Deny`
- [ ] PaC pipeline is idempotent, DRY, and version-controlled