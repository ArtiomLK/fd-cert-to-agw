# [Creating a local PFX copy of App Service Certificate][1]

Using App Service Certificate in an Application Gateway Inbound https listener

## Introduction

Last year, we introduced ‘App Service Certificate’, a certificate lifecycle management offering. Azure portal provides a user-friendly experience for creating App Service Certificates and using them with App Service Apps. You can read more about this service [here][2]. While the portal provides first class experience for deploying App Service Certificate through Key Vault to App Service Apps, we have been receiving customer requests where they would like to use these certificates outside of App Service platform, say with Azure Virtual Machines. In this blogpost, I am going to describe how to create a local PFX copy of App Service Certificate so that you can use it anywhere you want.

## Prerequisites

Before making a local copy, make sure that you already have:

1. An App Certificate with either wildcard *.domain.com or subdomain.domain.com. [Create an App Service Certificate][4]
2. A KeyVault where the App Certificate will be stored
3. Verified the domain ownership (domain.com) If a @ CNAME already exists, it won't allow it
4. The App Service Certificate is in ‘Issued’ state
5. It’s assigned to a Key Vault (Step 1 in the link shared above).

## Creating a local copy of the issued SSL certificate using PowerShell

You can use the following PowerShell script to create a local PFX copy. You need to first install Azure PowerShell and have the required modules installed. Follow this to [install the Azure PowerShell commandlets][3]. To use the script, open a PowerShell window, copy the entire script below (uncomment Remove-AzKeyVaultAccessPolicy line if you want to remove Access policies after export, check your permissions to see if you want to remove the policy) and paste it on the PowerShell window and hit enter.

```PowerShell
Function Export-AppServiceCertificate
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

# HardCode Params *4Debug
# $loginId='artiomlk@contoso.com'
# $subscriptionId='xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
# $resourceGroupName='rg-app-where-app-certificate'
# $name='my-app-certificate'

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

# Print Variables *4Debug
Write-Host "ascResource: $ascResource `n"
Write-Host "certProps: $certProps `n"
Write-Host "keyVaultIdParts: $keyVaultIdParts `n"

Write-Host "certificateName: $certificateName `n"
Write-Host "keyVaultId: $keyVaultId `n"
Write-Host "keyVaultSecretName: $keyVaultSecretName `n"
Write-Host "keyVaultResourceGroupName: $keyVaultResourceGroupName `n"
Write-Host "keyVaultName: $keyVaultName `n"

## --- !! NOTE !! ----
## Only users who can set the access policy and has the the right RBAC permissions can set the access policy on KeyVault, if the command fails contact the owner of the KeyVault
Set-AzKeyVaultAccessPolicy -ResourceGroupName $keyVaultResourceGroupName -VaultName $keyVaultName -UserPrincipalName $loginId -PermissionsToSecrets get
Write-Host "Get Secret Access to account $loginId has been granted from the KeyVault, please check and remove the policy after exporting the certificate"

# Allows to display and get secrets *4Debug
Set-AzKeyVaultAccessPolicy -ResourceGroupName $keyVaultResourceGroupName -VaultName $keyVaultName -UserPrincipalName $loginId -PermissionsToSecrets get,list

## Getting the secret from the KeyVault
$secret = Get-AzKeyVaultSecret -VaultName $keyVaultName -Name $keyVaultSecretName -AsPlainText
$pfxCertObject= New-Object System.Security.Cryptography.X509Certificates.X509Certificate2 -ArgumentList @([Convert]::FromBase64String($secret),"",[System.Security.Cryptography.X509Certificates.X509KeyStorageFlags]::Exportable)
$pfxPassword = -join ((65..90) + (97..122) + (48..57) | Get-Random -Count 50 | % {[char]$_})
$currentDirectory = (Get-Location -PSProvider FileSystem).ProviderPath

# Print Variables *4Debug
Write-Host secret: $secret`n
Write-Host pfxCertObject: $pfxCertObject`n
Write-Host pfxPassword: $pfxPassword`n
Write-Host currentDirectory: $currentDirectory`n

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

Now you will have a new command called Export-AppServiceCertificate, use the command as follows:

```PowerShell
# Replace the values withing the ""
Export-AppServiceCertificate -loginId "artiomlk@contoso.com" -subscriptionId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" -resourceGroupName "rg-app-where-app-certificate" -name "my-app-certificate"
```

Once the command is executed, you would see a new file in the current directory called ‘appservicecertificate.pfx’. This is a password protected PFX, the PowerShell console would display the corresponding password. For security reasons, do not store this password in a text file. You can use the password directly from the console as required. Also, don’t forget to delete the local PFX file once you no longer need it.

## Exporting the certificate with the chain included for App Service Web App consumption

The pfx created by the above commands will not include certificates from the chain. Services like Azure App Services expect the certificates that are being uploaded to have all the certificates in the chain included as part of the pfx file. To get the certificates of the chain to be part of the pfx, you will need to install the exported certificate on your machine first using the password that is provided by the script, make sure you mark the certificate as exportable.

## 1. [Import the Certificate Locally][5]

## 2. [Install Intermediate and Root Certificates][6]


Go to <https://certs.godaddy.com/repository> and download the intermediate certificates G2 and the root G2 certificate. For instance, `gdroot-g2.crt` & `gdig2.crt (DER)`. Install all of the certificates downloaded to the same store as your certificate. Once you confirmed that all the certificates in the chain have been installed we can export the certificate. For instance:

## 3. Export Certificate

1. Open Manage User Certificates and
2. Find your installed Azure App Certificate from previous steps
3. Right Click -> All Tasks -> [Export][7]
   1. [Follow this guide to export the Certificate][7]
   2. Validate the exported certificate by running the command:
      1. `certutil -dump <path of the certificate file>`
         1. `certutil -dump agwappservicecertificate.pfx`
      2. You will see the list of the certificates that are part of the pfx from the root to your certificate. A pfx file created with the above steps with all the certificates of the chain contained is well formed and can be uploaded to App Service Web Apps with confidence. Note the CA part of the uploaded pfx file will be discarded when we process the uploaded certificate, we store all the intermediate certificates associated with the certificate to enable the chain to be remade properly in the runtime. Once all the export operation is complete and you have successfully uploaded your certificate clean your machine of any trace of the SSL certificate by deleting the certificate from the store to secure your certificate.

[1]: https://azure.github.io/AppService/2017/02/24/Creating-a-local-PFX-copy-of-App-Service-Certificate.html
[2]: https://docs.microsoft.com/en-us/azure/app-service/configure-ssl-certificate?tabs=apex%2Cportal
[3]: https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-7.3.2
[4]: https://docs.microsoft.com/en-us/azure/app-service/configure-ssl-certificate?tabs=apex%2Cportal#start-certificate-order
[5]: ./import_cert_wizard.md
[6]: https://certs.godaddy.com/repository
[7]: ./export_cert_wizard.md
