--- 
name: nxtools-dsc-linux
description: conventions and guidelines for building nxtools-dsc-linux configurations
argument-hint: "[check|fix]" 
compatibility: Linux
--- 

# nxtools DSC Resources for Linux

## When to use this skill

Use this skill when:
- Writing PowerShell DSC configurations targeting Linux nodes
- Using the nxtools module (Azure/nxtools) with Azure Machine Configuration
- Reviewing or auditing existing Linux DSC configurations
- Asked to manage files, users, groups, services, or packages on Linux via DSC
- Building Guest Configuration policies for Linux

---

## 1. Module Overview and Prerequisites

**nxtools** is a PowerShell DSC module that acts as a POSIX wrapper around Linux commands. It is the recommended resource collection for managing Linux nodes via Azure Machine Configuration (formerly Azure Policy Guest Configuration).

**The 7 DSC resources:**

| Resource | Purpose |
|---|---|
| `nxFile` | Manage files and folders (presence, content, mode, owner, group) |
| `nxFileLine` | Ensure an exact line is present/absent in a file |
| `nxFileContentReplace` | Replace content in a file when a regex pattern matches |
| `nxGroup` | Manage local groups and group membership |
| `nxUser` | Manage local user accounts |
| `nxService` | Manage systemd services (start/stop/enable/disable) |
| `nxScript` | Execute arbitrary PowerShell scripts with full Get/Test/Set logic |

**Additional resource (audit only):**

| Resource | Purpose |
|---|---|
| `nxPackage` | Audit whether a package is installed (apt only, no remediation) |

**Required modules for compilation:**

```powershell
# Install on the Linux compilation machine
Install-Module -Name nxtools -Force
Install-Module -Name PSDesiredStateConfiguration -RequiredVersion 3.0.0-beta1 -AllowPrerelease -Force
Install-Module -Name GuestConfiguration -RequiredVersion 4.1.0 -Force
```

> **Critical:** DSC configurations using nxtools **must be compiled on a Linux machine** with PowerShell 7. Compilation on Windows or macOS fails because the resources use Linux-native calls. The `AzurePolicyForLinux` extension installs PowerShell at a fixed path and does not modify the system PATH — PowerShell is only available to the Guest Configuration agent, not interactively.

**Compilation command:**

```powershell
# On Linux — compile and package for Azure Machine Configuration
New-GuestConfigurationPackage `
    -Name 'MyLinuxConfig' `
    -Configuration './MyConfig.mof' `
    -Type AuditAndSet `
    -Force
```

---

## 2. nxFile — File and Folder Management

Manages the existence, content, permissions, and ownership of a file or directory.

**Key properties:**

| Property | Type | Values / Notes |
|---|---|---|
| `Ensure` | String | `Present` (default), `Absent` |
| `DestinationPath` | String | Absolute path on the Linux node |
| `Type` | String | `File`, `Directory` |
| `Contents` | String | File content to enforce (mutually exclusive with `SourcePath`) |
| `SourcePath` | String | Local path to copy from (mutually exclusive with `Contents`) |
| `Mode` | String | Octal permission string, e.g. `'0644'`, `'0755'` |
| `Owner` | String | Username of the file owner |
| `Group` | String | Group name for the file |
| `Force` | Boolean | Overwrite existing file/directory if it conflicts |

**Example:**

```powershell
Configuration Example {
    Import-DscResource -ModuleName 'nxtools'

    nxFile MyFile {
        Ensure          = 'Present'
        DestinationPath = '/tmp/myfile'
        Type            = 'File'
        Contents        = 'Some content I want to manage here.'
        Mode            = '0644'
        Owner           = 'root'
        Group           = 'root'
    }
}
```

**Best practices:**
- Always specify `Mode` explicitly — do not rely on defaults for security-sensitive files
- Use `Contents` for small, inline content; use `SourcePath` for larger files
- Set `Type = 'Directory'` when managing folders — `Mode` applies to the directory itself
- Use `Force = $true` with caution: it overwrites existing conflicting resources

---

## 3. nxFileLine — Line Presence in a File

Ensures a specific line is present or absent in an existing file. Useful for managing configuration files like `/etc/sudoers` or `/etc/ssh/sshd_config`.

**Key properties:**

| Property | Type | Notes |
|---|---|---|
| `FilePath` | String | Absolute path to the file to modify |
| `ContainsLine` | String | The exact line that must be present |
| `DoesNotContainPattern` | String | Regex — if this pattern is found, it will be removed before adding `ContainsLine` |

**Example — disable TTY requirement for a user in sudoers:**

```powershell
Configuration Example {
    Import-DscResource -ModuleName 'nxtools'

    nxFileLine DoNotRequireTTY {
        FilePath             = '/etc/sudoers'
        ContainsLine         = 'Defaults:monuser !requiretty'
        DoesNotContainPattern = 'Defaults:monuser[ ]+requiretty'
    }
}
```

**Best practices:**
- Use `DoesNotContainPattern` to remove conflicting lines before inserting — prevents duplicate or contradictory entries
- `ContainsLine` is an exact string match — include every character (spaces, punctuation) precisely
- The target file must already exist — nxFileLine does not create files
- For `/etc/sudoers`, always validate syntax separately; a bad sudoers entry can lock out sudo access

---

## 4. nxFileContentReplace — Regex-Based Content Replacement

Replaces content in a file when a regex pattern is matched. More powerful than nxFileLine for structured or formatted configuration files.

**Key properties:**

| Property | Type | Notes |
|---|---|---|
| `FilePath` | String | Absolute path to the file |
| `Ensure` | String | `Present` (replace found matches), `Absent` (remove matched lines) |
| `SearchPattern` | String | Regex pattern to search for |
| `EnsureExpectedPattern` | String | Regex to verify correct state (used for idempotency check) |
| `ReplacementString` | String | Replacement string; supports regex backreferences (`${group}`) |
| `Multiline` | Boolean | Enable multiline regex matching |
| `SimpleMatch` | Boolean | Treat `SearchPattern` as a literal string, not regex |
| `CaseSensitive` | Boolean | Default `$false` |

**Example — remove NOPASSWD entries from sudoers:**

```powershell
Configuration Example {
    Import-DscResource -ModuleName 'nxtools'

    nxFileContentReplace MyFile {
        Ensure                = 'Absent'
        FilePath              = '/etc/sudoers.d/90-cloud-init-users'
        EnsureExpectedPattern = '(?<user>[\w]+)\s(?<hosts>[^=]+)=(?<rule>[^\s]+)\sNOPASSWD:(?<target>.*)'
        SearchPattern         = '(?<user>[\w]+)\s(?<hosts>[^=]+)=(?<rule>[^\s]+)\sNOPASSWD:(?<target>.*)'
        Multiline             = $false
        SimpleMatch           = $false
        ReplacementString     = '${user} ${hosts}=${rule} ${target}'
        CaseSensitive         = $false
    }
}
```

**Best practices:**
- Use named capture groups (`(?<name>...)`) and backreferences (`${name}`) for safe, readable replacements
- Set `EnsureExpectedPattern` to a pattern that matches the *desired* final state — this is what the resource tests to determine compliance
- Prefer `nxFileLine` for simple line insertions; use `nxFileContentReplace` only when regex replacement is needed
- Test regex patterns independently (`Select-String`) before embedding in DSC

---

## 5. nxGroup — Local Group Management

Manages local Linux groups and controls their membership.

**Key properties:**

| Property | Type | Notes |
|---|---|---|
| `Ensure` | String | `Present`, `Absent` |
| `GroupName` | String | Name of the local group |
| `Members` | String[] | Exact list of members — replaces current membership (use `MembersToInclude`/`MembersToExclude` to manage partially) |
| `MembersToInclude` | String[] | Additive — ensure these users are members without affecting others |
| `MembersToExclude` | String[] | Ensure these users are not members |

**Example:**

```powershell
configuration CreateGroupFoobar {
    Import-DscResource -ModuleName 'nxtools'

    node localhost {
        nxGroup CreateGroupFoobar {
            Ensure    = 'Present'
            GroupName = 'foobar'
            Members   = @('root')
        }
    }
}
```

**Best practices:**
- Use `Members` when you need to enforce exact group membership (desired state) — it removes unlisted members
- Use `MembersToInclude` / `MembersToExclude` when other processes also manage group membership
- Ensure referenced users exist before or alongside the group resource — use `DependsOn` if managing users in the same configuration

---

## 6. nxUser — Local User Account Management

Creates, configures, and removes local Linux user accounts.

**Key properties:**

| Property | Type | Notes |
|---|---|---|
| `Ensure` | String | `Present`, `Absent` |
| `UserName` | String | Login name |
| `FullName` | String | GECOS full name field |
| `Description` | String | Additional GECOS description |
| `HomeDirectory` | String | Absolute path to home directory |
| `Password` | PSCredential | Encrypted password (use `@secure()` equivalent) |
| `Disabled` | Boolean | Disable the account without deleting it |
| `PasswordChangeRequired` | Boolean | Force password change on next login |
| `GroupID` | String | Primary group name or GID |

**Example:**

```powershell
configuration CreateUserTest {
    Import-DscResource -ModuleName 'nxtools'

    node localhost {
        nxUser CreateUserTest {
            Ensure        = 'Present'
            UserName      = 'test'
            FullName      = 'test user'
            HomeDirectory = '/home/test'
            Description   = 'Service account for app X'
        }
    }
}
```

**Best practices:**
- If the user must belong to a specific group, create the group first using `nxGroup` with `DependsOn`
- Avoid setting `Password` directly in configurations checked into source control — use Azure Key Vault references or parameter injection
- Use `Disabled = $true` to disable accounts rather than deleting them when audit trails are needed

---

## 7. nxService — Systemd Service Management

Manages the state and startup behavior of systemd services. Currently supports **systemd only**.

**Key properties:**

| Property | Type | Values / Notes |
|---|---|---|
| `Name` | String | Full service unit name, e.g. `'sshd.service'` |
| `State` | String | `'running'`, `'stopped'` |
| `Enabled` | Boolean | `$true` = enabled at boot, `$false` = disabled |

**Example — ensure waagent is stopped but enabled:**

```powershell
configuration waagentStopped {
    Import-DscResource -ModuleName 'nxtools'

    node localhost {
        nxService ManageWaagent {
            Name    = 'waagent.service'
            State   = 'stopped'
            Enabled = $true
        }
    }
}
```

**Best practices:**
- Always include the `.service` suffix in `Name` for explicitness
- `Enabled` and `State` are independent — a service can be `Enabled = $true` (starts at boot) but `State = 'stopped'` (not currently running), which is valid for services that should only run on schedule
- nxService only manages existing services — it does not install them; use `nxPackage` or `nxScript` first if the service may not yet be installed

---

## 8. nxScript — Custom Script Execution

The most flexible resource — executes PowerShell 7 scripts with full Get/Test/Set logic. Use when no other nxtools resource covers the requirement.

**Key properties:**

| Property | Type | Notes |
|---|---|---|
| `GetScript` | ScriptBlock | Must return a Hashtable with a `Reasons` key containing `[Reason[]]` objects |
| `TestScript` | ScriptBlock | Must return `$true` (compliant) or `$false` (non-compliant) |
| `SetScript` | ScriptBlock | Remediation logic — run when `TestScript` returns `$false` |

**The `Reason` class** (required by Azure Machine Configuration for compliance reporting):

```powershell
# Reason objects explain WHY a resource is non-compliant in the portal
$Reason = [Reason]::new()
$Reason.Code   = 'ResourceName:ResourceName:UniqueReasonCode'  # format: Type:Type:Code
$Reason.Phrase = 'Human-readable explanation of the compliance state'
```

**Example — create and validate a file with content:**

```powershell
configuration CreateFileNxScript {
    param (
        [Parameter(Mandatory = $true)]
        [String] $FilePath,

        [Parameter(Mandatory = $true)]
        [String] $FileContent
    )

    Import-DscResource -ModuleName 'nxtools'

    nxScript MyScript {
        GetScript = {
            $Reason = [Reason]::new()
            $Reason.Code   = 'Script:Script:FileMissing'
            $Reason.Phrase = 'File does not exist'

            if (Test-Path -Path $using:FilePath) {
                $text = $(Get-Content -Path $using:FilePath -Raw).Trim()
                if ($text -eq $using:FileContent) {
                    $Reason.Code   = 'Script:Script:Success'
                    $Reason.Phrase = 'File exists with correct content'
                } else {
                    $Reason.Code   = 'Script:Script:ContentMissing'
                    $Reason.Phrase = 'File exists but content does not match'
                }
            }

            return @{ Reasons = @($Reason) }
        }
        TestScript = {
            if (Test-Path -Path $using:FilePath) {
                $text = $(Get-Content -Path $using:FilePath -Raw).Trim()
                return $text -eq $using:FileContent
            }
            return $false
        }
        SetScript = {
            Set-Content -Path $using:FilePath -Value $using:FileContent
        }
    }
}
```

**Best practices:**
- `GetScript` **must** return a Hashtable with `Reasons` — omitting this breaks compliance reporting in the Azure Portal
- Use `$using:` to reference configuration parameters inside script blocks — direct variable access does not work
- Follow the `Type:Type:Code` convention for `Reason.Code` — Azure Machine Configuration enforces this format
- Keep `TestScript` idempotent and side-effect-free — it may be called frequently
- Use nxScript only when other resources cannot satisfy the requirement — it is harder to maintain than purpose-built resources
- Always implement all three blocks (`Get`, `Test`, `Set`) even for audit-only scenarios — `SetScript` can be a no-op, but it must exist

---

## 9. nxPackage — Package Audit (apt only)

**Audit-only** — verifies whether a package is installed. Does not install or remove packages. Only supports **apt** package manager.

```powershell
Configuration Example {
    Import-DscResource -ModuleName 'nxtools'

    nxPackage EnsureCurlInstalled {
        Ensure      = 'Present'
        Name        = 'curl'
        PackageType = 'apt'
    }
}
```

> Use `nxScript` with `apt-get install` / `apt-get remove` if remediation (install/uninstall) is required.

---

## 10. Azure Machine Configuration Context

When deploying configurations via Azure Machine Configuration (Azure Policy Guest Configuration):

**Package types:**
- `Audit` — read-only compliance check, no remediation
- `AuditAndSet` — compliance check + auto-remediation on non-compliant resources

**Configuration packaging workflow:**

```powershell
# 1. Compile the MOF (must run on Linux)
MyConfiguration -OutputPath ./output

# 2. Package for Azure Machine Configuration
New-GuestConfigurationPackage `
    -Name 'MyLinuxConfig' `
    -Configuration './output/localhost.mof' `
    -Type AuditAndSet `
    -Force

# 3. Test locally before uploading
Start-GuestConfigurationPackageRemediation -Path './MyLinuxConfig/MyLinuxConfig.zip'

# 4. Publish to Azure Storage and create policy definition via EPAC or az cli
```

**Managed Identity requirement:** Guest Configuration assignments with `AuditAndSet` type require a **system-assigned managed identity** on the VM with `Guest Configuration Resource Contributor` role.

---

## 11. Best Practices Summary

- **Compile on Linux** — never attempt to compile nxtools configurations on Windows or macOS
- **Use `$using:` for all parameter references** inside `nxScript` blocks — lexical scoping does not apply inside DSC script blocks
- **Always implement `Reason` objects in `GetScript`** — they surface in the Azure Portal compliance view and are required for Machine Configuration
- **Prefer specific resources over nxScript** — use `nxFile`, `nxFileLine`, `nxGroup`, `nxUser`, `nxService` before falling back to `nxScript`
- **Use `DependsOn`** when resources have ordering requirements (e.g., user must exist before being added to a group)
- **Test with `Start-GuestConfigurationPackageRemediation`** locally on a Linux VM before uploading to Azure
- **Pin module versions** — use `-RequiredVersion` when installing nxtools and PSDesiredStateConfiguration to avoid breaking changes
- **Keep configurations small and focused** — one configuration per policy assignment; avoid monolithic configs

---

## 12. Quick Review Checklist

When writing or reviewing nxtools DSC configurations:

- [ ] Configuration will be compiled on a Linux machine with PowerShell 7
- [ ] `Import-DscResource -ModuleName 'nxtools'` is present in every configuration
- [ ] `nxScript.GetScript` returns `@{ Reasons = @($Reason) }` with proper `[Reason]` objects
- [ ] `Reason.Code` follows `Type:Type:Code` format
- [ ] All parameter references inside script blocks use `$using:varName`
- [ ] `nxFile` specifies `Mode`, `Owner`, and `Group` explicitly for security-sensitive files
- [ ] `nxService.Name` includes the `.service` suffix
- [ ] `nxGroup.Members` vs `MembersToInclude` — correct property chosen based on whether full membership control is intended
- [ ] `nxPackage` is used only for audit; `nxScript` is used if installation remediation is required
- [ ] `DependsOn` is set where resource ordering matters (user before group membership, package before service)
- [ ] Configuration packaged with `New-GuestConfigurationPackage` before deployment
- [ ] `AuditAndSet` packages have a system-assigned managed identity configured on the assignment
- [ ] Tested locally with `Start-GuestConfigurationPackageRemediation` before publishing
