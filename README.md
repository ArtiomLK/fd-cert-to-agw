# [Creating a local PFX copy of App Service Certificate][1]

Using App Service Certificate in an Application Gateway Inbound https listener

## Introduction

Last year, we introduced ‘App Service Certificate’, a certificate lifecycle management offering. Azure portal provides a user-friendly experience for creating App Service Certificates and using them with App Service Apps. You can read more about this service [here][2]. While the portal provides first class experience for deploying App Service Certificate through Key Vault to App Service Apps, we have been receiving customer requests where they would like to use these certificates outside of App Service platform, say with Azure Virtual Machines. In this blogpost, I am going to describe how to create a local PFX copy of App Service Certificate so that you can use it anywhere you want.

## Prerequisites

Before making a local copy, make sure that you already have:

1. An App Certificate with either wildcard *.domain.com or subdomain.domain.com
2. A KeyVault where the App Certificate will be stored
3. Verified the domain ownership (domain.com) If a @ CNAME already exists, it won't allow it
4. The App Service Certificate is in ‘Issued’ state
5. It’s assigned to a Key Vault (Step 1 in the link shared above).

## Creating a local copy of the issued SSL certificate using PowerShell

You can use the following PowerShell script to create a local PFX copy. You need to first install Azure PowerShell and have the required modules installed. Follow this to [install the Azure PowerShell commandlets][3]. To use the script, open a PowerShell window, copy the entire script below (uncomment Remove-AzKeyVaultAccessPolicy line if you want to remove Access policies after export, check your permissions to see if you want to remove the policy) and paste it on the PowerShell window and hit enter.

```PowerShell
Function Export-AppServiceCertificateN
{
###########################################################

Param(
[Parameter(Mandatory=$true,Position=1,HelpMessage="ARM Login Url. E.G. artiomlk@contoso.com")]
[string]$loginId,

[Parameter(Mandatory=$true,HelpMessage="Subscription Id. In the format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")]
[string]$subscriptionId,

[Parameter(Mandatory=$true,HelpMessage="Resource Group Name. E.G. rg-app-where-app-certificate")]
[string]$resourceGroupName,

[Parameter(Mandatory=$true,HelpMessage="Name of the App Service Certificate Resource. E.G. my-app-certificate")]
[string]$name
)

###########################################################

Connect-AzAccount
Set-AzContext -Subscription $subscriptionId

## Get the KeyVault Resource Url and KeyVault Secret Name were the certificate is stored
$ascResource= Get-AzResource -ResourceId "/subscriptions/$subscriptionId/resourceGroups/$resourceGroupName/providers/Microsoft.CertificateRegistration/certificateOrders/$name"
$certProps = Get-Member -InputObject $ascResource.Properties.certificates[0] -MemberType NoteProperty
$certificateName = $certProps[0].Name
$keyVaultId = $ascResource.Properties.certificates[0].$certificateName.KeyVaultId
$keyVaultSecretName = $ascResource.Properties.certificates[0].$certificateName.KeyVaultSecretName

## Split the resource URL of KeyVault and get KeyVaultName and KeyVaultResourceGroupName
$keyVaultIdParts = $keyVaultId.Split("/")
$keyVaultName = $keyVaultIdParts[$keyVaultIdParts.Length - 1]
$keyVaultResourceGroupName = $keyVaultIdParts[$keyVaultIdParts.Length - 5]

## --- !! NOTE !! ----
## Only users who can set the access policy and has the the right RBAC permissions can set the access policy on KeyVault, if the command fails contact the owner of the KeyVault
Set-AzKeyVaultAccessPolicy -ResourceGroupName $keyVaultResourceGroupName -VaultName $keyVaultName -UserPrincipalName $loginId -PermissionsToSecrets get
Write-Host "Get Secret Access to account $loginId has been granted from the KeyVault, please check and remove the policy after exporting the certificate"

## Getting the secret from the KeyVault
$secret = Get-AzKeyVaultSecret -VaultName $keyVaultName -Name $keyVaultSecretName
$pfxCertObject= New-Object System.Security.Cryptography.X509Certificates.X509Certificate2 -ArgumentList @([Convert]::FromBase64String($secret.SecretValueText),"",[System.Security.Cryptography.X509Certificates.X509KeyStorageFlags]::Exportable)
$pfxPassword = -join ((65..90) + (97..122) + (48..57) | Get-Random -Count 50 | % {[char]$_})
$currentDirectory = (Get-Location -PSProvider FileSystem).ProviderPath
[Environment]::CurrentDirectory = (Get-Location -PSProvider FileSystem).ProviderPath
[io.file]::WriteAllBytes(".\appservicecertificate.pfx",$pfxCertObject.Export([System.Security.Cryptography.X509Certificates.X509ContentType]::Pkcs12,$pfxPassword))

## --- !! NOTE !! ----
## Remove the Access Policy required for exporting the certificate once you have exported the certificate to prevent giving the account prolonged access to the KeyVault
## The account will be completely removed from KeyVault access policy and will prevent to account from accessing any keys/secrets/certificates on the KeyVault,
## Run the following command if you are sure that the account is not used for any other access on the KeyVault or login to the portal and change the access policy accordingly.
# Remove-AzKeyVaultAccessPolicy -ResourceGroupName $keyVaultResourceGroupName -VaultName $keyVaultName -UserPrincipalName $loginId
# Write-Host "Access to account $loginId has been removed from the KeyVault"

# Print the password for the exported certificate
Write-Host "Created an App Service Certificate copy at: $currentDirectory\appservicecertificate.pfx"
Write-Warning "For security reasons, do not store the PFX password. Use it directly from the console as required."
Write-Host "PFX password: $pfxPassword"

}
```

[1]: https://azure.github.io/AppService/2017/02/24/Creating-a-local-PFX-copy-of-App-Service-Certificate.html
[2]: https://docs.microsoft.com/en-us/azure/app-service/configure-ssl-certificate?tabs=apex%2Cportal
[3]: https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-7.3.2
