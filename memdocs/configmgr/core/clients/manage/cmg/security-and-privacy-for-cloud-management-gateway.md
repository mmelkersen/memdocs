---
title: CMG security and privacy
titleSuffix: Configuration Manager
description: Learn about guidance and recommendations for security and privacy with the cloud management gateway.
author: aczechowski
ms.author: aaroncz
manager: dougeby
ms.date: 04/13/2021
ms.topic: conceptual
ms.prod: configuration-manager
ms.technology: configmgr-client
ms.assetid: 7304730b-b517-4c76-aadd-4cbd157dc971
---

# Security and privacy for the cloud management gateway

*Applies to: Configuration Manager (current branch)*

This article includes security and privacy information for the Configuration Manager cloud management gateway (CMG). For more information, see [Overview of cloud management gateway](overview.md).

## CMG security details

The CMG accepts and manages connections from CMG connection points. It uses mutual authentication using certificates and connection IDs.

The CMG accepts and forwards client requests using the following methods:

- Pre-authenticates connections using mutual HTTPS with the PKI-based client authentication certificate or Azure AD.

  - IIS on the CMG VM instances verifies the certificate path based on the trusted root certificates that you upload to the CMG.

  - If you enable certificate revocation, IIS on the VM instance also verifies client certificate revocation. For more information, see [Publish the certificate revocation list](#bkmk_crl).

- The certificate trust list (CTL) checks the root of the client authentication certificate. It also does the same validation as the management point for the client. For more information, see [Review entries in the site's certificate trust list](#bkmk_ctl).

- Validates and filters client requests (URLs) to check if any CMG connection point can service the request.  

- Checks content length for each publishing endpoint.

- Uses round-robin behavior to load-balance CMG connection points in the same site.

The CMG connection point uses the following methods:

- Builds consistent HTTPS/TCP connections to all VM instances of the CMG. It checks and maintains these connections every minute.

- Uses mutual authentication with the CMG using certificates.

- Forwards client requests based on URL mappings.

- Reports connection status to show service health status in the console.

- Reports traffic per endpoint every five minutes.

Starting in version 2010, Configuration Manager rotates the storage account key for the CMG. This process happens automatically every 180 days.<!-- 8613077 -->

### Configuration Manager client-facing roles

The management point and software update point host endpoints in IIS to service client requests. The CMG doesn't expose all internal endpoints. Every endpoint published to the CMG has a URL mapping.

- The external URL is the one the client uses to communicate with the CMG.

- The internal URL is the CMG connection point used to forward requests to the internal server.

#### URL mapping example

When you enable CMG traffic on a management point, Configuration Manager creates an internal set of URL mappings for each management point server. For example: ccm_system, ccm_incoming, and sms_mp. The external URL for the management point ccm_system endpoint might look like:  
`https://<CMG service name>/CCM_Proxy_MutualAuth/<MP Role ID>/CCM_System`  
The URL is unique for each management point. The Configuration Manager client then puts the CMG-enabled management point name into its internet management point list. This name looks like:  
`<CMG service name>/CCM_Proxy_MutualAuth/<MP Role ID>`  
The site automatically uploads all published external URLs to the CMG. This behavior allows the CMG to do URL filtering. All URL mappings replicate to the CMG connection point. It then forwards the communication to internal servers according to the external URL from the client request.

## Security guidance for CMG

<a name="bkmk_crl"></a>

### Publish the certificate revocation list

Publish your PKI's certificate revocation list (CRL) for internet-based clients to access. When deploying a CMG using PKI, configure the service to **Verify client certificate revocation** on the Settings tab. This setting configures the service to use a published certificate revocation list (CRL). For more information, see [Plan for PKI certificate revocation](../../../plan-design/security/plan-for-security.md#BKMK_PlanningForCRLs).

This CMG option verifies the client authentication certificate.

- If the client is using Azure AD or Configuration Manager token-based authentication, the CRL doesn't matter.

- If you use PKI, and externally publish the CRL, then enable this option (recommended).

- If you use PKI, don't publish the CRL, then disable this option.

- If you misconfigure this option, it can cause additional traffic from clients to the CMG. This additional traffic can increase the Azure egress data, which can increase your Azure costs.<!-- SCCMDocs#1434 -->

<a name="bkmk_ctl"></a>

### Review entries in the site's certificate trust list

<!--503739-->
Each Configuration Manager site includes a list of trusted root certification authorities, the certificate trust list (CTL). View and modify the list by going to the **Administration** workspace, expand **Site Configuration**, and select **Sites**. Select a site, and then select **Properties** in the ribbon. Switch to the **Communication Security** tab, and then select **Set** under Trusted Root Certification Authorities.

Use a more restrictive CTL for a site with a CMG using PKI client authentication. Otherwise, clients with client authentication certificates issued by any trusted root that already exists on the management point are automatically accepted for client registration.

This subset provides administrators with more control over security. The CTL restricts the server to only accept client certificates that are issued from the certification authorities in the CTL. For example, Windows ships with a number of well-known third-party certification authority (CA) certificates, such as VeriSign and Thawte. By default, the computer running IIS trusts certificates that chain to these well-known CAs. Without configuring IIS with a CTL, any computer that has a client certificate issued from these CAs are accepted as a valid Configuration Manager client. If you configure IIS with a CTL that didn't include these CAs, client connections are refused if the certificate chained to these CAs.

### <a name="bkmk_tls"></a> Enforce TLS 1.2

<!-- SCCMDocs-pr#4021 -->

Use the CMG setting to **Enforce TLS 1.2**. It only applies to the Azure cloud service VM. It doesn't apply to any on-premises Configuration Manager site servers or clients. For more information on TLS 1.2, see [How to enable TLS 1.2](../../../plan-design/security/enable-tls-1-2.md).

### Use token-based authentication

Starting in version 2002,<!--5686290--> Configuration Manager extends its support for internet-based devices that don't often connect to the internal network, aren't able to join Azure AD, and don't have a method to install a PKI-issued certificate. The site automatically issues tokens for devices that register on the internal network. You can create a bulk registration token for internet-based devices. For more information, see [Token-based authentication for CMG](../../deploy/deploy-clients-cmg-token.md).<!-- SCCMDocs#2331 -->
