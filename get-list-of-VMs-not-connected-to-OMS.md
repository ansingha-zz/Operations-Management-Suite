### Getting status of Classic VMs from multiple subscriptions - are VMs Running, Stopped, Connected to OMS?
If you are managing multiple Azure subscription and need to keep track of VM's status, the following PowerShell script can do it:

``` 
Add-AzureAccount
#path to the csv file that has all Subscriptions' information like SubscriptionName, SubscriptionId
$path = ''

#path where the report will get generated
$reportLoc = ''

$Subs =  import-csv -Path $path
foreach($Sub in $Subs)
{
    $subName = $Sub.SubscriptionName
    Select-AzureSubscription -SubscriptionName $subName
    $NonRunningVMs = Get-AzureVM | ? Status -Like '*Stopped*'
    $RunningVMs = Get-AzureVM | ? Status -Like '*Ready*'

    $VMs = Get-AzureVM 
    $i=0
    $VMs | ForEach-Object{
        if($_.ResourceExtensionStatusList.HandlerName -notcontains 'Microsoft.EnterpriseCloud.Monitoring.MicrosoftMonitoringAgent')
        {$i = $i + 1} 
        }
    $subName | Select-Object @{Name=’SubscriptionName’;Expression={$subName}},@{Name=’Total VMs’;Expression={$VMs.count}},@{Name=’Total VMs Running’;Expression={$RunningVMs.count}},@{Name=’Total VMs Not Running’;Expression={$NonRunningVMs.count}},@{Name='Num of VMs not connected to OMS’;Expression={$i}} | Export-Csv $reportLoc -Append -NoTypeInformation
}
```
