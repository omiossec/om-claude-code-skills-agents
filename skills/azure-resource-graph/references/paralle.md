# Parallel Batch Processing

To be used to execute several query in parallel

```powershell
$BatchQueries = @( 
    @{ 
        Query = "Resources | where type =~ 'Microsoft.Compute/virtualMachines'" 
        Subscriptions = @("SUB_A") # Query 1 Scope 
    }, 
    @{ 
        Query = "Resources | where type =~ 'Microsoft.Network/publicIPAddresses'" 
        Subscriptions = @("SUB_B", "SUB_C") # Query 2 Scope 
    } 
) 
$BatchResults = $BatchQueries | ForEach-Object -Parallel { 
    $QueryConfig = $_ 
    $Query = $QueryConfig.Query 
    $Subs = $QueryConfig.Subscriptions 
    
    Write-Host "[Batch Worker] Starting query: $($Query.Substring(0, [Math]::Min(50, $Query.Length)))..." -ForegroundColor Cyan 
    
    $QueryResults = @() 
    
    # Process each subscription in this query's scope 
    foreach ($SubId in $Subs) { 
        $SkipToken = $null 
        
        do { 
            $Params = @{ 
                Query = $Query 
                Subscription = $SubId 
                First = 1000 
            } 
            
            if ($SkipToken) { 
                $Params['SkipToken'] = $SkipToken 
            } 
            
            $Result = Search-AzGraph @Params 
            
            if ($Result) { 
                $QueryResults += $Result 
            } 
            
            $SkipToken = $Result.SkipToken 
            
        } while ($SkipToken) 
    } 

    # Return results with metadata 
    [PSCustomObject]@{ 
        Query = $Query 
        Subscriptions = $Subs 
        Data = $QueryResults 
        Count = $QueryResults.Count 
    } 
    
} -ThrottleLimit 5 

Write-Host "`nBatch complete. Reviewing results..." 

# The results are returned in the same order as the input array 
$VMCount = $BatchResults[0].Data.Count 
$IPCount = $BatchResults[1].Data.Count 

Write-Host "Query 1 (VMs) returned: $VMCount results." 
Write-Host "Query 2 (IPs) returned: $IPCount results." 

```