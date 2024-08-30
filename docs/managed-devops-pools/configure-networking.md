---
title: Configure networking
description: Learn how to configure networking for Managed DevOps Pools.
ms.date: 08/22/2024
---

# Configure Managed DevOps Pools networking

## Adding agents to your own Virtual network

You may want to add agents from Managed DevOps Pools to your own virtual network for scenarios such as:

* Your CI/CD Agents need to access resources that are only available in your company network through a service like Express Route
* Your CI/CD Agents need to access resources that are isolated to private endpoints
* You want to network isolate your CI/CD infrastructure by bringing your own VNet with company specific firewall rules
* Any other unique use cases that can't be achieved by out-of-box Managed DevOps Pools networking related features

You can add your pool's agents to your virtual network using the following steps.

1. [Create or bring your virtual network and subnet](#create-or-bring-your-virtual-network-and-subnet)
2. [Delegate the subnet to Microsoft.DevOpsInfrastructure/pools](#delegate-the-subnet-to-microsoftdevopsinfrastructurepools)
3. [Associate the subnet with your Managed DevOps Pool](#associate-the-subnet-with-your-managed-devops-pool)

The previous steps delegate the subnet for exclusive access by the pool and the subnet can't be used by other pools or resources.
In order to connect multiple pools to the same virtual network, multiple subnets can be used, each delegated and associated with their own pool.

## Create or bring your virtual network and subnet

The subnet must have enough address space to accommodate the max pool size of the pool you want to associate (include the 5 IP address Azure reserves in the subnet).
If you're using Express Route, you need to temporary drop or change the management lock on the resource group to allow writes.

> [!IMPORTANT]
> The Managed DevOps Pool and virtual network must be in the same region, or you'll get an error similar to the following when you try to create the pool or update the network configuration. `Virtual network MDPVN is in region eastus, but pool mdpnonprodsub is in region australiaeast. These must be in the same region.`

Ensure the DevOpsInfrastructure principal has the following access on the virtual network:
- `Reader` and `Network Contributor`
- OR add the following permission to a custom role:
  - `Microsoft.Network/virtualNetworks/*/read`
  - `Microsoft.Network/virtualNetworks/subnets/join/action`
  - `Microsoft.Network/virtualNetworks/subnets/serviceAssociationLinks/validate/action`
  - `Microsoft.Network/virtualNetworks/subnets/serviceAssociationLinks/write`
  - `Microsoft.Network/virtualNetworks/subnets/serviceAssociationLinks/delete`

Make a custom role for the **Service Association Link** access. An example role can be created at the resource group or subscription level in the Access Control tab, as shown in the following example.

:::image type="content" source="./media/configure-networking/customrole.png" alt-text="Screenshot of custom role permissions.":::

### To check the DevOpsInfrastructure principal access

1. Choose **Access control (IAM)** for the virtual network, and choose **Check access**.

   :::image type="content" source="./media/configure-networking/permissions-check-access.png" alt-text="Screenshot of VNet Permissions for SubNet Delegation.":::

2. Search for **DevOpsInfrastructure** and select it.

   :::image type="content" source="./media/configure-networking/devops-infrastructure-access.png" alt-text="Screenshot of selecting AzureDevOpsInfrastructure principal.":::

3. Verify **Reader** access. Verify that `Microsoft.Network/virtualNetworks/subnets/join/action`, `Microsoft.Network/virtualNetworks/subnets/serviceAssociationLinks/validate/action` and `Microsoft.Network/virtualNetworks/subnets/serviceAssociationLinks/write` access is assigned. Your custom role should appear here.

   :::image type="content" source="./media/configure-networking/verify-permissions.png" alt-text="Screenshot of VNet Permissions.":::

4. If **DevOpsInfrastructure** doesn't have those permissions, add them by choosing **Access control (IAM)** for the virtual network, and choose **Grant access to this resource** and add them.

## Delegate the subnet to Microsoft.DevOpsInfrastructure/pools

The subnet needs to be delegated to the `Microsoft.DevOpsInfrastructure/pools` to be used.
Open the subnet properties in the Portal and select `Microsoft.DevOpsInfrastructure/pools` under the Subnet Delegation section and choose **Save**.

:::image type="content" source="./media/configure-networking/MDP-BYO-Networking-MDPSubNet-Delegation.png" alt-text="Screenshot of configuring the subnet delegation.":::

This delegates the subnet for exclusive access for the pool and the subnet can't be used by other pools or resources. In order to connect multiple pools to the same virtual network, multiple subnets must be used, each delegated and associated with their own pool. More information on subnet delegation can be found [here](/azure/virtual-network/subnet-delegation-overview).

Once the subnet is delegated to `Microsoft.DevOpsInfrastructure/pools`, the pool can be updated to use the subnet.

## Associate the subnet with your Managed DevOps Pool

#### [Azure portal](#tab/azure-portal/)

1. If you're creating a new pool, go to the **Networking** tab. To update an existing pool, go to **Settings** > **Networking**, and choose **Agents injected into existing virtual network**, **Configure**.

   :::image type="content" source="./media/configure-networking/inject-agents-configure-option.png" alt-text="Screenshot of configure option.":::

1. Choose the **Subscription**, **Virtual Network**, and **Subnet** you delegated to `Microsoft.DevOpsInfrastructure/pools`, and choose **Ok**.

   :::image type="content" source="./media/configure-networking/subnet-resource.png" alt-text="Screenshot of associating the subnet to the pool.":::

Once the network update completes, newly created resource in the pool will use the delegated subnet.

#### [ARM template](#tab/arm/)

If you are using ARM templates, add a `networkProfile` property if it doesn't already exist, then add a `subnetId` property under `networkProfile` with the resource ID of your subnet. 

```json
{
    "name": "MyManagedDevOpsPool",
    "type": "Microsoft.DevOpsInfrastructure/pools",
    "apiVersion": "2024-04-04-preview",
    "location": "eastus",
    "properties": {
        ...
        "networkProfile": {
          "subnetId":"/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/myResourceGroup/providers/Microsoft.Network/virtualNetworks/myVirtualNetwork/subnets/mySubnet",
        }
        ...
    }
}
```

* * *

## Restricting outbound connectivity

If you have systems in place on your network (NSG, Firewall etc.) which restrict outbound connectivity, you need to ensure that the following domains can be accessed, otherwise your Managed DevOps Pool will not be functional.
All of them are HTTPS, unless otherwise stated.

* Highly secure endpoints that our service depends on:
  *  `*.prod.manageddevops.microsoft.com` - Managed DevOps Pools endpoint
  *  `rmprodbuilds.azureedge.net` - Worker binaries
  *  `vstsagentpackage.azureedge.net` - Azure DevOps agent CDN location
  *  `*.queue.core.windows.net` - Worker queue for communicating with Managed DevOps Pools service
  *  `server.pipe.aria.microsoft.com` - Common client side telemetry solution (and used by the Agent Pool Validation extension among others)
  *  `azure.archive.ubuntu.com` - Provisioning Linux machines - this is HTTP, not HTTPS
  *  `www.microsoft.com` - Provisioning Linux machines
* Less secure, more open endpoints that our service depends on:
   * Needed by our service:
     * `packages.microsoft.com` - Provisioning Linux machines
     * `ppa.launchpad.net` - Provisioning Ubuntu machines
     * `dl.fedoraproject.org` - Provisioning certain Linux distros
   * Needed by Azure DevOps agent:
     * `dev.azure.com`
     * `*.services.visualstudio.com`
     * `*.vsblob.visualstudio.com`
     * `*.vssps.visualstudio.com`
     * `*.visualstudio.com`
     These entries are the minimum domains required. If you have any issues, see [Azure DevOps allowlist](/azure/devops/organizations/security/allow-list-ip-url) for the full list of domains required.

If you configure your Azure DevOps Pipeline to run inside of a container, you need to also allowlist the source of the container image (Docker or ACR).

## See also

* [Configure pool settings](./configure-pool-settings.md)
