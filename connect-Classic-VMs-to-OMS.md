###Connecting all Classic VMs with OMS (that are not already connected) across multiple subscriptions
If you are managing multiple Azure subscriptions and thus managing a lot of Azure VMs and want to connect all VMs to OMS (the VMs that are not already connected), this script will do that. Can be used for ARM VMs too with minor changes:

```
Add-AzureAccount
#path to the csv file that has all Subscriptions' information like SubscriptionName, SubscriptionId
$path = ''

$Subs =  import-csv -Path $path
foreach ($sub in $subs)
{
    $subName = $Sub.SubscriptionName
    Select-AzureSubscription -SubscriptionName $subName

    #select OMS workspace within the selected Azure subscription
    $OMSWorkspace = Get-AzureRmOperationalInsightsWorkspace
    $keys = Get-AzureRmOperationalInsightsWorkspaceSharedKeys -ResourceGroupName $OMSWorkspace.ResourceGroupName -Name $OMSWorkspace.Name

    $workspaceId = $OMSWorkspace.CustomerId
    $workspaceKey = $keys.PrimarySharedKey

    $VMs = Get-AzureVM
 
    $VMs | ForEach-Object{
        if($_.ResourceExtensionStatusList.HandlerName -notcontains 'Microsoft.EnterpriseCloud.Monitoring.MicrosoftMonitoringAgent')
        {
            Set-AzureVMExtension -VM $_ -Publisher 'Microsoft.EnterpriseCloud.Monitoring' -ExtensionName 'MicrosoftMonitoringAgent' -Version '1.*' -PublicConfiguration "{'workspaceId': $workspaceId}" -PrivateConfiguration "{'workspaceKey': $workspaceKey}" | Update-AzureVM
        }
    }  
}
```

To get the status of all VMs - are they connected to OMS or not, refer this:

[Getting status of Classic VMs from multiple subscriptions - are VMs Running, Stopped, Connected to OMS?](https://github.com/ansingha/Operations-Management-Suite/blob/master/get-list-of-VMs-not-connected-to-OMS.md)
