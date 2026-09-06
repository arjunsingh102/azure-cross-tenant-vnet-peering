# Azure Cross-Tenant VNet Peering Troubleshooting

## Resolving "Remote Sync Required" and "Access Token Is From the Wrong Issuer"

This case study documents the troubleshooting of an Azure Virtual Network peering synchronization issue between VNets hosted in separate Microsoft Entra ID tenants and Azure subscriptions.

> **Security Notice**
>
> All resource names, tenant IDs, subscription IDs, IP addresses and account information used in this document are generic placeholders. No production environment information is included.

---

## Table of Contents

- [Architecture](#architecture)
- [Problem Statement](#problem-statement)
- [Symptoms](#symptoms)
- [Root Cause](#root-cause)
- [Cross-Tenant Identity Model](#cross-tenant-identity-model)
- [RBAC Configuration](#rbac-configuration)
- [Resolution](#resolution)
- [Azure CLI Synchronization](#azure-cli-synchronization)
- [Validation](#validation)
- [Troubleshooting Flow](#troubleshooting-flow)
- [Security Considerations](#security-considerations)
- [Key Lessons](#key-lessons)

---

## Architecture

The environment consisted of two Azure subscriptions associated with separate Microsoft Entra ID tenants.

```text
┌─────────────────────────────┐
│     Microsoft Entra ID      │
│          Tenant A           │
│                             │
│   Azure Subscription A      │
│                             │
│   ┌─────────────────────┐   │
│   │       VNET-A        │   │
│   │                     │   │
│   │ Application Subnets │   │
│   │ Platform Subnets    │   │
│   └──────────┬──────────┘   │
└──────────────┼──────────────┘
               │
               │ Cross-Tenant
               │ VNet Peering
               │
┌──────────────┼──────────────┐
│              │              │
│   ┌──────────▼──────────┐   │
│   │       VNET-B        │   │
│   │                     │   │
│   │ Application Subnets │   │
│   │ Platform Subnets    │   │
│   └─────────────────────┘   │
│                             │
│   Azure Subscription B      │
│                             │
│     Microsoft Entra ID      │
│          Tenant B           │
└─────────────────────────────┘
```

---

## Problem Statement

An existing cross-tenant VNet peering displayed:

```text
Peering State: Connected
Peering Sync Status: Remote sync required
```

Refreshing the Azure Portal did not clear the synchronization warning.

The peering itself existed and remained connected.

---

## Symptoms

When opening the VNet peering configuration and attempting remote-directory authentication, Azure returned an error similar to:

```text
The access token is from the wrong issuer.

It must match the tenant associated with this subscription.
```

The corresponding Azure Resource Manager request returned:

```text
HTTP 401 Unauthorized
```

The important observation was that the access token had been issued by one Microsoft Entra tenant while the target Azure subscription belonged to another tenant.

---

## Root Cause

The issue was caused by a **cross-tenant authentication context mismatch**.

Conceptually:

```text
Token Issuer
    |
    v
Tenant A
    |
    |       X
    +-----------> Subscription B
                   belongs to
                   Tenant B
```

Azure Resource Manager validates whether the token used for the management operation is valid for the tenant associated with the target subscription.

Therefore, even though the VNet peering remained connected, management operations such as synchronization could fail.

### Important distinction

```text
Peering Connected
        ≠
Peering Fully Synchronized
```

A connected data-plane path does not necessarily mean that every management-plane operation is succeeding.

---

## Cross-Tenant Identity Model

Microsoft Entra B2B guest identities were used to establish administrative access between the two tenants.

```text
TENANT A                              TENANT B
--------                              --------

Admin-A ----------------------------> Guest Admin-A
                                      |
                                      +-- Network Contributor
                                          on VNET-B


Guest Admin-B <---------------------- Admin-B
     |
     +-- Network Contributor
         on VNET-A
```

A B2B guest object can appear in the resource tenant in a format similar to:

```text
user_domain.com#EXT#@tenant.onmicrosoft.com
```

This is not a new account with an independent password.

The user continues to authenticate using their original **home tenant identity**.

---

## RBAC Configuration

The guest identity requires sufficient Azure RBAC permissions to manage the remote networking resource.

For VNet peering administration:

```text
Network Contributor
```

can be assigned at the required VNet scope.

Example scope:

```text
Subscription
   |
   +-- Resource Group
          |
          +-- Virtual Network
                 |
                 +-- Network Contributor
```

Where possible, assign access at the smallest practical scope.

Avoid assigning subscription-wide `Owner` or `Contributor` unless there is a genuine requirement.

---

## Resolution

### Step 1 - Verify Tenant Ownership

Identify which Microsoft Entra tenant owns each Azure subscription.

Conceptually:

```text
Subscription A -> Tenant A
Subscription B -> Tenant B
```

Do not assume that the currently selected Azure Portal directory owns the target subscription.

---

### Step 2 - Configure B2B Guest Access

Invite the required administrator from Tenant A into Tenant B.

Repeat in the opposite direction if administrators from both tenants require access.

In Microsoft Entra:

```text
Microsoft Entra ID
    -> Users
    -> New user
    -> Invite external user
```

---

### Step 3 - Redeem the Invitation

The guest invitation must be redeemed using the user's original home identity.

Do not attempt to authenticate directly using the generated `#EXT#` guest UPN.

The guest object does not normally have a separate guest password.

---

### Step 4 - Assign Azure RBAC

On the required remote VNet:

```text
Virtual Network
    -> Access control (IAM)
    -> Add role assignment
    -> Network Contributor
```

Assign the role to the guest identity.

---

### Step 5 - Authenticate Against the Correct Remote Directory

For the peering from VNET-A to VNET-B:

```text
Local VNet:       VNET-A
Remote VNet:      VNET-B
Remote Directory: Tenant B
```

For the reverse peering:

```text
Local VNet:       VNET-B
Remote VNet:      VNET-A
Remote Directory: Tenant A
```

After authentication, verify that Azure reports successful authentication and no longer returns the wrong-token-issuer error.

---

### Step 6 - Do Not Toggle Networking Settings to Force Save

Do not disable options simply to make the Azure Portal **Save** button active.

Settings such as:

```text
Allow virtual network access
Allow forwarded traffic
Allow gateway transit
Use remote gateways
```

can affect active network traffic.

A disabled Save button may simply mean that no configuration property has changed.

---

## Azure CLI Synchronization

If the Azure Portal continues reporting:

```text
Remote sync required
```

after correcting authentication, use Azure CLI to explicitly synchronize the peering.

### Login to the appropriate tenant

```bash
az logout

az login --tenant <TENANT-ID>
```

### Select the subscription

```bash
az account set \
  --subscription <SUBSCRIPTION-ID>
```

### Verify the current context

```bash
az account show --output table
```

Confirm the expected:

```text
Subscription
TenantId
```

before making changes.

### Synchronize the peering

```bash
az network vnet peering sync \
  --resource-group <RESOURCE-GROUP> \
  --vnet-name <VNET-NAME> \
  --name <PEERING-NAME>
```

Perform the corresponding validation/synchronization from the appropriate tenant and subscription context for the opposite side where required.

---

## Validation

After synchronization, verify the peering state.

```bash
az network vnet peering show \
  --resource-group <RESOURCE-GROUP> \
  --vnet-name <VNET-NAME> \
  --name <PEERING-NAME> \
  --output table
```

Also validate actual workload connectivity.

### Windows

```powershell
Test-NetConnection <REMOTE-PRIVATE-IP> -Port <APPLICATION-PORT>
```

### Linux

```bash
nc -zv <REMOTE-PRIVATE-IP> <APPLICATION-PORT>
```

Do not rely exclusively on ICMP/ping because ICMP may be intentionally blocked.

Also validate, where applicable:

- NSG rules
- User Defined Routes
- Azure Firewall/NVA routing
- Effective routes
- DNS resolution
- Application ports
- Gateway transit configuration

---

## Troubleshooting Flow

```text
                VNet Peering Warning
                         |
                         v
                Check Peering State
                         |
               +---------+---------+
               |                   |
          Disconnected          Connected
               |                   |
               v                   v
       Troubleshoot basic     Check Sync Status
          peering                 |
                                   v
                         Remote Sync Required?
                                   |
                                  Yes
                                   |
                                   v
                        Check Address Spaces
                                   |
                                   v
                     Authenticate Remote Tenant
                                   |
                                   v
                      HTTP 401 / Wrong Issuer?
                                   |
                                  Yes
                                   |
                                   v
                     Verify Subscription Tenant
                                   |
                                   v
                       Configure Entra B2B
                                   |
                                   v
                       Assign Required RBAC
                                   |
                                   v
                       Redeem Guest Invite
                                   |
                                   v
                  Authenticate Correct Directory
                                   |
                                   v
                         Synchronize Peering
                                   |
                                   v
                       Validate Connectivity
```

---

## Security Considerations

When implementing cross-tenant Azure networking:

1. Use Microsoft Entra B2B identities instead of shared administrative credentials.
2. Follow least-privilege RBAC principles.
3. Scope `Network Contributor` to the required resources where practical.
4. Review Conditional Access requirements for guest administrators.
5. Protect privileged accounts with MFA.
6. Review guest-user lifecycle and remove access when no longer required.
7. Never expose real tenant IDs, subscription IDs, internal IP addresses or production resource names in public documentation.
8. Validate network connectivity after changing peering configuration.

---

## Key Lessons

### 1. A networking warning may actually be an identity problem

The visible symptom was:

```text
Remote sync required
```

but the underlying issue was:

```text
HTTP 401
Access token is from the wrong issuer
```

---

### 2. Connected does not necessarily mean synchronized

```text
Peering State: Connected
```

and:

```text
Peering Sync Status: Fully Synchronized
```

represent different aspects of the configuration.

---

### 3. Tenant context matters

For cross-tenant Azure operations, always verify:

```bash
az account show
```

before performing changes.

---

### 4. B2B guests authenticate using their home identity

Do not treat the generated `#EXT#` identity as an independent user account with its own password.

---

### 5. Do not modify production networking settings just to force a Portal refresh

If Azure Portal does not enable **Save**, do not toggle routing or peering options unnecessarily.

Use the appropriate synchronization operation instead.

---

## Final Resolution Pattern

```text
Remote Sync Required
        |
        v
Wrong Token Issuer / HTTP 401
        |
        v
Identify Correct Tenant
        |
        v
Microsoft Entra B2B Guest
        |
        v
Network Contributor RBAC
        |
        v
Redeem Invitation
        |
        v
Authenticate Remote Directory
        |
        v
Synchronize VNet Peering
        |
        v
Validate Connectivity
```

---

## Technologies Used

- Microsoft Azure
- Azure Virtual Network
- VNet Peering
- Microsoft Entra ID
- Microsoft Entra B2B
- Azure RBAC
- Azure CLI
- Azure Resource Manager

---

## Disclaimer

This repository documents a generalized technical troubleshooting scenario.

All tenant names, subscription IDs, account names, resource groups, VNet names, IP addresses and other environment-specific information have been anonymized.

---

## Author

Cloud Infrastructure / Azure Engineering

If this troubleshooting guide helped you, feel free to ⭐ the repository.
