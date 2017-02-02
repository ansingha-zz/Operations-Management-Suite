 
### Consolidating report from multiple Azure subscriptions/OMS Workspaces in a single Excel file

We can integrate OMS with PowerBI to generate reports from a single subscription. But if we have multiple subscriptions and need to pull a consolidated report or if we just want reports out in a Excel file - the following script can do it.

The concept I used is:
Writing a query and saving a new search based on this query. Then pulling the logs from this saved search in an Excel file. To get logs from multiple subscriptions, I am using the -append switch. 

```
Login-AzureRmAccount
#path to the csv file that has all Subscriptions' information like SubscriptionName, SubscriptionId
$InputFilepath = ''

#path where the report excel file will get generated in .csv format
$ReportPath = ''

$Subs =  import-csv -Path $InputFilepath
foreach ($sub in $subs)
{
$subName = $Sub.SubscriptionName

Select-AzureRmSubscription -SubscriptionName $subName
$subscriptionId = (Get-AzureRmSubscription -SubscriptionName $subName).SubscriptionId

$WorkspaceName = (Get-AzureRmOperationalInsightsWorkspace).Name
$ResourceGroupName = (Get-AzureRmOperationalInsightsWorkspace).ResourceGroupName

#saving the search
$Query = @"
Type=Update OSType!=Linux UpdateState=Needed Optional=false (Classification=\"Security Updates\" OR Classification=\"Critical Updates\") | select Computer,Title,KBID,Classification,UpdateSeverity,PublishedDate,RebootBehavior
"@
New-AzureRmOperationalInsightSavedSearch -ResourceGroupName $ResourceGroupName -WorkspaceName $WorkspaceName -SavedSearchId "UpdatesMissing-Id" -DisplayName "Updates missing from VMs across subscriptions" -Category "Reports" -Query $Query -Version "1" -Force

#getting results from saved search
$results = Get-AzureRmOperationalInsightsSavedSearchResults -ResourceGroupName $ResourceGroupName -WorkspaceName $WorkspaceName -SavedSearchId "UpdatesMissing-Id"

$results.Value | ConvertFrom-Json | Select-Object *,@{Name='Subscription';Expression={$subName}} | Export-Csv $ReportPath -NoTypeInformation -Append
}
``` 


Or if we dont have a saved search, we can directly get the results in Excel from a dynamic query as follows: 
```
$dynamicQuery = "" #write the dynamic query
$result = Get-AzureRmOperationalInsightsSearchResults -ResourceGroupName $ResourceGroupName -WorkspaceName $WorkSpaceName -Query $dynamicQuery

$result.Value | ConvertFrom-Json | Select-Object *,@{Name='Subscription';Expression={$subName}} | Export-Csv $ReportPath -NoTypeInformation
```
