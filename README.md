## A PowerShell script which shows Host Keys and Function Keys for all the functions within the function app.
```
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls -bor [Net.SecurityProtocolType]::Tls11 -bor [Net.SecurityProtocolType]::Tls12

$ResourceGroupName = '{Resource Group Name}'
$FunctionAppName = '{Function App Name}'
$FunctionsList = New-Object PSObject
$DefaultHostKeys = New-Object PSObject

# This would return Master Host Key
function Get-MasterHostKey($FunctionAppName){

    $apiUrl = "https://$FunctionAppName.scm.azurewebsites.net/api/functions/admin/masterkey"
    $result = Invoke-RestMethod -Uri $apiUrl -Headers @{Authorization=("Basic {0}" -f $base64AuthInfo)} -UserAgent $userAgent -Method GET
    return $result.masterKey

}

# This would return all the host keys (default + user added)
function Get-DefaultHostKey($FunctionAppName, $masterHostKey){    
 
    $apiUrl = "https://$FunctionAppName.azurewebsites.net/admin/host/keys?code=$masterHostKey"
    $result = Invoke-RestMethod -Uri $apiUrl -Headers @{Authorization=("Basic {0}" -f $base64AuthInfo)} -UserAgent $userAgent -Method GET

    Foreach($defaultKey in $result.keys){
    $DefaultHostKeys | Add-Member -MemberType NoteProperty -Name $defaultKey.name -Value $defaultKey.value -Force
    }
    return $DefaultHostKeys
}

# This would return all the function keys (default + user added) for all the functions of a function app
function Get-DefaultFunctionKeys($FunctionAppName){    
 
    $apiUrl = "https://$FunctionAppName.scm.azurewebsites.net/api/functions/admin/token"
    $token = Invoke-RestMethod -Uri $apiUrl -Headers @{Authorization=("Basic {0}" -f $base64AuthInfo)} -UserAgent $userAgent -Method GET

    $versionCheckUrl = "https://$FunctionAppName.azurewebsites.net/admin/host/status"
    $result = Invoke-RestMethod -Uri $versionCheckUrl -Headers @{Authorization=("Bearer {0}" -f $token)} -UserAgent $userAgent -Method GET
    $RuntimeVersion = $result.version
    Write-Host ""
    Write-Host "The function app is running on version :"$RuntimeVersion
    Write-Host ""

    $Functions = ""

    if($RuntimeVersion -ge 2){
    $Functions = Invoke-RestMethod -Method GET -Headers @{Authorization = ("Bearer {0}" -f $token)} -Uri https://$FunctionAppName.azurewebsites.net/admin/functions
    }
    else{
    $Functions = Invoke-RestMethod -Method GET -Headers @{Authorization = ("Basic {0}" -f $base64AuthInfo)} -Uri https://$FunctionAppName.scm.azurewebsites.net/api/functions
    }

    ForEach ($functionName in $Functions.Name) {
       $result = Invoke-RestMethod -Method GET -Headers @{Authorization = ("Bearer {0}" -f $token)} -Uri "https://$FunctionAppName.azurewebsites.net/admin/functions/$functionName/keys"
       $FunctionsList | Add-Member -MemberType NoteProperty -Name $functionName -Value $result.keys.value -Force
    }

    return $FunctionsList
}

$resource = Invoke-AzureRmResourceAction -ResourceGroupName $resourceGroupName -ResourceType Microsoft.Web/sites/config -ResourceName $FunctionAppName/publishingcredentials -Action list -ApiVersion 2018-02-01 -Force

$username = $resource.properties.publishingUserName
$password = $resource.properties.publishingPassword

$base64AuthInfo = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(("{0}:{1}" -f $username, $password)))
$userAgent = "powershell/1.0"

$masterHostKey = Get-MasterHostKey $FunctionAppName
$defaultHostKey = Get-DefaultHostKey $FunctionAppName $masterHostKey
$defaultFunctionKey = Get-DefaultFunctionKeys $FunctionAppName

Write-Host "------------------------------- Showing Master Host Key ----------------------------- "
Write-Host ""
Write-Host "Master Host Key: " $masterHostKey
Write-Host ""
Write-Host "------------------------------- Showing Default Host Keys ----------------------------- "
$defaultHostKey | Format-List
Write-Host "------------------------------- Showing Function Keys ----------------------------- "
$defaultFunctionKey | Format-List 

```
