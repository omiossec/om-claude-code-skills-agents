--- 
name: azure-policy-assignment
description: conventions and guidelines for Azure policy assignment structures, including parameterization, scope management
argument-hint: "[check|fix]" 
--- 

# Azure Policy Assignment, exempltions and remediation

## When to use this skill

Use this skill when:
- Designing or audit policy assignments and their scope
- Questions about governance, compliance, or policy exemptions

---

## 1. Policy Assignments

Each assignment must include:
- **`name`** — semi-readable, max 24 characters (portal-visible ID)
- **`displayName`** — full human-readable name shown in Portal
- **`description`** — what compliance requirement or risk this assignment addresses
- **`metadata`** — optionally link to work items (ADO, GitHub, Jira ticket IDs)

**Scope rules:**
- Assign at the **highest Management Group** level possible so inherited subscriptions are automatically covered
- Never assign at the subscription or resource group level when MG assignment is possible
- The default subscription location parameter must be set at or below the security-oriented assignment scope

**Managed Identity (required for `Modify` and `DeployIfNotExists`):**
- **System-assigned MI** — preferred when the MI is used only for this single assignment; cannot be reused outside the assignment
- **User-assigned MI** — preferred when reducing the total number of role assignments matters (one MI, many assignments)
- The MI must be granted the Azure roles specified in the policy's `roleDefinitionIds`

---

## 2. Management Group Placement

**Policy assignments:**
- Must be at **MG level** or lower
- Subscriptions inherit assignments from parent MGs — do not duplicate assignments at the subscription level
- If auto-assigned (e.g., Azure Security Benchmark via Defender for Cloud), create a new MG-level assignment to centrally control effects and remove the auto-assigned subscription-level ones

```
Tenant Root Group
└── Top MG  ← deploy all custom definitions here
    ├── Platform MG  ← assign platform policies here
    │   └── Connectivity Subscription
    └── Landing Zones MG  ← assign landing zone policies here
        ├── Corp Subscription
        └── Online Subscription
```

---

## 3. Exemptions

Grant exemptions only with a documented business justification within acceptable risk parameters.

**Two exemption categories:**

| Type | Use When | Duration |
|---|---|---|
| **Mitigated** | A compensating control replaces the policy intent (e.g., public IP allowed on storage because virus scanning + data purge is in place) | Permanent |
| **Waiver** | A team needs time to remediate — deadline-driven | Temporary — set expiry to "Monday after the ETA" |

**Best practices:**
- Always include `metadata` with a **link to a work item** (ADO board, GitHub issue, Jira ticket)
- If an entire subscription needs exempting, prefer using **`notScope`** (Excluded Scope in Portal) on the assignment rather than individual exemptions
- Never delete an assignment that has active exemptions without first removing those exemptions — deletion leaves **orphaned exemptions** that consume quota and cause confusion

```json
"metadata": {
  "requestedBy": "team-name",
  "approvedBy": "security-team",
  "workItemId": "https://dev.azure.com/org/project/_workitems/edit/12345",
  "reason": "Compensating control: storage firewall restricts access to corp network only."
}
```

---

## 4. Operations and Maintenance

**Automate operational tasks separately:**
- Use scripts (PowerShell, Azure CLI) for: triggering remediation, generating compliance reports, cleaning up orphaned exemptions
- Do **not** embed operational commands in CI/CD deployment pipelines

---

## 5. Quick Review Checklist

When writing or reviewing Azure Policy assignments or remediation, verify:

- [ ] Assignments include `name` (≤ 24 chars), `displayName`, `description`, and `metadata`
- [ ] Assignments are at the highest applicable MG level
- [ ] Exemptions have documented justification, work item links, and expiry (waivers)
- [ ] No orphaned exemptions from deleted assignments
- [ ] Operational tasks (remediation, reporting) are separate from deployment pipeline
- [ ] Grant time-bound waivers for teams that need remediation time