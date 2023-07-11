# Azure Connected Machine Agent uninstallation

## Steps to uninstall

1. Remove VM extensions
2. Disconnect the server from Azure Arc
3. Uninstall the Windows agent

## Pre-requisites

- Install [Azure PowerShell](https://learn.microsoft.com/powershell/azure/install-azps-windows?view=azps-10.1.0&tabs=powershell&pivots=windows-psgallery).
- Create a [Service Principal](https://learn.microsoft.com/azure/azure-arc/servers/onboard-service-principal#create-a-service-principal-for-onboarding-at-scale) with at least _[Azure Connected Machine Resource Administrator](https://learn.microsoft.com/azure/role-based-access-control/.built-in-roles#azure-connected-machine-resource-administrator)_ role. Download its Id and secret to be used later.

>NOTE
> Set a short expiry date for the Service Principal, and make sure to delete it after completing the agent uninstallation on all servers.

- Run _[Connect-AzAccount](https://learn.microsoft.com/powershell/module/az.accounts/connect-azaccount?view=azps-10.1.0)_ command to connect to Azure using Azure Powershell.
- Set the [default context](https://learn.microsoft.com/powershell/module/az.accounts/set-azcontext?view=azps-10.1.0) in Azure PowerShell.

```powershell
Set-AzContext -Subscription "xxxx-xxxx-xxxx-xxxx"
```

### Remove VM extensions

>NOTE
> This script should run on a management machine or in Azure Cloud Shell

#### Resource group scope

```powershell
$resourceGroupName= "<Name of the Resource Group where the Arc-enabled Servers are located>"
Get-AzConnectedMachine -ResourceGroupName $ResourceGroupName | ForEach-Object{
    $serverName = $_.Name
    Get-AzConnectedMachineExtension -ResourceGroupName $resourceGroupName -MachineName $serverName | foreach-object{
        $extensionName = $_.Name
        Remove-AzConnectedMachineExtension -ResourceGroupName $resourceGroupName -MachineName $serverName -Name $extensionName -NoWait -Confirm:$false
    }
}
```

#### Subscription scope

```powershell
$subscriptionId= "<Id of the Subscription where the Arc-enabled Servers are located>"
Get-AzConnectedMachine -SubscriptionId $subscriptionId | ForEach-Object{
    $serverName = $_.Name
    Get-AzConnectedMachineExtension -ResourceGroupName $resourceGroupName -MachineName $serverName | foreach-object{
        $extensionName = $_.Name
        Remove-AzConnectedMachineExtension -ResourceGroupName $resourceGroupName -MachineName $serverName -Name $extensionName -NoWait -Confirm:$false
    }
}
```

### Disconnect the server from Azure Arc and uninstall agent

>NOTE
> This script should run on the Arc-enabled Servers to be disconnected

#### Windows

```powershell
# Set the Service Principal Id and Secret variables
$ServicePrincipalId = <The Servic Principal Id>
$ServicePrincipalClientSecret = <The Servic Principal secret>

# Set location to the Azure Connected Machine Agent folder
Set-Location "C:\Program Files\AzureConnectedMachineAgent"

# Disconnecting the Azure Connected Machine Agent
$agentStatus = .\azcmagent.exe show --json | ConvertFrom-Json
if($agentStatus.status -ne "Connected"){
    .\azcmagent.exe disconnect --force-local-only
}
else{
    .\azcmagent disconnect --service-principal-id $ServicePrincipalId --service-principal-secret $ServicePrincipalClientSecret
}

# Uninstalling the Azure Connected Machine Agent msi installer
Get-ChildItem -Path HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall | `
Get-ItemProperty |Where-Object {$_.DisplayName -eq "Azure Connected Machine Agent"} | ForEach-Object {MsiExec.exe /x "$($_.PsChildName)" /qn}
```

#### Linux

#### Ubuntu

```bash
#!/bin/bash

# Set the Service Principal Id and Secret variables
ServicePrincipalId=<The Servic Principal Id>
ServicePrincipalClientSecret=<The Servic Principal secret>

# Elevate to sudo privileges
sudo -s

# Disconnecting the Azure Connected Machine Agent
agentStatus=$(azcmagent show | grep "Agent Status" | cut -d ":" -f2 | awk '{$1=$1;print}')
if [ "$agentStatus" != "Connected" ]; then
    azcmagent disconnect --force-local-only
else
    azcmagent disconnect --service-principal-id $ServicePrincipalId --service-principal-secret $ServicePrincipalClientSecret
fi

# Uninstalling the Azure Connected Machine Agent
sudo apt purge azcmagent
```

#### RHEL, CentOS, Oracle Linux, and Amazon Linux

```bash
#!/bin/bash

# Set the Service Principal Id and Secret variables
ServicePrincipalId=<The Servic Principal Id>
ServicePrincipalClientSecret=<The Servic Principal secret>

# Elevate to sudo privileges
sudo -s

# Disconnecting the Azure Connected Machine Agent
agentStatus=$(azcmagent show | grep "Agent Status" | cut -d ":" -f2 | awk '{$1=$1;print}')
if [ "$agentStatus" != "Connected" ]; then
    azcmagent disconnect --force-local-only
else
    azcmagent disconnect --service-principal-id $ServicePrincipalId --service-principal-secret $ServicePrincipalClientSecret
fi

# Uninstalling the Azure Connected Machine Agent
sudo yum remove azcmagent
```

#### SLES

```bash
#!/bin/bash

# Set the Service Principal Id and Secret variables
ServicePrincipalId=<The Servic Principal Id>
ServicePrincipalClientSecret=<The Servic Principal secret>

# Elevate to sudo privileges
sudo -s

# Disconnecting the Azure Connected Machine Agent
agentStatus=$(azcmagent show | grep "Agent Status" | cut -d ":" -f2 | awk '{$1=$1;print}')
if [ "$agentStatus" != "Connected" ]; then
    azcmagent disconnect --force-local-only
else
    azcmagent disconnect --service-principal-id $ServicePrincipalId --service-principal-secret $ServicePrincipalClientSecret
fi

# Uninstalling the Azure Connected Machine Agent
sudo zypper remove azcmagent
```
