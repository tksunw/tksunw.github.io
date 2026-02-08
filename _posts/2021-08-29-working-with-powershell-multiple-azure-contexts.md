---
title: "Working with PowerShell & Multiple Azure Contexts"
date: 2021-08-29 12:00:00 -0500
categories: [Automation]
tags: [powershell, azure, azcontext, cloud]
---

When working with multiple Azure subscriptions, the PowerShell Az.* modules allow for easy context switching. This means that you can run commands against multiple subscriptions, or you can run commands against subscriptions without changing your default context.

An Azure Context object contains information about the Account that was used to sign into Azure, the active (for that context) Azure Subscription, and an auth token cache is not actually empty, it just can't read from here for security reasons, though you can read it with the `Get-AzAccessToken` command.

## Azure Context Object Structure

```powershell
PS> Get-AzContext | fl *
Name               : TK-PRD (yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy) - tim@timkennedy.net
Account            : tim@timkennedy.net
Environment        : AzureCloud
Subscription       : yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
Tenant             : zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz
TokenCache         :
VersionProfile     :
ExtendedProperties : {}
```

If you just wanted to change the context in your terminal, so that future commands all run against that new subscription, you could use `Set-AzContext` to change to any Subscription your account has access to.

```powershell
Set-AzContext -Subscription 'TK-PRD'
```

or

```powershell
Get-AzContext -SubscriptionName 'TK-PRD' | Set-AzContext
```

## Using -DefaultProfile Parameter Per Command

But, even better, you can set the context per command via the `-DefaultProfile` argument or its alias `-AzContext`. This works with all the Az.* PowerShell modules. For example:

```powershell
$Subscription = 'TK-DEV'
$AzContext = Get-AzContext -ListAvailable | ?{$_.Subscription.Name -eq "$Subscription"}
```

Now that `$AzContext` variable can be used with any Az PowerShell command or function, to force a command to be run within that context.

```powershell
Get-AzResourceGroup -DefaultProfile $AzContext
```

Now you can list all the Resource Groups in the TK-PRD subscription, even if your shell has the default context set to the TK-DEV subscription.

```powershell
PS> Get-AzContext

Name                                     Account          SubscriptionName    Environm
----                                     -------          ----------------    --------
TK-DEV (xxxxxxxx-xxxx-xxxx-xxxx-xxxx…    tim@tkdev…       TK-DEV              AzureClo


PS> Get-AzContext -DefaultProfile $AzContext

Name                                     Account          SubscriptionName    Environm
----                                     -------          ----------------    --------
Default                                  tim@tkdev…       TK-PRD              AzureClo


PS> Get-AzResourceGroup -DefaultProfile $AzContext -Location eastus

ResourceGroupName : cloud-shell-storage-eastus
Location          : eastus
ProvisioningState : Succeeded
Tags              :
ResourceId        : /subscriptions/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy/resourceGroups
```

## Foreach Loop Across Multiple Subscriptions

You can work this into a foreach loop if you want to do something in multiple subscriptions. If you want to find all the Recovery Services Vaults in all your subscriptions:

```powershell
PS>  foreach ($c in (Get-AzContext -ListAvailable)) {
>>     "searching $($c.Subscription.Name)"
>>     Get-AzRecoveryServicesVault -DefaultProfile $c
>> }
searching TK-DEV

Name              : rsvault-dev
ID                : /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups
                    /dev-rs-rg/providers/Microsoft.RecoveryServices/vaults/rsvault-dev
Type              : Microsoft.RecoveryServices/vaults
Location          : useast
ResourceGroupName : dev-rs-rg
SubscriptionId    : xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Properties        : Microsoft.Azure.Commands.RecoveryServices.ARSVaultProperties
Identity          :

searching TK-PRD

Name              : rsvault-prd
ID                : /subscriptions/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy/resourceGroups
                    /prd-rs-rg/providers/Microsoft.RecoveryServices/vaults/rsvault-prd
Type              : Microsoft.RecoveryServices/vaults
Location          : useast
ResourceGroupName : prd-rs-rg
SubscriptionId    : yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
Properties        : Microsoft.Azure.Commands.RecoveryServices.ARSVaultProperties
Identity          :
```

## Security Benefits

Another advantage of explicitly setting the context you want to work against in your scripts and functions is that you reduce the risk of accidentally running commands against the wrong Subscription when you forget to change your context.

I maintain my default context set to a Subscription used only for testing proofs of concept, or for following along on training or tutorials. Since everything in that subscription is considered ephemeral, this helps protect production subscriptions from accidental changes, because of the intent required to change the Azure Context to the intended subscription.

## Example Function Implementation

It is simple to build into scripts and functions:

```powershell
function Get-SubscriptionAsrVaults {
    [CmdletBinding()]
    Param(
        [Parameter(Mandatory=$true)]
        [String]$Subscription
    )

    $AzContext = Get-AzContext -ListAvailable | ?{$_.Subscription.Name -eq "$Subscription" }

    if(!$AzContext) {
        throw "No Azure Context found for Subscription $Subscription"
    }
    $subvaults = Get-AzRecoveryServicesVault -DefaultProfile $AzContext -Ea SilentlyContinue
    return $subvaults
}
```

Now you can iterate through all subscriptions to find Recovery Services Vaults.

## Team Collaboration Benefit

There's an advantage too when sharing scripts with other team members or folks on different teams. When writing scripts to automate tasks for the HelpDesk, requiring the subscription to be specified and forcing commands to run against the specified subscription increases the risk mitigation slightly, though it's not a complete protection against human error.
