 ### Steps to be followed to create a new OMS workspace, install required solutions and install 'Microsoft Monitoring Agent' virtual machine extension to all VMs in the Azure Subscription:

```
#please provide the subscription name
$Subscription = ""

#Name of the new OMS Workspace
$WorkspaceName = ""

#ResourceGroup Name
$ResourceGroupName = ""

#specify Location
#only locations allowed are: East US; West Europe; Southeast Asia; Australia Southeast
$Loc = ""

#Location for Automation account
#If Loc is "East US", change Loc2 to "East US2". Otherwise, Loc and Loc2 can be same
$Loc2 = ""

Login-AzureRmAccount
Select-AzureRmSubscription -SubscriptionName $Subscription

#create a new Resource Group
$RG = New-AzureRmResourceGroup -Name $ResourceGroupName -Location $Loc

#create OMS Workspace 
New-AzureRmOperationalInsightsWorkspace -Location $Loc -Name $WorkspaceName -Sku free -ResourceGroupName $ResourceGroupName

#create Automation Account 
New-AzureRmAutomationAccount -Name "OMSAutomationAccount1" -Location $Loc2 -ResourceGroupName $ResourceGroupName

#to get a list of all possible solutions
Get-AzureRmOperationalInsightsIntelligencePacks -ResourceGroupName $ResourceGroupName -WorkspaceName $WorkspaceName | Out-File c:\solutions

　
#add a solution to the workspace. Other solutions can be added similarly
Set-AzureRmOperationalInsightsIntelligencePack -ResourceGroupName $ResourceGroupName -WorkspaceName $WorkspaceName -IntelligencePackName ADAssessment -Enabled $true

$workspace1 = Get-AzureRmOperationalInsightsWorkspace -ResourceGroupName $ResourceGroupName -Name $WorkspaceName

$keys = Get-AzureRmOperationalInsightsWorkspaceSharedKeys -ResourceGroupName $ResourceGroupName -Name $WorkspaceName

　
#steps to connect Classic VMs  to  this Workspace
Add-AzureAccount

Select-AzureSubscription -SubscriptionName $subscription

$Allvm = Get-AzureVM

$workspaceId = $workspace1.CustomerId
$workspaceKey = $keys.PrimarySharedKey

　
ForEach($vm in $Allvm)
{

Set-AzureVMExtension -VM $vm -Publisher 'Microsoft.EnterpriseCloud.Monitoring' -ExtensionName 'MicrosoftMonitoringAgent' -Version '1.*' -PublicConfiguration "{'workspaceId': '$workspaceId'}" -PrivateConfiguration "{'workspaceKey': '$workspaceKey' }" | Update-AzureVM -Verbose
}
``` 

To connect ARM VMs, use the following command:
```
Set-AzureRmVMExtension -ResourceGroupName $ResourceGroupName -VMName '' -Name 'MicrosoftMonitoringAgent' -Publisher 'Microsoft.EnterpriseCloud.Monitoring' -ExtensionType 'MicrosoftMonitoringAgent' -TypeHandlerVersion '1.0' -Location $location -SettingString "{'workspaceId': '$workspaceId'}" -ProtectedSettingString "{'workspaceKey': '$workspaceKey'}"
```

Also refer: https://docs.microsoft.com/en-us/azure/log-analytics/log-analytics-azure-vm-extension
 
