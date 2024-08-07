---
title: "Tiering Entra roles and application permissions based on known attack paths"
catchphrase: "An attempt to better understand the security implications of cloud administrative assets."
image: "/assets/images/blog/2024-08-07-introducing-cloud-tier-models-based-on-known-attack-paths/00_introducing_cloud_tier_models_based_on_known_attack_paths.png"
last_modified_at: 2024-08-07
tags:
  - Administrative tiering
  - Tier model
  - Microsoft Graph
  - Entra ID
  - Azure
  - Pentesting
  - Red-teaming
---

## Introduction

[![](/assets/images/blog/2024-08-07-introducing-cloud-tier-models-based-on-known-attack-paths/01_screenshot.png)](/assets/images/blog/2024-08-07-introducing-cloud-tier-models-based-on-known-attack-paths/01_screenshot.png)

I have recently released on my [GitHub](https://github.com/emiliensocchi/azure-tiering) a new kind of cloud tier model to categorize **Entra roles** and **application permissions** in MS Graph **based on known attack paths**. The goal is to provide a clear understanding of the security implications of each role and permission, so that red- and blue-teamers can understand more easily to which extent those can be abused.


## Motivation

In 2020, Microsoft introduced the [Enterprise Access Model]( https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model) as a new approach to tiered administration in Azure and Entra ID.

After witnessing several attempts at implementing the EAM, I kept asking myself the same question: what are the actual security implications of the roles and permissions categorized as Tier-0? In other words, what is a threat actor really able to do with them?
I am aware the EAM has different definition of Tier-0, but coming from a red-team perspective, I always interpreted the meaning of Tier-0 as “potential for becoming Global Admin”, or at least not far from it. 

Most companies I have witnessed trying to implement the EAM have used [Thomas Naunheim](https://x.com/Thomas_Live)’s awesome [AzurePrivilegedIAM project]( https://github.com/Cloud-Architekt/AzurePrivilegedIAM). To my knowledge, this is the only project providing tangible lists of tiered roles and permissions, which makes the EAM a lot more concrete and easier to implement.

After starting to dig into the Tier-0 classification of the EAM, I started noticing that some of those assets had very different security implications. For example, [Tier-0 application permissions]( https://raw.githubusercontent.com/Cloud-Architekt/AzurePrivilegedIAM/main/Classification/Classification_AppRoles.json) include permissions such as the following:

| Application permission | Description |
|---|---|
| [DirectoryRecommendations.ReadWrite.All]( https://learn.microsoft.com/en-us/graph/permissions-reference#directoryrecommendationsreadwriteall) | Allows reading and updating all Microsoft Entra recommendations. |
| [RoleManagement.ReadWrite.Directory]( https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementreadwritedirectory) | Allows reading and managing the role-based access control (RBAC) settings for your company's directory. This includes instantiating directory roles and managing directory role membership, and reading directory role templates, directory roles and memberships. |

In case a Threat Actor (TA) compromises a service principal (SP) with the first application permission, a worst-case scenario is that the TA is able to dismiss important security recommendations from Entra ID. With the second permission, a TA would be able to assign the Global Administrator role to the SP they have compromised and escalate to Global Admin. Despite both belonging to Tier-0 in the EAM, the security implications of those permissions are *very* different, as the first one has no real impact on the tenant’s confidentiality, integrity or availability, while the second one leads to full tenant takeover. 

The EAM also classifies [Entra roles]( https://github.com/Cloud-Architekt/AzurePrivilegedIAM/blob/main/Classification/Classification_EntraIdDirectoryRoles.json#L4) with very different security implications as Tier-0. Here are a couple of examples:

| Entra role | Description |
|---|---|
| [Security Reader]( https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#security-reader) | Users with this role have global read-only access on security-related feature, including all information in Microsoft 365 Defender portal, Microsoft Entra ID Protection, Privileged Identity Management, as well as the ability to read Microsoft Entra sign-in reports and audit logs, and in Microsoft Purview compliance portal. |
| [Global Administrator]( https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#global-administrator) | Self-explanatory. |

With the first role, a TA would be able to read security-related information in the tenant as a worst-case scenario. Although this impacts confidentiality to a certain extent, we are far from a risk of escalating to Global Admin or any similar scenario. With the second role however, the security implications are self-explanatory, as a TA who has compromised an identity with the Global Admin role is able to take over the entire tenant. 

We can see that the EAM (in its current form at least) provides little visibility into the actual security implications of its roles/permissions in terms of impact, and especially in terms of risks for privilege escalation. The consequence is dual for most companies. On one hand, SecOps teams  may use unnecessary resources on controlling the use of certain roles and permissions that should not be classified as Tier-0 in the first place. On the other hand, blue teams may have a hard time understanding the actual blast radius of Tier-0 assets, making it hard to foresee what a TA is trying to achieve during an incident.


## Introducing: tier models based on known attack paths

In an attempt to better understand the security implications of cloud administrative assets, the idea of this project is to categorize roles and permissions, based on known attack paths. For those unfamiliar with the concept, attack paths are a way to document the steps necessary to escalate privileges from one asset to another. 

### Objective 1: provide a better understanding of security implications

By categorizing roles and permissions based on known attack paths, the objective is to provide a better understanding of what a threat actor with those administrative permissions can do in terms of privilege escalation, while providing a better understanding of the security implications of those assets.

### Objective 2: provide a base for further development

The definition of a "tier" and its content is highly dependent on the business requirements, risk appetite and governance strategy of a company. The second objective of this project is to provide a base that can be adapted to develop company-specific tier models based on the same philosophy, but answering different requirements. 


## Approach

The baseline approach for this project is to rely on what a role or permission is **effectively** capable of doing.

### Entra roles: do not to rely on role actions

Entra roles are made of what Microsoft refers to as “role actions”. For those unfamiliar with role actions, they consist of a list of operations within certain Microsoft services that a role supposedly allows to perform. 

Here are a few examples of Entra role actions (see [here]( https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference) for more information):

- `microsoft.directory/users/password/update`
-  `microsoft.office365.protectionCenter/attackSimulator/payload/allProperties/allTasks`
- `microsoft.azure.supportTickets/allEntities/allTasks`

The main issue with role actions is that they are not sufficient to be certain of what an Entra role is effectively capable of doing, as additional access control is sometimes implicitly applied on top of them. For example, the `microsoft.directory/users/password/update` action seems to allow Entra roles with that action to reset passwords for any user. However, the effective permissions provided by that action are enforced server side, based on the Entra role using the action. 

For example, both the [Authentication Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#authentication-administrator) and [Privileged Authentication Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#privileged-authentication-administrator) role contain the `microsoft.directory/users/password/update` action. However, an [Authentication Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#authentication-administrator) can only reset the password of certain users within a directory, whereas a [Privileged Authentication Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#privileged-authentication-administrator) can reset the password of any user, without additional constrains enforced server side ([more information](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/privileged-roles-permissions?tabs=admin-center#who-can-reset-passwords)). This example illustrates the unreliability of an approach based on role actions, as it does not provide a clear understanding of the effective capabilities of an Entra role.

### App permissions: do not rely on permission names or documentation

The [official documentation](https://learn.microsoft.com/en-us/graph/permissions-reference) for MS Graph application permissions is extremely vague regarding their capabilities, as the name of a permission is supposed to be self-sufficient to understand its access within Graph, based on the following format: 

- `[Resource_in_Graph].[Permission(s)].([Optional_Scope])`

Here are few examples of application permissions illustrating that naming convention:

- [`Application.Read.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#applicationreadall)
- [`Application.ReadWrite.OwnedBy`](https://learn.microsoft.com/en-us/graph/permissions-reference#applicationreadwriteownedby)
- [`Directory.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#directoryreadwriteall)
- [`Contacts.Read`](https://learn.microsoft.com/en-us/graph/permissions-reference#contactsread)

However, similar to Entra role actions, the name of an application permission is not sufficient to be certain of what it is effectively capable of doing, as additional access control is sometimes applied on top of it server side. For example, [`Directory.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#directoryreadwriteall) indicates that whoever holds that permission should have full read and write permissions to all directory objects within the Entra ID directory. However, as [Andy Robbins](https://x.com/_wald0) has already documented in a comprehensive [blog post](https://posts.specterops.io/directory-readwrite-all-is-not-as-powerful-as-you-might-think-c5b09a8f78a8) about the subject, the [`Directory.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#directoryreadwriteall) permission does **not** provide write access to the entire directory, as its name suggests. This example illustrates how misleading the name of an application permission can be, making it unreliable as a source for categorizing permissions into tiers.

### Rely on testing the <u>effective</u> capabilities of roles and permissions

Based on those observations, the only reliable approach to understanding the effective capabilities of a role/permission is to verify what it can effectively do through testing. The goal is not to re-invent the wheel, but to provide a clear understanding of the security implications of each role and permission, so that red- and blue-teamers can understand more easily to which extent those can be abused.


## Initial release

The initial release for this project attempts to categorize **Entra roles** and **application permissions** in Microsoft Graph based on manually testing attack paths, as well as reviewing public research documentating privilege escalation for some of those administrative assets.

### Defining tiers

**Tier 0** contains assets with at least one known technique to create a path to Global Admin. This does **not** mean a path necessarily exist in every tenant, as privilege escalations are often tenant specific, but the goal is to identify roles and permissions with a **risk** of having a path to Global Admin. 

**Tier 1** contains administrative assets with limited write access, and *without* any known path to Global Admin. In case a new path is discovered for a Tier-1 asset, the latter is automatically bumped to Tier-0. 

**Tier 2** contains remaining roles and permissions with little to no security implications.

### Documenting shortest path to Global Admin

Tier-0 assets contain descriptive information about known shortest paths to Global Admin, in order to demonstrate the risk of a role or permission. It is important to note that the shortest paths documented are not necessarily the most common or the only ones. In many case, a Tier-0 role or permission has **many** paths to Global Admin. 

Furthermore, this does **not** mean there exists a path in every tenant, as privilege escalations are often tenant specific, dependent on the structure of the tenant, the services enabled, etc. The goal is to identify Tier-0 roles and permissions that have a **risk** of having a path to Global Admin.

### Standardizing Tier-0 information

In both models, Tier-0 assets are documented based on the same standard, using the following pieces of information:

| Role name / Application permission | Path type | Known shortest path | Example |
|---|---|---|---|
| Name of the Entra role / MS Graph permission | "Direct" means the escalation requires a single step to become Global Admin. "Indirect" means the privilege escalation requires two or more steps. | One of the shortest paths possible to Global Admin that is known with the application permission. <br> It does **not** mean this is the most common or only possible path. In most cases, a large number of paths are possible, but the idea is to document one of the shortest to demonstrate the risk. | A concrete high-level example with a Threat Actor (TA), illustrating the "Known shortest path". |

Here is a sample from the Entra role tier model:

| Entra role | Path type | Known shortest path | Example |
|---|---|---|---|
| [Application Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#application-administrator) <a id='application-admin'></a> | Indirect | Can impersonate any SP with privileged application permissions granted for Microsoft Graph, which can be abused to become Global Admin. | TA identifies an SP with the [`RoleManagement.ReadWrite.Directory`](https://github.com/emiliensocchi/azure-tiering/tree/main/Microsoft%20Graph%20application%20permissions#rolemanagement-readwrite-directory) application permission. TA creates a new secret for the SP, impersonates it and abuses the granted permissions to escalate to Global Admin. |
| [Cloud Application Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#cloud-application-administrator) | Indirect | Same as [Application Administrator](#application-admin). | Same as [Application Administrator](#application-admin). |
| [Directory Synchronization Accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#directory-synchronization-accounts) | Indirect | Same as [Application Administrator](#application-admin). <br> As of September 2023, *cannot* reset the password of cloud-only users via the synchronization API to take over break-glass accounts. | Same as [Application Administrator](#application-admin). |
| [On Premises Directory Sync Account](https://graph.microsoft.com/v1.0/directoryRoleTemplates/a92aed5d-d78a-4d16-b381-09adb37eb3b0) | n/a | ⚠️ *Untested! Needs more research.* <br> Note: does not seem to be able to create new SP credentials or give consent. | n/a |
| [Partner Tier2 Support](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#partner-tier2-support) <a id='partner-tier2'></a> | Direct | Can reset the password of a [break-glass account](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access) and take it over. | TA resets the password of a [break-glass account](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access) and authenticates as Global Admin. <br>Note: many other paths are possible. |
| [Privileged Role Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#privileged-role-administrator) | Direct | Can assign the Global Admin role to itself. | TA assigns the Global Admin role to the compromised user account, and authenticates as Global Admin. |

Here is a sample from the tier model for MS Graph application permissions:

| Application permission | Path type | Known shortest path | Example |
|---|---|---|---|
| [`Application.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#applicationreadwriteall) <a id='application-readwrite-all'></a> | Indirect | Can impersonate any SP with more privileged application permissions granted for MS Graph, and impersonate it to escalate to Global Admin. | TA identifies an SP with the [`RoleManagement.ReadWrite.Directory`](#rolemanagement-readwrite-directory) application permission. TA creates a new secret for the SP, impersonates it and follows the same path as that permission to escalate to Global Admin. |
| [`AppRoleAssignment.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#approleassignmentreadwriteall) | Indirect | Can assign the [`RoleManagement.ReadWrite.Directory`](#rolemanagement-readwrite-directory) permission to the compromised SP *without* requiring admin consent, and escalate to Global Admin. | TA assigns the [`RoleManagement.ReadWrite.Directory`](#rolemanagement-readwrite-directory) permission to the compromised SP and follows the same path as that permission to escalate to Global Admin. |
| [`EntitlementManagement.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#entitlementmanagementreadwriteall) | Indirect | Can update the assignment policy of an access package provisioning access to Global Admin, so that requesting the package without approval is possible from a controlled user account. | TA identifies an access package providing access to a security group with an active Global Admin assignment. TA adds an assignment policy to the access package, so that the latter can be requested from a controlled user account, without manual approval. TA requests the access package and escalates to Global Admin via group membership. |
| [`Policy.ReadWrite.AuthenticationMethod`](https://learn.microsoft.com/en-us/graph/permissions-reference#policyreadwriteauthenticationmethod) <a id='policy-readwrite-authenticationmethod'></a> | Indirect | When combined with [`UserAuthenticationMethod.ReadWrite.All`](#userauthenticationmethod-readwrite-all), can enable the [Temporary Access Pass (TAP)](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-authentication-temporary-access-pass) authentication method to help leveraging and follow the same path as that permission. | TA enables the TAP authentication method for the whole tenant and follows the same path as [`UserAuthenticationMethod.ReadWrite.All`](#userauthenticationmethod-readwrite-all). |
| [`RoleManagement.ReadWrite.Directory`](https://learn.microsoft.com/en-us/graph/permissions-reference#rolemanagementreadwritedirectory) <a id='rolemanagement-readwrite-directory'></a> | Direct | Can assign the Global Admin role to a controlled principal. | TA assigns the Global Admin role to the compromised SP, re-authenticates with the SP and escalates to Global Admin. |
| [`Synchronization.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#synchronizationreadwriteall) | n/a | ⚠️ *Untested! Needs more research.* <br> Note: does not seem to be able to create new SP credentials. | n/a |
| [`UserAuthenticationMethod.ReadWrite.All`](https://learn.microsoft.com/en-us/graph/permissions-reference#userauthenticationmethodreadwriteall) <a id='userauthenticationmethod-readwrite-all'></a> | Direct | Can generate a [Temporary Access Pass (TAP)](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-authentication-temporary-access-pass) and take over any user account in the tenant. <br> Note: if TAP is not an enabled authentication method in the tenant, this path needs to be combined with [`Policy.ReadWrite.AuthenticationMethod`](#policy-readwrite-authenticationmethod) to be successful. | TA creates a TAP for a [break-glass account](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access), authenticates with the TAP instead of the account's password and escalates to Global Admin. | 

Note that other Tiers may have a different formats in their respective models, depending on their exact definitions. 

For convenience when browsing the model without authenticated access to an Entra tenant, most roles and permissions have a hyperlink to their definition in the Microsoft documentation. However, for those available in MS Graph, but without an entry in the documentation, they are hyperlinked directly to their definition object in the Graph API for completeness.

Furthermore, note that some roles/permissions are marked with ”⚠️ *Untested! Needs more research.*”. Those are suspicious administrative assets without published research and that I have not tested personally. Some of them might have notes with observations I have made during basic testing, but those assets require more research to have a comprehensive understanding of their capabilities and be tiered properly. Until this is the case, they will remain categorized as Tier-0 for safety.

## Current limitations

### Manual discovery of privilege escalations

As previously mentioned, the tiering of Entra roles and application permissions is based on **known** techniques for privilege escalations. Obviously, this makes the models only as good as the research that has been conducted around those assets. I have conducted a fair amount of research myself while categorizing those assets, but the model definitely has room for improvement. I hope that a community-based effort can help improving and maintaining the overall quality over time, as assets evolve and new privilege escalations are discovered.

For companies using the presented tier models as a base for developing their own tiering, they should be aware that **any custom role or application permission** they have defined themselves (technically called “application roles” if the latter are custom) **should be tested and tiered manually**.

### Evolution of role and permission capabilities over time

Over time, the set of existing roles and permissions is almost certain to evolve, as Microsoft is likely to add and remove some of them with the evolution of the Entra ID platform. Each addition will require testing for privilege escalations, while removals will require updating the relevant tier model accordingly. Additionally, new capabilities may theoretically be added silently to existing roles and permissions, which may modify the tiering of an asset without warning.

The project uses some level of automation to detect the addition of new roles and permissions by Microsoft, using a similar approach to [AzRoleWatcher](https://github.com/emiliensocchi/az-role-watcher). New additions are reviewed and added to the relevant models as soon as possible, but this is where the automation currently stops.

## Future work

### Automated tiering via continuous testing 

In the long term, I would like to tackle the limitations of this project through full automation. The idea is to make a library of modules for Tier-0 operations, run every single role/permission through those modules in a dedicated tenant every night, and verify their effective capabilities continuously. Here are a few examples of what Tier-0 operations could be: 

- Creating a new credential for an existing service principal
- Resetting the password of a user assigned a Tier-0 Entra role
- Assigning a Tier-0 Entra role to a security principal

Thanks to this automated approach, silent updates of roles and permissions could be detected on the spot, regardless of the information provided through documentation, namespaces or role actions. Additionally, custom roles and permissions defined by enterprise organizations would be automatically tested once the project is imported to their tenant.

When and how this project will happen is unclear for now, as I am about to go on a break for the rest of the year. However, leveraging an existing tool such as [Andy Robbins](https://x.com/_wald0)’ [BloodHound Attack Research Kit (BARK)](https://github.com/BloodHoundAD/BARK) or similar is probably the approach that will be favored at some point. The core of the project lies mostly in setting up the automation and develop a complete set of Tier-0 operations.

### Tiering other cloud assets based on known attack paths

This first release contains tier models for 2 sets of administrative assets:

- Entra roles
- MS Graph application permissions

In the long term, I plan on developing similar tier models for the following assets, as I work with them daily:

- [Azure roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Cloud Identity roles](https://cloud.google.com/identity)
- [GCP roles](https://cloud.google.com/iam/docs/understanding-roles)

Note that I do not plan on including anything related to AWS, as I simply do not work with the platform.


## Conclusion

The presented project is a humble attempt at creating a tier model based on attack paths, to understand the security implication of administrative assets in MS Graph, Entra ID and later Azure. The project does not mean to replace the EAM, but rather propose a different approach to tiering administrative assets.

It is important to keep in mind that the current tiering is based on my own understanding of attack paths within MS Graph and Entra ID. It is therefore highly possible that some assets are categorized inappropriately, but I hope that a community-driven approach can reduce the amount of miscategorized assets and enhance research around them.
