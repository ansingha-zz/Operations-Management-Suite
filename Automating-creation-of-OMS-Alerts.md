### Managing multiple Azure subscriptions\Operations Management Suite Workspaces and need to automate the configuration of OMS alerts

#### The Alerts that I am configuring in this example:

- Alert 1: any admin signs-into the VM and resets the password
- Alert 2: any admin resets the password from the Azure portal
- Alert 3: Free Disk Space is less than 20%
- Alert 4: CPU utilization is more othan 80% for a period of 30 minutes
- Alert 5: RAM used is more than 80% for a period of 30 minutes

　
```
#login to ARMClient
ARMClient login

#Login to Azure
Login-AzureRmAccount

#path to the csv file that has all Subscriptions' information like SubscriptionName, SubscriptionId
$path = ''

$Subs =  import-csv -Path $path
foreach ($sub in $subs)
{
$subName = $Sub.SubscriptionName
Select-AzureRmSubscription -SubscriptionName $subName

$subscriptionId = (Get-AzureRmSubscription -SubscriptionName $subName).SubscriptionId 

$Workspace = (Get-AzureRmOperationalInsightsWorkspace).Name
$ResourceGroup = (Get-AzureRmOperationalInsightsWorkspace).ResourceGroupName

#enabling required performance metrics
New-AzureRmOperationalInsightsWindowsPerformanceCounterDataSource -ResourceGroupName $ResourceGroup -WorkspaceName $Workspace -ObjectName "LogicalDisk" -InstanceName "*" -CounterName "% Free Space" -IntervalSeconds 30 -Name "Free Disk Space"
New-AzureRmOperationalInsightsWindowsPerformanceCounterDataSource -ResourceGroupName $ResourceGroup -WorkspaceName $Workspace -ObjectName "Processor" -InstanceName "_Total" -CounterName "% Processor Time" -IntervalSeconds 30 -Name "CPU Utilization"
New-AzureRmOperationalInsightsWindowsPerformanceCounterDataSource -ResourceGroupName $ResourceGroup -WorkspaceName $Workspace -ObjectName "Memory" -InstanceName "*" -CounterName "% Committed Bytes In Use" -IntervalSeconds 30 -Name "Memory In Use"

　
#1

$search="PasswordResetFromVM"
$searchInfo="Password was reset within the VM"
$QueryJson = $search + "Json"
　
$QueryJson = @"
{'properties': { 'Category': 'Alert', 'DisplayName':'$searchInfo', 'Query':'Type=SecurityEvent (EventID=4724 OR EventID=4723) EventSourceName=Microsoft-Windows-Security-Auditing AccountType!=Machine', 'Version':'1'}
"@ 
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id?api-version=2015-03-20 $QueryJson
　
$scheduleJson = "{'properties': { 'Interval': 5, 'QueryTimeSpan':5, 'Active':'true' }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule?api-version=2015-03-20 $scheduleJson
　
$emailJson = "{'properties': { 'Name': '$searchInfo', 'Version':'1', 'Severity':'Warning', 'Type':'Alert', 'Threshold': { 'Operator': 'gt', 'Value': 0 }, 'EmailNotification': {'Recipients': ['recepient@email.com'], 'Subject':'$searchInfo', 'Attachment':'None'} }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule/actions/$Search-action?api-version=2015-03-20 $emailJson

　
#2
 
$search="PasswordResetFromPortal"
$searchInfo="Password was reset from Azure Portal"
$QueryJson = $search + "Json"

$QueryJson = @"
{'properties': { 'Category': 'Alert', 'DisplayName':'$searchInfo', 'Query':'Type=AzureActivity VMAccessWindowsPasswordReset* ActivityStatus=Started', 'Version':'1'}
"@ 
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id?api-version=2015-03-20 $QueryJson

$scheduleJson = "{'properties': { 'Interval': 5, 'QueryTimeSpan':5, 'Active':'true' }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule?api-version=2015-03-20 $scheduleJson
　
$emailJson = "{'properties': { 'Name': '$searchInfo', 'Version':'1', 'Severity':'Warning', 'Type':'Alert', 'Threshold': { 'Operator': 'gt', 'Value': 0 }, 'EmailNotification': {'Recipients': ['recepient@email.com'], 'Subject':'$searchInfo', 'Attachment':'None'} }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule/actions/$Search-action?api-version=2015-03-20 $emailJson

　
#3

$search="FreeDiskSpace"
$searchInfo="Free Disk Space is less than 20%"
$QueryJson = $search + "Json"
　
$QueryJson = @"
{'properties': { 'Category': 'Alert', 'DisplayName':'$searchInfo', 'Query':'Type=Perf ObjectName=LogicalDisk CounterName=\"% Free Space\" InstanceName="_Total" CounterValue<20', 'Version':'1'}
"@ 
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id?api-version=2015-03-20 $QueryJson

$scheduleJson = "{'properties': { 'Interval': 5, 'QueryTimeSpan':5, 'Active':'true' }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule?api-version=2015-03-20 $scheduleJson

$emailJson = "{'properties': { 'Name': '$searchInfo', 'Version':'1', 'Severity':'Warning', 'Type':'Alert', 'Threshold': { 'Operator': 'gt', 'Value': 0 }, 'EmailNotification': {'Recipients': ['recepient@email.com'], 'Subject':'$searchInfo', 'Attachment':'None'} }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule/actions/$Search-action?api-version=2015-03-20 $emailJson

　
#4

$search="HighCPUUtilization"
$searchInfo="CPU utilization is high"
$QueryJson = $search + "Json"
　
$QueryJson = @"
{'properties': { 'Category': 'Alert', 'DisplayName':'$searchInfo', 'Query':'Type=Perf (ObjectName=Processor) CounterName=\"% Processor Time\" InstanceName="_Total" | Measure avg(CounterValue) by Computer Interval 30Minutes | where AggregatedValue>80', 'Version':'1'}
"@ 
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id?api-version=2015-03-20 $QueryJson
　
$scheduleJson = "{'properties': { 'Interval': 60, 'QueryTimeSpan':60, 'Active':'true' }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule?api-version=2015-03-20 $scheduleJson
　
$emailJson = "{'properties': { 'Name': '$searchInfo', 'Version':'1', 'Severity':'Warning', 'Type':'Alert', 'Threshold': { 'Operator': 'gt', 'Value': 0 }, 'EmailNotification': {'Recipients': ['recepient@email.com'], 'Subject':'$searchInfo', 'Attachment':'None'} }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule/actions/$Search-action?api-version=2015-03-20 $emailJson

　
#5

$search="MemoryInUse"
$searchInfo="Committed Bytes in use is high"
$QueryJson = $search + "Json"
　
$QueryJson = @"
{'properties': { 'Category': 'Alert', 'DisplayName':'$searchInfo', 'Query':'Type=Perf (ObjectName=Memory) CounterName=\"% Committed Bytes in Use\" InstanceName="*" | Measure avg(CounterValue) by Computer Interval 30Minutes | where AggregatedValue>80', 'Version':'1'}
"@ 
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id?api-version=2015-03-20 $QueryJson
　
$scheduleJson = "{'properties': { 'Interval': 60, 'QueryTimeSpan':60, 'Active':'true' }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule?api-version=2015-03-20 $scheduleJson
　
$emailJson = "{'properties': { 'Name': '$searchInfo', 'Version':'1', 'Severity':'Warning', 'Type':'Alert', 'Threshold': { 'Operator': 'gt', 'Value': 0 }, 'EmailNotification': {'Recipients': ['recepient@email.com'], 'Subject':'$searchInfo', 'Attachment':'None'} }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule/actions/$Search-action?api-version=2015-03-20 $emailJson
}
```

#### Another example to configure alerts

```
$searchValues=@("PasswordResetFromVM", "PasswordResetFromPortal")
$searchInfoStrings= @("Password was reset within the VM", "Password was reset from Azure Portal")
$queryInfo =  @("Type=SecurityEvent (EventID=4724 OR EventID=4723) EventSourceName=Microsoft-Windows-Security-Auditing AccountType!=Machine", "Type=AzureActivity VMAccessWindowsPasswordReset* ActivityStatus=Started")

for( $i = 0; $i -lt $searchValues.length; $i++) 
{
$search = $searchValues[$i]
$searchInfo = $searchInfoStrings[$i]
$query = $queryInfo[$i]
  
$QueryJson = $search + "Json"

$QueryJson = @"
{'properties': { 'Category': 'Security', 'DisplayName':'$searchInfo', 'Query':'$query', 'Version':'1'  }
"@
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id?api-version=2015-03-20 $QueryJson

$scheduleJson = "{'properties': { 'Interval': 5, 'QueryTimeSpan':5, 'Active':'true' }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule?api-version=2015-03-20 $scheduleJson

$emailJson = "{'properties': { 'Name': '$searchInfo', 'Version':'1', 'Severity':'Warning', 'Type':'Alert', 'Threshold': { 'Operator': 'gt', 'Value': 0 }, 'EmailNotification': {'Recipients': ['recepient@email.com'], 'Subject':'$searchInfo', 'Attachment':'None'} }"
armclient put /subscriptions/$subscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.OperationalInsights/workspaces/$workspace/savedSearches/$Search-Id/schedules/$Search-schedule/actions/$Search-action?api-version=2015-03-20 $emailJson
}
```
For more information, please refer:
https://docs.microsoft.com/en-us/azure/log-analytics/log-analytics-api-alerts

https://docs.microsoft.com/en-us/azure/log-analytics/log-analytics-alerts

https://docs.microsoft.com/en-us/azure/log-analytics/log-analytics-log-search-api 
