# Use Skip token for pagination with PowerShell

only when the query return more than 1000 lines.

```powershell
function Get-AllResourceGraphResults {
    param(
        [Parameter(Mandatory)]
        [string]$Query,
        [string[]]$Subscriptions
    )

    $results   = [System.Collections.Generic.List[object]]::new()
    $skipToken = $null

    do {
        $params = @{ Query = $Query; First = 1000 }
        if ($Subscriptions) { $params.Subscription = $Subscriptions }
        if ($skipToken)     { $params.SkipToken    = $skipToken }

        try {
            $page = Search-AzGraph @params -ErrorAction Stop
        }
        catch {
            Write-Error "Query failed: $($_.Exception.Message)"
            return $results
        }

        $results.AddRange($page.Data)
        $skipToken = $page.SkipToken

    } while ($skipToken)

    return $results
}
``` 