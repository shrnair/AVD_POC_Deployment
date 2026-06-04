# AVD POC — Deployment Guide

## Architecture Overview

| Component | Config |
|-----------|--------|
| Host Pool | Pooled, breadth-first, 6× Standard_D8as_v5 |
| Max Users/Host | 10 |
| Total Capacity | 60 sessions |
| Network | 10.20.0.0/20 VNet (3 subnets) |
| Storage | Premium Azure Files (4 TiB), private endpoint |
| Identity | AD DS domain join, OU-based GPO control |
| Internet | NAT Gateway with static public IP |
| Golden Image | Windows 11 23H2 Enterprise Multi-Session, FSLogix, New Teams, Adobe Acrobat Reader DC, VDOT, Windows Updates |

```text
┌─────────────────────────────────────────────────────────────┐
│ VNet: 10.20.0.0/20 (AVD VNet)                               │
│                                                             │
│  ┌──────────────────┐    ┌──────────────────┐              │
│  │ Session Hosts    │    │ Private Endpoint │              │
│  │ 10.20.0.0/24     │    │ 10.20.1.0/27     │              │
│  │ avdsh-01..06     │    │ pe-storage       ├─→ Azure Files│
│  │ (no public IPs)  ├───→│ (SMB 445)        │              │
│  └────────┬─────────┘    └──────────────────┘              │
│           │                                                 │
│           v NAT GW                                          │
│      ↓ Internet/M365/Windows Update                         │
│                                                             │
│  ┌──────────────────┐                                       │
│  │ AD/DNS           │                                       │
│  │ 10.20.2.0/27     │                                       │
│  │ dc01 (10.20.2.4) │                                       │
│  │ dc02 (10.20.2.5) │                                       │
│  └────────┬─────────┘                                       │
└────────────────────────────────────────────────────────────┬┘
             │                                                │
             │ VNet Peering (bidirectional) ←───────────────→
             │
┌────────────────────────────────────────────────────────────┬┘
│ DC VNet (if separate)                                       │
│  ┌──────────────────┐                                       │
│  │ Domain Control   │                                       │
│  │ (on-prem or      │                                       │
│  │  separate VNet)  │                                       │
│  └──────────────────┘                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Pre-Flight: Gather These Values

| Item | How to Get It | Example |
|------|--------------|---------|
| Subscription ID | `az account show --query id -o tsv` | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| Domain Controller IP | Check the DC NIC in Azure Portal or run `ipconfig` on the DC | `10.0.0.4` |
| AD Domain FQDN | Run `echo %USERDNSDOMAIN%` on a domain-joined machine | `contoso.com` |
| Domain Admin UPN | Your AD admin account | `contoso.com\domainadmin` |
| DC VNet Name & RG | Azure Portal → DC VM → Networking | `vnet-dc` in `rg-dc` |
| AVD Users Group Object ID | Entra ID → Groups → your group → Object ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| Session Host OU DN | Run `Get-ADOrganizationalUnit -Filter *` on the DC | `OU=AVD-SessionHosts,OU=Azure,DC=contoso,DC=com` |
| Storage OU DN | Run `Get-ADOrganizationalUnit -Filter *` on the DC | `OU=AVD-Storage,OU=Azure,DC=contoso,DC=com` |

## Set Your Variables

```bash
SubscriptionId="<your-subscription-id>"
Location="eastus2"
RgName="rg-avd-poc"
VnetName="vnet-avd-poc"
VnetCidr="10.20.0.0/20"
SubnetHosts="snet-sessionhosts"
SubnetHostsCidr="10.20.0.0/24"
SubnetPe="snet-privateendpoints"
SubnetPeCidr="10.20.1.0/27"
NsgName="nsg-sessionhosts"
NatGwName="natgw-avd"
NatGwPipName="pip-natgw-avd"
AdDomain="contoso.com"
AdDcIp="10.0.0.4"
AdDomainAdmin="contoso.com\\domainadmin"
AdSessionHostOu="OU=AVD-SessionHosts,OU=Azure,DC=contoso,DC=com"
AdStorageOu="OU=AVD-Storage,OU=Azure,DC=contoso,DC=com"
DcVnetRg="rg-domain-controllers"
DcVnetName="vnet-dc"
HostPoolName="hp-avd-poc"
AppGroupName="dag-avd-poc"
WorkspaceName="ws-avd-poc"
MaxSessionLimit=10
AvdUsersGroupOid="<object-id-of-avd-group>"
SessionHostCount=6
VmSize="Standard_D8as_v5"
AlternateVmSize="Standard_D4as_v5"
VmNamePrefix="avdsh"
VmAdminUser="azureadmin"
VmAdminPassword="<strong-password>"
StorageAccountName="stavdfslogix$((RANDOM % 9000 + 1000))"
FileShareName="fslogix-profiles"
FileShareQuota=100
GalleryName="galAvdPoc"
ImageDefName="avd-win11-multisession"
ImageTemplateName="it-avd-golden"
IdentityName="id-imagebuilder"
InstallerStorageAccount="<your-installer-storage-account>"
InstallerContainer="installers"
```

> OU tip: use a dedicated OU for session hosts and a separate OU for the storage computer account.

## Prerequisites

| Check | Requirement |
|------|-------------|
| Yes | Azure CLI installed and signed in |
| Yes | `desktopvirtualization` CLI extension available |
| Yes | Domain controller reachable on 53, 88, 389, 445, and 636 |
| Yes | NAT Gateway approved for session host outbound internet |
| Yes | Storage account `$InstallerStorageAccount` exists |
| Yes | Container `$InstallerContainer` exists in that storage account |
| Yes | Teams and Adobe installers are uploaded to `$InstallerContainer` |
| Yes | Domain-joined Windows machine available for `Join-AzStorageAccount` |

## Install Required Tools

```bash
# Install or update Azure CLI (if not already installed)
winget install --id Microsoft.AzureCLI -e

# Install the desktopvirtualization extension (required for all AVD commands)
az extension add --name desktopvirtualization --upgrade

# Sign in to Azure
az login
az account set --subscription $SubscriptionId
```

Validation: `az extension list --query "[?name=='desktopvirtualization'].version" -o tsv`

## RBAC Roles Needed

| Role | Scope | Why |
|------|-------|-----|
| Contributor | Resource group | Create Azure resources |
| User Access Administrator | Resource group | Assign RBAC roles |
| Desktop Virtualization Contributor | Resource group | Manage host pool, app group, and workspace |
| Desktop Virtualization User | App group | Launch the desktop |
| Storage File Data SMB Share Contributor | Storage account | Access FSLogix profiles |
| Storage File Data SMB Share Elevated Contributor | Storage account | Optional admin access |
| Storage Blob Data Reader | `$InstallerStorageAccount` storage account | Read Teams and Adobe installers |

## Step 1: Create the Resource Group
Requires: Contributor on the target subscription or resource group scope.  
Creates the resource container for the AVD POC.

```bash
az group create --name $RgName --location $Location
```

Validation: `az group show --name $RgName --query "properties.provisioningState" -o tsv`

## Step 2: Create the VNet and Subnets
Requires: Contributor on the resource group.  
Creates the session host subnet, private endpoint subnet, and AD DNS setting.

```bash
az network vnet create --resource-group $RgName --name $VnetName --address-prefix $VnetCidr --location $Location
az network vnet subnet create --resource-group $RgName --vnet-name $VnetName --name $SubnetHosts --address-prefix $SubnetHostsCidr
az network vnet subnet create --resource-group $RgName --vnet-name $VnetName --name $SubnetPe --address-prefix $SubnetPeCidr
az network vnet update --resource-group $RgName --name $VnetName --dns-servers $AdDcIp
```

Validation: `az network vnet show --resource-group $RgName --name $VnetName --query "dhcpOptions.dnsServers" -o tsv`

## Step 3: Peer the AVD VNet to the DC VNet
Requires: Contributor on both VNets if the DC is in a separate VNet.  
Creates bidirectional connectivity between the AVD network and the DC network.

```bash
az network vnet peering create --resource-group $RgName --name avd-to-dc --vnet-name $VnetName --remote-vnet /subscriptions/$SubscriptionId/resourceGroups/$DcVnetRg/providers/Microsoft.Network/virtualNetworks/$DcVnetName --allow-vnet-access true
az network vnet peering create --resource-group $DcVnetRg --name dc-to-avd --vnet-name $DcVnetName --remote-vnet /subscriptions/$SubscriptionId/resourceGroups/$RgName/providers/Microsoft.Network/virtualNetworks/$VnetName --allow-vnet-access true
```

Validation: `az network vnet peering show --resource-group $RgName --vnet-name $VnetName --name avd-to-dc --query peeringState -o tsv`

## Step 4: Create the NSG and Associate It to the Session Host Subnet
Requires: Contributor on the resource group.  
Applies the network security group to the session host subnet.

```bash
az network nsg create --resource-group $RgName --name $NsgName
az network nsg rule create --resource-group $RgName --nsg-name $NsgName --name AllowOutboundHttps --priority 100 --direction Outbound --access Allow --protocol Tcp --destination-port-ranges 443
az network vnet subnet update --resource-group $RgName --vnet-name $VnetName --name $SubnetHosts --network-security-group $NsgName
```

Validation: `az network vnet subnet show --resource-group $RgName --vnet-name $VnetName --name $SubnetHosts --query "networkSecurityGroup.id" -o tsv`

## Step 5: Create the NAT Gateway and Associate It to the Session Host Subnet
Requires: Contributor on the resource group.  
Provides outbound internet for AVD registration, updates, and downloads.

```bash
az network public-ip create --resource-group $RgName --name $NatGwPipName --sku Standard --allocation-method Static
az network nat gateway create --resource-group $RgName --name $NatGwName --public-ip-addresses $NatGwPipName --idle-timeout 10
az network vnet subnet update --resource-group $RgName --vnet-name $VnetName --name $SubnetHosts --nat-gateway $NatGwName
```

Validation: `az network vnet subnet show --resource-group $RgName --vnet-name $VnetName --name $SubnetHosts --query "natGateway.id" -o tsv`

## Step 6: Create the AVD Host Pool
Requires: Contributor and Desktop Virtualization Contributor on the resource group.  
Creates the pooled host pool with breadth-first load balancing.

```bash
az desktopvirtualization hostpool create --resource-group $RgName --name $HostPoolName --host-pool-type Pooled --load-balancer-type BreadthFirst --max-session-limit $MaxSessionLimit --preferred-app-group-type Desktop --location $Location
```

Validation: `az desktopvirtualization hostpool show --resource-group $RgName --name $HostPoolName --query loadBalancerType -o tsv`

## Step 7: Create the Application Group
Requires: Contributor and Desktop Virtualization Contributor on the resource group.  
Creates the desktop application group for the host pool.

```bash
az desktopvirtualization applicationgroup create --resource-group $RgName --name $AppGroupName --application-group-type Desktop --host-pool-arm-path /subscriptions/$SubscriptionId/resourceGroups/$RgName/providers/Microsoft.DesktopVirtualization/hostPools/$HostPoolName --location $Location
```

Validation: `az desktopvirtualization applicationgroup show --resource-group $RgName --name $AppGroupName --query applicationGroupType -o tsv`

## Step 8: Create the Workspace and Link the App Group
Requires: Contributor and Desktop Virtualization Contributor on the resource group.  
Creates the workspace and publishes the desktop application group.

```bash
az desktopvirtualization workspace create --resource-group $RgName --name $WorkspaceName --location $Location --application-group-references /subscriptions/$SubscriptionId/resourceGroups/$RgName/providers/Microsoft.DesktopVirtualization/applicationGroups/$AppGroupName
```

Validation: `az desktopvirtualization workspace show --resource-group $RgName --name $WorkspaceName --query applicationGroupReferences -o tsv`

## Step 9: Assign Users to the App Group
Requires: User Access Administrator or Owner on the app group scope.  
Assigns the Desktop Virtualization User role to your AVD users group.

```bash
az role assignment create --assignee-object-id $AvdUsersGroupOid --assignee-principal-type Group --role "Desktop Virtualization User" --scope /subscriptions/$SubscriptionId/resourceGroups/$RgName/providers/Microsoft.DesktopVirtualization/applicationGroups/$AppGroupName
```

Validation: `az role assignment list --assignee $AvdUsersGroupOid --scope /subscriptions/$SubscriptionId/resourceGroups/$RgName/providers/Microsoft.DesktopVirtualization/applicationGroups/$AppGroupName --query "[].roleDefinitionName" -o tsv`

## Steps 10a & 10b: Custom Golden Image Build

> **Using Marketplace Image instead?** If the customer does not need a custom image, **skip Steps 10a and 10b entirely** and proceed to Step 11 Option B. Post-deployment steps (Step 23) will be required to manually install FSLogix, Teams, Adobe, and apply VDOT optimizations on each session host.

The following two steps create a custom golden image with all required software pre-installed (FSLogix, Teams, Adobe, VDOT, Windows Updates). VMs deployed from this image are ready to use immediately with no additional configuration.

### Step 10a: Create Installer Storage Account and Upload Installers

Requires: Contributor on the resource group, Storage Blob Data Contributor on the storage account.  
Creates a dedicated storage account to host application installers (Teams, Adobe) used during image build.

**Create the storage account and container:**

```bash
# Create the installer storage account
az storage account create --resource-group $RgName --name $InstallerStorageAccount --location $Location --sku Standard_LRS --kind StorageV2

# Create the installer container
az storage container create --account-name $InstallerStorageAccount --name $InstallerContainer --auth-mode login
```

**Download the installers:**
- **Adobe Acrobat Reader DC:** Download the enterprise offline installer from [https://get.adobe.com/reader/enterprise/](https://get.adobe.com/reader/enterprise/)
- **Microsoft Teams:** No manual download needed — the bootstrapper downloads the latest version automatically during image build (requires internet access on the build VM)

> **Note:** The WebRTC Redirector Service is also installed during image build for Teams media optimization (audio/video offload to client).

**Upload installers to blob storage:**

```bash
# Upload Adobe Acrobat Reader DC
az storage blob upload --account-name $InstallerStorageAccount --container-name $InstallerContainer --name AcroRdrDC2600121529_en_US.exe --file ./AcroRdrDC2600121529_en_US.exe --auth-mode login
```

Validation: `az storage blob list --account-name $InstallerStorageAccount --container-name $InstallerContainer --auth-mode login --query "[].name" -o tsv`

### Step 10b: Build the Golden Image

Requires: Contributor, User Access Administrator, and Storage Blob Data Reader on `$InstallerStorageAccount`.  
Builds a reusable image with FSLogix, Teams from blob, Adobe from blob, VDOT, and Windows Updates.

Blob note: make sure `$InstallerStorageAccount` has a container named `$InstallerContainer`, the Teams and Adobe files are uploaded (Step 10a), and the Image Builder managed identity has `Storage Blob Data Reader` on that storage account.

```bash
az provider register --namespace Microsoft.VirtualMachineImages
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.KeyVault
az identity create --resource-group $RgName --name $IdentityName
IdentityId=$(az identity show --resource-group $RgName --name $IdentityName --query id -o tsv)
IdentityPrincipalId=$(az identity show --resource-group $RgName --name $IdentityName --query principalId -o tsv)
az role assignment create --assignee-object-id $IdentityPrincipalId --assignee-principal-type ServicePrincipal --role Contributor --scope /subscriptions/$SubscriptionId/resourceGroups/$RgName
InstallerStorageId=$(az resource list --name $InstallerStorageAccount --resource-type Microsoft.Storage/storageAccounts --query '[0].id' -o tsv)
az role assignment create --assignee-object-id $IdentityPrincipalId --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $InstallerStorageId
az sig create --resource-group $RgName --gallery-name $GalleryName
az sig image-definition create --resource-group $RgName --gallery-name $GalleryName --gallery-image-definition $ImageDefName --publisher POC --offer Windows11AVD --sku win11-23h2-avd-custom --os-type Windows --os-state Generalized --hyper-v-generation V2 --features SecurityType=TrustedLaunch
```

**Create the image template JSON file:**

Save the following as `image-template.json` (provided in the repository as [`image-template-fixed.json`](./image-template-fixed.json)).

**Before deploying, replace all placeholders:**

Bash (Cloud Shell or Linux/Mac):
```bash
IdentityId=$(az identity show --resource-group $RgName --name $IdentityName --query id -o tsv)
sed -i "s|<INSERT_YOUR_IDENTITY_RESOURCE_ID>|$IdentityId|g" image-template-fixed.json
sed -i "s|<SUBSCRIPTION_ID>|$SubscriptionId|g" image-template-fixed.json
sed -i "s|<RG_NAME>|$RgName|g" image-template-fixed.json
sed -i "s|<GALLERY_NAME>|$GalleryName|g" image-template-fixed.json
sed -i "s|<IMAGE_DEF_NAME>|$ImageDefName|g" image-template-fixed.json
sed -i "s|<INSTALLER_STORAGE_ACCOUNT>|$InstallerStorageAccount|g" image-template-fixed.json
sed -i "s|<INSTALLER_CONTAINER>|$InstallerContainer|g" image-template-fixed.json
sed -i "s|<STORAGE_ACCOUNT_NAME>|$StorageAccountName|g" image-template-fixed.json
sed -i "s|<FILE_SHARE_NAME>|$FileShareName|g" image-template-fixed.json
```

PowerShell (Windows):
```powershell
$IdentityId = az identity show --resource-group $RgName --name $IdentityName --query id -o tsv
$json = Get-Content ./image-template-fixed.json -Raw
$json = $json.Replace('<INSERT_YOUR_IDENTITY_RESOURCE_ID>', $IdentityId)
$json = $json.Replace('<SUBSCRIPTION_ID>', $SubscriptionId)
$json = $json.Replace('<RG_NAME>', $RgName)
$json = $json.Replace('<GALLERY_NAME>', $GalleryName)
$json = $json.Replace('<IMAGE_DEF_NAME>', $ImageDefName)
$json = $json.Replace('<INSTALLER_STORAGE_ACCOUNT>', $InstallerStorageAccount)
$json = $json.Replace('<INSTALLER_CONTAINER>', $InstallerContainer)
$json = $json.Replace('<STORAGE_ACCOUNT_NAME>', $StorageAccountName)
$json = $json.Replace('<FILE_SHARE_NAME>', $FileShareName)
$json | Set-Content ./image-template-fixed.json
```

**Deploy and build the image:**

```bash
az resource create --resource-group $RgName --resource-type Microsoft.VirtualMachineImages/imageTemplates --name $ImageTemplateName --is-full-object --properties @image-template-fixed.json
az resource invoke-action --resource-group $RgName --resource-type Microsoft.VirtualMachineImages/imageTemplates --name $ImageTemplateName --action Run
```

Validation: `az sig image-version list --resource-group $RgName --gallery-name $GalleryName --gallery-image-definition $ImageDefName -o table`

## Step 11: Deploy the Session Host VMs
Requires: Contributor on the resource group.  

### Step 11a: Verify VM Size Availability
Requires: Reader on the subscription.  
Checks that your chosen VM size is available in your region and subscription, and lets you pick an alternative before deployment if needed.

```bash
az vm list-skus --location $Location --size $VmSize --output table
az vm list-skus --location $Location --resource-type virtualMachines --query "[?name=='$AlternateVmSize'].{Name:name, Zones:locationInfo[0].zones, Restrictions:restrictions[0].reasonCode}" --output table
```

Validation: `az vm list-skus --location $Location --size $VmSize --query "[?name=='$VmSize'].name" -o tsv`

### Option A: Deploy from Custom Gallery Image (completed Step 10)

Creates the session host VMs from the custom gallery image built in Step 10.

```bash
for i in $(seq 1 $SessionHostCount); do
  VmName=$(printf "%s-%02d" "$VmNamePrefix" "$i")
  az vm create --resource-group $RgName --name $VmName --image /subscriptions/$SubscriptionId/resourceGroups/$RgName/providers/Microsoft.Compute/galleries/$GalleryName/images/$ImageDefName/versions/latest --size $VmSize --vnet-name $VnetName --subnet $SubnetHosts --admin-username $VmAdminUser --admin-password $VmAdminPassword --public-ip-address "" --nsg "" --security-type TrustedLaunch
done
```

### Option B: Deploy from Marketplace Image (skipped Step 10)

Creates the session host VMs directly from a Microsoft marketplace image. Use this when the customer does not require a custom golden image.

```bash
for i in $(seq 1 $SessionHostCount); do
  VmName=$(printf "%s-%02d" "$VmNamePrefix" "$i")
  az vm create --resource-group $RgName --name $VmName --image MicrosoftWindowsDesktop:windows-11:win11-23h2-avd:latest --size $VmSize --vnet-name $VnetName --subnet $SubnetHosts --admin-username $VmAdminUser --admin-password $VmAdminPassword --public-ip-address "" --nsg "" --security-type TrustedLaunch
done
```

> **Important (Option B only):** Since the marketplace image does not include pre-installed software, you must complete **Step 23** after deployment to manually configure FSLogix, and install Teams and Adobe Acrobat Reader on each session host.

Validation: `az vm list --resource-group $RgName --query "[].name" -o tsv`

## Step 12: Domain-Join the Session Hosts
Requires: Contributor on the resource group and AD rights to join computers to the target OU.  
Adds each session host to Active Directory by using the JsonADDomainExtension.

```bash
for i in $(seq 1 $SessionHostCount); do
  VmName=$(printf "%s-%02d" "$VmNamePrefix" "$i")
  az vm extension set --resource-group $RgName --vm-name $VmName --name JsonADDomainExtension --publisher Microsoft.Compute --version 1.3 --settings '{"Name":"'"$AdDomain"'","User":"'"$AdDomainAdmin"'","OUPath":"'"$AdSessionHostOu"'","Restart":"true","Options":"3"}' --protected-settings '{"Password":"<DOMAIN_ADMIN_PASSWORD>"}'
done
```

Validation: `az vm extension list --resource-group $RgName --vm-name $(printf "%s-%02d" "$VmNamePrefix" 1) --query "[?name=='JsonADDomainExtension'].provisioningState" -o tsv`

## Step 13: Register the Session Hosts with AVD
Requires: Contributor and Desktop Virtualization Contributor on the resource group.  
Registers each VM with the host pool by using the AVD DSC extension and a registration token.

```bash
Expiration=$(date -u -d "+1 day" +"%Y-%m-%dT%H:%M:%SZ")
az desktopvirtualization hostpool update --resource-group $RgName --name $HostPoolName --registration-info expiration-time="$Expiration" registration-token-operation="Update"
Token=$(az desktopvirtualization hostpool retrieve-registration-token --resource-group $RgName --name $HostPoolName --query token -o tsv)
for i in $(seq 1 $SessionHostCount); do
  VmName=$(printf "%s-%02d" "$VmNamePrefix" "$i")
  az vm extension set --resource-group $RgName --vm-name $VmName --name DSC --publisher Microsoft.Powershell --version 2.77 --settings '{"modulesUrl":"https://wvdportalstorageblob.blob.core.windows.net/galleryartifacts/Configuration_1.0.02714.342.zip","configurationFunction":"Configuration.ps1\\AddSessionHost","properties":{"hostPoolName":"'"$HostPoolName"'","registrationInfoToken":"'"$Token"'","aadJoin":false}}'
done
```

Validation: `az rest --method GET --uri "/subscriptions/$SubscriptionId/resourceGroups/$RgName/providers/Microsoft.DesktopVirtualization/hostPools/$HostPoolName/sessionHosts?api-version=2024-04-03" --query "value[].{Name:name,Status:properties.status}" -o table`

## Step 14: Create the Storage Account
Requires: Contributor on the resource group.  
Creates the Premium FileStorage account for FSLogix profiles with public access denied.

```bash
az storage account create --resource-group $RgName --name $StorageAccountName --kind FileStorage --sku Premium_LRS --location $Location --default-action Deny
```

Validation: `az storage account show --resource-group $RgName --name $StorageAccountName --query "networkRuleSet.defaultAction" -o tsv`

## Step 15: Create the File Share
Requires: Contributor on the resource group.  
Creates the Azure file share that stores FSLogix profile containers.

```bash
az storage share-rm create --resource-group $RgName --storage-account $StorageAccountName --name $FileShareName --quota $FileShareQuota
```

Validation: `az storage share-rm show --resource-group $RgName --storage-account $StorageAccountName --name $FileShareName --query name -o tsv`

## Step 16: Domain-Join the Storage Account
Requires: Domain join rights and a domain-joined Windows machine (e.g., the Domain Controller).  
Adds the storage account to Active Directory so SMB access can use AD identities.

> **Reference:** Follow the detailed instructions at [Configure FSLogix profile container with Azure Files and AD DS](https://learn.microsoft.com/en-us/fslogix/how-to-configure-profile-container-azure-files-active-directory?tabs=adds) for downloading and setting up the AzFilesHybrid module.

**Steps (run on a domain-joined machine in an elevated PowerShell session):**

```powershell
# 1. Download and extract the AzFilesHybrid module
#    Download from: https://github.com/Azure-Samples/azure-files-samples/releases
#    Extract the zip to a folder, e.g., C:\AzFilesHybrid

# 2. Navigate to the extracted folder and run the setup
Set-Location -Path "C:\AzFilesHybrid"
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser
.\CopyToPSPath.ps1

# 3. Import the module and connect to Azure
Import-Module -Name AzFilesHybrid
Connect-AzAccount

# 4. Set variables
$SubscriptionId = "<your-subscription-id>"
$RgName = "rg-avd-poc"
$StorageAccountName = "<your-storage-account-name>"

Select-AzSubscription -SubscriptionId $SubscriptionId

# 5. Join the storage account to AD (omit -OrganizationalUnitDistinguishedName to use the default Computers container)
Join-AzStorageAccount -ResourceGroupName $RgName -StorageAccountName $StorageAccountName -DomainAccountType ComputerAccount
```

> **Note:** If you have a specific OU for storage accounts, add `-OrganizationalUnitDistinguishedName "OU=StorageAccounts,DC=yourdomain,DC=com"` to the `Join-AzStorageAccount` command.

Validation: `az storage account show --resource-group $RgName --name $StorageAccountName --query "azureFilesIdentityBasedAuthentication.directoryServiceOptions" -o tsv`

## Step 17: Set the Default Share Permission
Requires: Contributor on the storage account.  
Sets the default Azure Files share permission for identity-based SMB access.

```bash
az storage account update --resource-group $RgName --name $StorageAccountName --default-share-permission StorageFileDataSmbShareContributor
```

Validation: `az storage account show --resource-group $RgName --name $StorageAccountName --query "azureFilesIdentityBasedAuthentication.defaultSharePermission" -o tsv`

## Step 18: Assign Storage RBAC to the AVD Users Group
Requires: User Access Administrator or Owner on the storage account.  
Grants the AVD users group access to the FSLogix share.

```bash
StorageId=$(az storage account show --name $StorageAccountName --resource-group $RgName --query id -o tsv)
az role assignment create --assignee-object-id $AvdUsersGroupOid --assignee-principal-type Group --role "Storage File Data SMB Share Contributor" --scope $StorageId
```

Validation: `az role assignment list --assignee $AvdUsersGroupOid --scope $StorageId --query "[].roleDefinitionName" -o tsv`

## Step 19: Create the Private Endpoint for Storage
Requires: Contributor on the resource group.  
Creates the file private endpoint for the storage account.

```bash
az network private-endpoint create --resource-group $RgName --name pe-$StorageAccountName --vnet-name $VnetName --subnet $SubnetPe --private-connection-resource-id $StorageId --group-id file --connection-name pe-conn-fslogix
```

Validation: `az network private-endpoint show --resource-group $RgName --name pe-$StorageAccountName --query provisioningState -o tsv`

## Step 20: Create the Private DNS Zone and Link It to the VNets
Requires: Contributor on the resource group and linked VNets.  
Creates the Azure Files private DNS zone and links it to the AVD VNet and DC VNet.

```bash
az network private-dns zone create --resource-group $RgName --name privatelink.file.core.windows.net
az network private-dns link vnet create --resource-group $RgName --zone-name privatelink.file.core.windows.net --name link-avd-vnet --virtual-network $VnetName --registration-enabled false
az network private-dns link vnet create --resource-group $RgName --zone-name privatelink.file.core.windows.net --name link-dc-vnet --virtual-network /subscriptions/$SubscriptionId/resourceGroups/$DcVnetRg/providers/Microsoft.Network/virtualNetworks/$DcVnetName --registration-enabled false
```

Validation: `az network private-dns link vnet list --resource-group $RgName --zone-name privatelink.file.core.windows.net -o table`

## Step 21: Create the DNS Zone Group on the Private Endpoint
Requires: Contributor on the resource group.  
Associates the private endpoint with the private DNS zone so the A record (`$StorageAccountName.privatelink.file.core.windows.net`) is created automatically.

```bash
PrivateDnsZoneId=$(az network private-dns zone show --resource-group $RgName --name privatelink.file.core.windows.net --query id -o tsv)
az network private-endpoint dns-zone-group create --resource-group $RgName --endpoint-name pe-$StorageAccountName --name default --private-dns-zone $PrivateDnsZoneId --zone-name file
```

Validation: Confirm the A record was created in the private DNS zone:
```bash
az network private-dns record-set a list --resource-group $RgName --zone-name privatelink.file.core.windows.net --query "[].{Name:name,IP:aRecords[0].ipv4Address}" -o table
```

## Step 22: Add the Conditional Forwarder on the Domain Controller
Requires: DNS admin rights on the domain controller.  
Makes the DC forward `privatelink.file.core.windows.net` queries to Azure DNS at `168.63.129.16`.

```bash
powershell -Command 'Add-DnsServerConditionalForwarderZone -Name "privatelink.file.core.windows.net" -MasterServers 168.63.129.16 -ReplicationScope Forest'
```

Validation: `nslookup "${StorageAccountName}.file.core.windows.net" 127.0.0.1`

## Step 22b: Set NTFS Permissions on the File Share
Requires: A domain-joined machine with access to the file share (private endpoint and DNS must be configured first).  
Configures NTFS-level permissions so each user can create and access their own FSLogix profile container while being unable to access other users' profiles.

**From a domain-joined machine (e.g., a session host), map the file share:**

Option 1 — Using storage account key:
```powershell
# Get the storage account key
$StorageKey = (az storage account keys list --account-name $StorageAccountName --resource-group $RgName --query "[0].value" -o tsv)

# Map the share using storage key
net use Z: \\$StorageAccountName.file.core.windows.net\$FileShareName /user:Azure\$StorageAccountName $StorageKey
```

Option 2 — Using domain credentials (requires storage account domain-joined in Step 16):
```powershell
# Log in as a domain admin on a domain-joined machine
# Kerberos auth is used automatically
net use Z: \\$StorageAccountName.file.core.windows.net\$FileShareName
```

**Set NTFS permissions:**

```powershell
# Remove inheritance and existing permissions
icacls Z:\ /inheritance:r

# Grant permissions
icacls Z:\ /grant "CREATOR OWNER:(OI)(CI)(IO)F"
icacls Z:\ /grant "NT AUTHORITY\SYSTEM:(OI)(CI)F"
icacls Z:\ /grant "Domain Admins:(OI)(CI)F"
icacls Z:\ /grant "AVD-Users:(M)"
icacls Z:\ /grant "AVD-Users:(CI)(DC)(AD)(R)"

# Remove the share mapping
net use Z: /delete
```

> **Note:** Replace `AVD-Users` with your actual AD group name for AVD users. Replace `Domain Admins` with your admin group if different.

Validation: `icacls \\$StorageAccountName.file.core.windows.net\$FileShareName`

## Step 23: Install FSLogix on Session Hosts (Option B — Marketplace Image)
Requires: Contributor on the resource group.  
**Required if you chose Option B (marketplace image) in Step 11.** Uses a Custom Script Extension to download and install FSLogix silently on all session hosts.

```bash
for i in $(seq 1 $SessionHostCount); do
  VmName=$(printf "%s-%02d" "$VmNamePrefix" "$i")
  az vm run-command invoke --resource-group $RgName --name $VmName --command-id RunPowerShellScript --scripts "
    Invoke-WebRequest -Uri 'https://aka.ms/fslogix_download' -OutFile C:\fslogix.zip
    Expand-Archive -Path C:\fslogix.zip -DestinationPath C:\FSLogix -Force
    Start-Process C:\FSLogix\x64\Release\FSLogixAppsSetup.exe -ArgumentList '/install /quiet /norestart' -Wait
    Remove-Item C:\fslogix.zip -Force
  "
done
```

Validation: `az vm run-command invoke --resource-group $RgName --name $(printf "%s-%02d" "$VmNamePrefix" 1) --command-id RunPowerShellScript --scripts "Get-Service frxsvc | Select-Object Status"`

## Step 24: Configure FSLogix Registry Settings (Option B — Marketplace Image)
Requires: Contributor on the resource group.  
**Required if you chose Option B (marketplace image) in Step 11.** Configures FSLogix profile container settings on all session hosts via run-command.

```bash
for i in $(seq 1 $SessionHostCount); do
  VmName=$(printf "%s-%02d" "$VmNamePrefix" "$i")
  az vm run-command invoke --resource-group $RgName --name $VmName --command-id RunPowerShellScript --scripts "
    New-Item -Path 'HKLM:\SOFTWARE\FSLogix\Profiles' -Force
    Set-ItemProperty -Path 'HKLM:\SOFTWARE\FSLogix\Profiles' -Name Enabled -Value 1 -Type DWord
    Set-ItemProperty -Path 'HKLM:\SOFTWARE\FSLogix\Profiles' -Name VHDLocations -Value '\\\\$StorageAccountName.file.core.windows.net\\$FileShareName' -Type String
    Set-ItemProperty -Path 'HKLM:\SOFTWARE\FSLogix\Profiles' -Name DeleteLocalProfileWhenVHDShouldApply -Value 1 -Type DWord
    Set-ItemProperty -Path 'HKLM:\SOFTWARE\FSLogix\Profiles' -Name FlipFlopProfileDirectoryName -Value 1 -Type DWord
    Set-ItemProperty -Path 'HKLM:\SOFTWARE\FSLogix\Profiles' -Name SizeInMBs -Value 30000 -Type DWord
    Set-ItemProperty -Path 'HKLM:\SOFTWARE\FSLogix\Profiles' -Name VolumeType -Value 'VHDX' -Type String
    Restart-Computer -Force
  "
done
```

Validation: `az vm run-command invoke --resource-group $RgName --name $(printf "%s-%02d" "$VmNamePrefix" 1) --command-id RunPowerShellScript --scripts "Get-ItemProperty 'HKLM:\SOFTWARE\FSLogix\Profiles' | Select-Object VHDLocations"`

## Step 25: Run End-to-End Validation
Requires: Reader in Azure, local admin on a session host, and an AVD test user assigned to the app group.  
Confirms host registration, private DNS, SMB profile access, and desktop sign-in.

```bash
az rest --method GET --uri "/subscriptions/$SubscriptionId/resourceGroups/$RgName/providers/Microsoft.DesktopVirtualization/hostPools/$HostPoolName/sessionHosts?api-version=2024-04-03" --query "value[].{Name:name,Status:properties.status}" -o table
az network private-endpoint show --resource-group $RgName --name pe-$StorageAccountName --query "customDnsConfigs[].ipAddresses[]" -o tsv
nslookup "${StorageAccountName}.file.core.windows.net"
```

Validation: Sign in to `https://client.wvd.microsoft.com/arm/webclient`, launch the desktop, create a file, sign out, sign back in, and confirm the file persists.

