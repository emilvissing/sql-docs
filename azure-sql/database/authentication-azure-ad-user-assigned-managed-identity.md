---
title: User-assigned managed identity in Azure AD for Azure SQL
description: User-assigned managed identities (UMI) in Azure AD (AD) for Azure SQL Database and SQL Managed Instance.
titleSuffix: Azure SQL Database & Azure SQL Managed Instance
ms.service: sql-db-mi
ms.subservice: security
ms.topic: conceptual
author: GithubMirek
ms.author: mireks
ms.reviewer: vanto
ms.date: 06/30/2022
monikerRange: "= azuresql || = azuresql-db || = azuresql-mi"
---

# User-assigned managed identity in Azure AD for Azure SQL

[!INCLUDE[appliesto-sqldb-sqlmi](../includes/appliesto-sqldb-sqlmi.md)]

Azure Active Directory (AD) supports two types of managed identities: System-assigned managed identity (SMI) and user-assigned managed identity (UMI). For more information, see [Managed identity types](/azure/active-directory/managed-identities-azure-resources/overview#managed-identity-types).

A system-assigned managed identity is automatically assigned to a managed instance when it is created. When using Azure AD authentication with Azure SQL Managed Instance, a managed identity must be assigned to the server identity. Previously, only a system-assigned managed identity could be assigned to the Managed Instance or SQL Database server identity. With support for user-assigned managed identity, the UMI can be assigned to Azure SQL Managed Instance or Azure SQL Database as the instance or server identity. This feature is now supported for SQL Database.

In addition to using UMI and SMI as the server or instance identity, the UMI and SMI can be used to access the database using the SQL connection string option `Authentication=Active Directory Managed Identity`. For more information, see [Using Azure Active Directory authentication with SqlClient](/sql/connect/ado-net/sql/azure-active-directory-authentication).
There will need to be a SQL user mapped to the managed identity in the target database.

## Benefits of using user-assigned managed identities

There are several benefits of using UMI as a server identity.

- User flexibility to create and maintain their own user-assigned managed identities for a given tenant. UMI can be used as server identities for Azure SQL. UMI is managed by the user, compared to an SMI, which identity is uniquely defined per server, and assigned by the system.
- In the past, the Azure AD [Directory Readers](authentication-aad-directory-readers-role.md) role was required when using SMI as the server or instance identity. With the introduction of accessing Azure AD using [Microsoft Graph](/graph/api/resources/azure-ad-overview), users concerned with giving high-level permissions such as the Directory Readers role to the SMI or UMI can alternatively give lower-level permissions so that the server or instance identity can access [Microsoft Graph](/graph/api/resources/azure-ad-overview). For more information on providing Directory Readers permissions and it's function, see [Directory Readers role in Azure Active Directory for Azure SQL](authentication-aad-directory-readers-role.md).
- Users can choose a specific UMI to be the server or instance identity for all SQL Databases or Managed Instances in the tenant, or have multiple UMIs assigned to different servers or instances. For example, different UMIs can be used in different servers representing different features. For example, a UMI serving transparent data encryption in one server, and a UMI serving Azure AD authentication in another server.
- UMI is needed to create an Azure SQL logical server configured with transparent data encryption (TDE) with customer-managed keys (CMK). For more information, see [Customer-managed transparent data encryption using user-assigned managed identity](transparent-data-encryption-byok-identity.md).
- User-assigned managed identities are independent from logical servers or managed instances. When a logical server or instance is deleted, the system-assigned managed identity is deleted as well. User-assigned managed identities aren't deleted with the server.

> [!NOTE]
> The instance identity (SMI or UMI) must be enabled to allow support for Azure AD authentication in Managed Instance. For SQL Database, enabling the server identity is optional and required only if an Azure AD service principal (Azure AD application) oversees creating and managing Azure AD users, groups, or application in the server. For more information, see [Azure Active Directory service principal with Azure SQL](authentication-aad-service-principal.md).

## Creating a user-assigned managed identity

For information on how to create a user-assigned managed identity, see [Manage user-assigned managed identities](/azure/active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities).

## Permissions

Once the UMI is created, some permissions are needed to allow the UMI to read from [Microsoft Graph](/graph/api/resources/azure-ad-overview) as the server identity. Grant the permissions below, or give the UMI the [Directory Readers](authentication-aad-directory-readers-role-tutorial.md) role. These permissions should be granted before provisioning an Azure SQL logical server or managed instance. Once the permissions are granted to the UMI, they're enabled for all servers or instances that are created with the UMI assigned as a server identity.

> [!IMPORTANT]
> Only a [Global Administrator](/azure/active-directory/roles/permissions-reference#global-administrator) or [Privileged Role Administrator](/azure/active-directory/roles/permissions-reference#privileged-role-administrator) can grant these permissions.

- [**User.Read.All**](/graph/permissions-reference#user-permissions) - allows access to Azure AD user information
- [**GroupMember.Read.All**](/graph/permissions-reference#group-permissions) – allows access to Azure AD group information
- [**Application.Read.ALL**](/graph/permissions-reference#application-resource-permissions) – allows access to Azure AD service principal (applications) information

### Grant permissions

The following is a sample PowerShell script that will grant the necessary permissions for UMI or SMI. This sample will assign permissions to the UMI `umiservertest`. To execute the script, you must sign in as a user with a "Global Administrator" or "Privileged Role Administrator" role, and have the following [Microsoft Graph permissions](/graph/auth/auth-concepts#microsoft-graph-permissions): 
- User.Read.All
- GroupMember.Read.All
- Application.Read.ALL

```powershell
# Script to assign permissions to the UMI "umiservertest"

import-module AzureAD
$tenantId = '<tenantId>' # Your Azure AD tenant ID

Connect-AzureAD -TenantID $tenantId
# Login as a user with a "Global Administrator" or "Privileged Role Administrator" role
# Script to assign permissions to existing UMI 
# The following Microsoft Graph permissions are required: 
#   User.Read.All
#   GroupMember.Read.All
#   Application.Read.ALL

# Search for Microsoft Graph
$AAD_SP = Get-AzureADServicePrincipal -SearchString "Microsoft Graph";
$AAD_SP
# Use Microsoft Graph; in this example, this is the first element $AAD_SP[0]

#Output

#ObjectId                             AppId                                DisplayName
#--------                             -----                                -----------
#47d73278-e43c-4cc2-a606-c500b66883ef 00000003-0000-0000-c000-000000000000 Microsoft Graph
#44e2d3f6-97c3-4bc7-9ccd-e26746638b6d 0bf30f3b-4a52-48df-9a82-234910c4a086 Microsoft Graph #Change 

$MSIName = "<managedIdentity>";  # Name of your user-assigned or system-assigned managed identity
$MSI = Get-AzureADServicePrincipal -SearchString $MSIName 
if($MSI.Count -gt 1)
{ 
Write-Output "More than 1 principal found, please find your principal and copy the right object ID. Now use the syntax $MSI = Get-AzureADServicePrincipal -ObjectId <your_object_id>"

# Choose the right UMI or SMI

Exit
} 

# If you have more UMIs with similar names, you have to use the proper $MSI[ ]array number

# Assign the app roles

$AAD_AppRole = $AAD_SP.AppRoles | Where-Object {$_.Value -eq "User.Read.All"}
New-AzureADServiceAppRoleAssignment -ObjectId $MSI.ObjectId  -PrincipalId $MSI.ObjectId  -ResourceId $AAD_SP.ObjectId[0]  -Id $AAD_AppRole.Id 
$AAD_AppRole = $AAD_SP.AppRoles | Where-Object {$_.Value -eq "GroupMember.Read.All"}
New-AzureADServiceAppRoleAssignment -ObjectId $MSI.ObjectId  -PrincipalId $MSI.ObjectId  -ResourceId $AAD_SP.ObjectId[0]  -Id $AAD_AppRole.Id
$AAD_AppRole = $AAD_SP.AppRoles | Where-Object {$_.Value -eq "Application.Read.All"}
New-AzureADServiceAppRoleAssignment -ObjectId $MSI.ObjectId  -PrincipalId $MSI.ObjectId  -ResourceId $AAD_SP.ObjectId[0]  -Id $AAD_AppRole.Id
```

In the final steps of the script, if you have more UMIs with similar names, you have to use the proper `$MSI[ ]array` number, for example, `$AAD_SP.ObjectId[0]`.

### Check permissions for user-assigned manage identity

To check permissions for a UMI, go to the [Azure portal](https://portal.azure.com). In the **Azure Active Directory** resource, go to **Enterprise applications**. Select **All Applications** for the **Application type**, and search for the UMI that was created.

:::image type="content" source="media/authentication-azure-ad-user-assigned-managed-identity/azure-ad-search-enterprise-applications.png" alt-text="Screenshot of Azure portal Enterprise application settings":::

Select the UMI, and go to the **Permissions** settings under **Security**.

:::image type="content" source="media/authentication-azure-ad-user-assigned-managed-identity/azure-ad-check-user-assigned-managed-identity-permissions.png" alt-text="Screenshot of user-assigned managed identity permissions":::

## Managing a managed identity for a server or instance

To create an Azure SQL logical server with a user-assigned managed identity, see the following guide: [Create an Azure SQL logical server using a user-assigned managed identity](authentication-azure-ad-user-assigned-managed-identity-create-server.md)

### Set managed identities in the Azure portal

To set the identity for the SQL server or SQL managed instance in the [Azure portal](https://portal.azure.com):

1. Go to your **SQL server** or **SQL managed instance** resource. 
1. Under **Security**, select the **Identity** setting. 
1. Under **User assigned managed identity**, select **Add**. 
1. Select the desired **Subscription** and then under **User assigned managed identities** select the desired user assigned managed identity from the selected subscription. Then select the **Select** button. 

:::image type="content" source="media/authentication-azure-ad-user-assigned-managed-identity/existing-server-select-managed-identity.png" alt-text="Azure portal screenshot of user assigned managed identity when configuring existing server identity":::

### Create or set a managed identity using the Azure CLI

The Azure CLI 2.26.0 (or higher) is required to run these commands with UMI.

#### Azure SQL Database

- To provision a new server with UMI, use the [az sql server create](/cli/azure/sql/server#az-sql-server-create) command.
- To obtain the UMI server information, use the [az sql server show](/cli/azure/sql/server#az-sql-server-show) command. 
- To update the UMI server setting, use the [az sql server update](/cli/azure/sql/server#az-sql-server-update) command.

#### Azure SQL Managed Instance

- To provision a new managed instance with UMI, use the [az sql mi create](/cli/azure/sql/mi#az-sql-mi-create) command.
- To obtain the UMI managed instance information, use the [az sql server show](/cli/azure/sql/mi#az-sql-mi-show) command.
- To update the UMI managed instance setting, use the [az sql mi update](/cli/azure/sql/mi#az-sql-mi-update) command.

### Create or set a managed identity using PowerShell

[Az.Sql module 3.4](https://www.powershellgallery.com/packages/Az.Sql/3.4.0) or greater is required when using PowerShell with UMI.

#### Azure SQL Database

- To provision a new server with UMI, use the [New-AzSqlServer](/powershell/module/az.sql/new-azsqlserver) command.
- To obtain the UMI server information, use the [Get-AzSqlServer](/powershell/module/az.sql/get-azsqlserver) command.
- To update the UMI server setting, use the [Set-AzSqlServer](/powershell/module/az.sql/set-azsqlserver) command.

#### Azure SQL Managed Instance

- To provision a new managed instance with UMI, use the [New-AzSqlInstance](/powershell/module/az.sql/new-azsqlinstance) command.
- To obtain the UMI managed instance information, use the [Get-AzSqlInstance](/powershell/module/az.sql/get-azsqlinstance) command.
- To update the UMI managed instance setting, use the [Set-AzSqlInstance](/powershell/module/az.sql/set-azsqlinstance) command.

### Create or set a managed identity using REST API

The REST API provisioning script used in [Creating an Azure SQL logical server using a user-assigned managed identity](authentication-azure-ad-user-assigned-managed-identity-create-server.md) or [Create an Azure SQL Managed Instance with a user-assigned managed identity](../managed-instance/authentication-azure-ad-user-assigned-managed-identity-create-managed-instance.md) can also be used to update the UMI settings for the server. Rerun the provisioning command in the guide with the updated user-assigned managed identity property that you want to update.

### Create or set a managed identity using an ARM template

The ARM template used in [Creating an Azure SQL logical server using a user-assigned managed identity](authentication-azure-ad-user-assigned-managed-identity-create-server.md) or [Create an Azure SQL Managed Instance with a user-assigned managed identity](../managed-instance/authentication-azure-ad-user-assigned-managed-identity-create-managed-instance.md) can also be used to update the UMI settings for the server. Rerun the provisioning command in the guide with the updated user-assigned managed identity property that you want to update.

> [!NOTE]
> You can't change the SQL server administrator or password, nor the Azure AD admin by re-running the provisioning command for the ARM template.

## Limitations and known issues

- After a Managed Instance is created, the **Active Directory admin** blade in the Azure portal shows a warning: `Managed Instance needs permissions to access Azure Active Directory. Click here to grant "Read" permissions to your Managed Instance.` If the user-assigned managed identity was given the appropriate permissions discussed in the above [Permissions](#permissions) section, this warning can be ignored.
- If a system-assigned or user-assigned managed identity is used as the server or instance identity, deleting the identity will result in the server or instance inability to access Microsoft Graph. Azure AD authentication and other functions will fail. To restore Azure AD functionality, a new SMI or UMI must be assigned to the server with appropriate permissions.
- Permissions to access Microsoft Graph using UMI or SMI can only be granted using PowerShell. These permissions can't be granted using the Azure portal.

## Next steps

> [!div class="nextstepaction"]
> [Create an Azure SQL logical server using a user-assigned managed identity](authentication-azure-ad-user-assigned-managed-identity-create-server.md)

> [!div class="nextstepaction"]
> [Create an Azure SQL Managed Instance with a user-assigned managed identity](../managed-instance/authentication-azure-ad-user-assigned-managed-identity-create-managed-instance.md)
