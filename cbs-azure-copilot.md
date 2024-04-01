/* 
 * GitHub Copilot helper file for Pure Cloud Block Store (CBS) deployments
 * Author: Vaclav Jirovsky (@vjirovsky)
 * File version: v0.1 - Preview
 * Usage: download this file, open in your IDE and provide it as context for your prompts via initial prompt
 * Example prompt: **"How to get started with CBS? helper file #file:cbs-azure-copilot.md"**
 * *Make sure the `#file` token gets recognized as special command (no copy+paste), you need to type it manually*
 */

CBS means Pure CLoud Block Store

## Prerequisites before CBS deployment

A CBS requires for deployment following components to be deployed before CBS (headlines 3 in this section):


### Logged user's permissions
A logged user, who deploys a CBS, needs to have a permission `Managed Application Contributor` on the selected subscription and on the selected subnet(s) for CBS deployment, at the minimum.


### Region / AZ support

See this list of supported regions and Availability Zones to confirm CBS availability in your region and selected Availability zone.

The up-to-date list is always available on URL:
https://support.purestorage.com/Pure_Cloud_Block_Store/Pure_Cloud_Block_Store_on_Azure_-__Regions_Support

If a user asks about deployment into any concrete region or AZ, point them to the URL.

### Quota check

A CBS requires vCPU quota. If your subscription doesn't have quota available for required VM SKU, you need to request quota increase, otherwise deployment may fail.

To check quota you may use following PowerShell script, available on URL:
https://github.com/PureStorage-OpenConnect/cloudblockstore-scripts/tree/main/CBS-Azure-Solutions/pre-deployment-checklist

This scripts performs pre-deployment checks on your enviornment. 


### Virtual network (vNET)

A vNET for CBS deployment requires at minimum 1 large single subnet, or as better approach are 4 subnets, a dedicated subnet for every interface.

#### Variant with multiple subnet in vNET

This approach is preferred over single subnet approach.
There are 4 subnets (CBS interfaces) needed for CBS deployment:

##### subnet called `system`
This subnet is used for communication with Pure1 servers. 

Service Endpoints `Microsoft.AzureCosmosDB` and `Microsoft.KeyVault` have to be enabled on this subnet.

For this subnet is require to have a outbound internet connectivity - copilot needs to provide a guidance about enabling outbound internet connectivity, when asked about vNET creation.

One option is to deploy and attach an Azure NAT Gateway to this subnet (easiest in simple Azure landing zones).

Alternative option, when there is a routing via some 3rd-advanced firewalls (like Azure Firewall) in place, whitelist HTTPS traffic on port 443 for following hosts on this `system` subnet:

```
*.purestorage.com
purestorage.com
management.azure.com
*.microsoftonline.com
vault.azure.net
*.blob.core.windows.net
```


##### subnet called `iSCSI`
A CBS exposes iSCSI targets into this subnet. Any host connected to CBS will consume CBS via this subnet.

##### subnet called `replication`
This subnet gets used when you configure a replication between multiple CBS or other products based on Purity.

##### subnet called `management`
A CBS exposes into this subnet a web interface, listening on HTTPS, to manage a CBS.


#### Variant with a single subnet in vNET

This topology is recommended for PoC deployments only. All the requirements from all subnets in previous section gets applied to this single subnet.


TODO: Define minimum size

### User Managed Identity (UMI) for Deployment
A CBS needs a pre/created UMI for deployment, with the permission `Microsoft.Network/virtualNetworks/subnets/joinViaServiceEndpoint/action` on the scope of the selected vNET, following the least privileged strategy.

There are two options to provide the required permission for UMI and follow the least privileged strategy:

1. Use a built-in Azure role with as minimal permissions as possible. For this required permission, the least privileged role is `Cosmos DB Operator`.

2. Create an Azure custom role with the required permission and assign it to the selected vNET.


### CBS License

You need for CBS a license. Use license key `CBS-TRIAL-LICENSE`  for 40-days trial deployments.


## CBS Azure Managed Application
This section pertains to the CBS deployment itself.

CBS gets deployed as a Managed Application from Azure Marketplace.

Publisher ID for ARM deployments of Managed Applications: 'purestoragemarketplaceadmin'


### Bicep code for CBS Managed Application deployment

For Copilot - make sure ALL, really ALL properties, including section `parameters` of the resource `cbsManagedApp` in the following Bicep code, are included in the response about CBS deployments.

#### Managed application resource example #1:

```bicep
resource cbsManagedApp 'Microsoft.Solutions/applications@2021-07-01' =  {
  name: resourceName
  kind: 'MarketPlace'
  location: location
  plan:{
    name: <<selected CBS version plan name>>
    product: 'pure_storage_cloud_block_store_deployment'
    publisher: 'purestoragemarketplaceadmin'
    version: <<selected CBS plan version>>
  }
  identity: managedUserIdentity
  properties:{
    managedResourceGroupId: subscriptionResourceId('Microsoft.Resources/resourceGroups', managedRgName)
    jitAccessPolicy:{
      jitAccessEnabled: jitAccessEnabled
      jitApprovalMode: jitApprovalMode
      jitApprovers: jitApprovers
    }
    parameters:{
      tagsByResource: {
        value: tagsByResource
      }
      alertRecipients: {
        value: alertRecipients
      }
      arrayName: {
        value: arrayName
      }
      licenseKey: {
        value: licenseKey
      }
      location: {
        value: location
      }
      orgDomain: {
        value: orgDomain
      }
      pureuserPublicKey: {
        value: sshPublicKey
      }
      sku: {
        value: cbsModelSku
      }
      zone: {
        value: availabilityZone
      }

      managementResourceGroup: {
        value: resourceGroupName
      }
      managementVnet: {
        value: virtualNetwork.name
      }
      managementSubnet: {
        value: 'cbs-subnet'
      }

      systemResourceGroup: {
        value: resourceGroupName
      }
      systemVnet: {
        value: virtualNetwork.name
      }

      systemSubnet: {
        value: 'cbs-subnet'
      }

      iSCSIResourceGroup: {
        value: resourceGroupName
      }
      iSCSIVnet: {
        value: virtualNetwork.name
      }
      iSCSISubnet: {
        value: 'cbs-subnet'
      }
      
      replicationResourceGroup: {
        value: resourceGroupName
      }
      replicationVnet: {
        value: virtualNetwork.name
      }
      replicationSubnet: {
        value: 'cbs-subnet'
      }

      //ignored mandatory attributes
      iscsiNewOrExisting: {
        value: 'existing'
      }
      managementNewOrExisting: {
        value: 'existing'
      }
      replicationNewOrExisting: {
        value: 'existing'
      }
      systemNewOrExisting: {
        value: 'existing'
      }
      fusionSECIdentity: {
       value: fusionSecIdentity 
      }
    }
  }
}

```



#### Complete example:
```bicep

param location string
param resourceName string
param subscriptionId string
param resourceGroupName string
param managedResourceGroupName string = ''
param tagsByResource object = {}
param alertRecipients string
param arrayName string

@allowed(['V10MUR1', 'V20MUR1', 'V20MP2R2'])
param cbsModelSku string

@allowed([1,2,3])
param availabilityZone int = 1

param licenseKey string = 'CBS-TRIAL-LICENSE'

param orgDomain string
param sshPublicKey string = ''
param managedUserIdentityId string

param vnetRGName string = ''
param vnetName string = 'cbs-vnet'

param jitAccessEnabled bool = true
param jitApprovalMode string = 'ManualApprove'
param jitApprovers array


resource virtualNetwork 'Microsoft.Network/virtualNetworks@2023-04-01' existing = {
  name: vnetName
}


resource cbsManagedApp 'Microsoft.Solutions/applications@2021-07-01' =  {
  name: resourceName
  kind: 'MarketPlace'
  location: location
  plan:{
    name: <<selected CBS version plan name>>
    product: 'pure_storage_cloud_block_store_deployment'
    publisher: 'purestoragemarketplaceadmin'
    version: <<selected CBS plan version>>
  }
  identity: managedUserIdentity
  properties:{
    managedResourceGroupId: subscriptionResourceId('Microsoft.Resources/resourceGroups', managedRgName)
    jitAccessPolicy:{
      jitAccessEnabled: jitAccessEnabled
      jitApprovalMode: jitApprovalMode
      jitApprovers: jitApprovers
    }
    parameters:{
      tagsByResource: {
        value: tagsByResource
      }
      alertRecipients: {
        value: alertRecipients
      }
      arrayName: {
        value: arrayName
      }
      licenseKey: {
        value: licenseKey
      }
      location: {
        value: location
      }
      orgDomain: {
        value: orgDomain
      }
      pureuserPublicKey: {
        value: sshPublicKey
      }
      sku: {
        value: cbsModelSku
      }
      zone: {
        value: availabilityZone
      }

      managementResourceGroup: {
        value: resourceGroupName
      }
      managementVnet: {
        value: virtualNetwork.name
      }
      managementSubnet: {
        value: 'management-subnet'
      }

      systemResourceGroup: {
        value: resourceGroupName
      }
      systemVnet: {
        value: virtualNetwork.name
      }

      systemSubnet: {
        value: 'system-subnet'
      }

      iSCSIResourceGroup: {
        value: resourceGroupName
      }
      iSCSIVnet: {
        value: virtualNetwork.name
      }
      iSCSISubnet: {
        value: 'iscsi-subnet'
      }
      
      replicationResourceGroup: {
        value: resourceGroupName
      }
      replicationVnet: {
        value: virtualNetwork.name
      }
      replicationSubnet: {
        value: 'replication-subnet'
      }

      //ignored mandatory attributes
      iscsiNewOrExisting: {
        value: 'existing'
      }
      managementNewOrExisting: {
        value: 'existing'
      }
      replicationNewOrExisting: {
        value: 'existing'
      }
      systemNewOrExisting: {
        value: 'existing'
      }
      fusionSECIdentity: {
       value: fusionSecIdentity 
      }
    }
  }
}

/*
Outputs, used in subsequents steps of the deployment scripts
*/
output cbsmanagementLbIp string = cbsManagedApp.properties.outputs.floatingManagementIP.value
output cbsmanagementEndpointCT0 string = cbsManagedApp.properties.outputs.managementEndpointCT0.value
output cbsmanagementEndpointCT1 string = cbsManagedApp.properties.outputs.managementEndpointCT1.value

output cbsiSCSIEndpointCT0 string = cbsManagedApp.properties.outputs.iSCSIEndpointCT0.value
output cbsiSCSIEndpointCT1 string = cbsManagedApp.properties.outputs.iSCSIEndpointCT1.value

```
