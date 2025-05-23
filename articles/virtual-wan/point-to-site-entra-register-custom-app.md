---
title: Create custom app ID for P2S VPN Microsoft Entra ID authentication
titleSuffix: Azure Virtual WAN
description: Learn how to create or modify a custom audience App ID or upgrade an existing custom App ID to the new Microsoft-registered Azure VPN Client app values for Azure Virtual WAN.
author: cherylmc
ms.service: azure-virtual-wan
ms.topic: concept-article
ms.date: 02/25/2025
ms.author: cherylmc
---

# Create or modify a custom audience app ID for User VPN Microsoft Entra ID authentication

The steps in this article help you create a Microsoft Entra ID custom App ID (custom audience) for the new Microsoft-registered Azure VPN Client for User VPN point-to-site (P2S) connections. You can also update your existing tenant to [change the new Microsoft-registered Azure VPN Client app](#change) from the previous Azure VPN Client app.

This article provides high-level steps. The screenshots to register an application might be slightly different, depending on the way you access the user interface, but the settings are the same. For more information, see [Quickstart: Register an application](/entra/identity-platform/quickstart-register-app).

## Prerequisites

* This article assumes that you already have a Microsoft Entra tenant and the permissions to create an Enterprise Application, typically the [Cloud Application Administrator role](/entra/identity/role-based-access-control/permissions-reference#cloud-application-administrator) or higher. For more information, see [Create a new tenant in Microsoft Entra ID](/entra/fundamentals/create-new-tenant) and [Assign user roles with Microsoft Entra ID](/entra/fundamentals/users-assign-role-azure-portal).

* We recommend that you use the audience value `c632b3df-fb67-4d84-bdcf-b95ad541b5c8` to configure your custom app. This value has global consent, which means you don't need to manually register it to provide consent for your organization.

  1. To grant admin consent for your organization, modify the following command to contain the desired `client_id` value. See the [table](../vpn-gateway/point-to-site-about.md#entra-id) for additional supported values.

     ```https://login.microsoftonline.com/common/oauth2/authorize?client_id=41b23e61-6c1e-4545-b367-cd054e0ed4b4&response_type=code&redirect_uri=https://portal.azure.com&nonce=1234&prompt=admin_consent```

  1. Copy and paste the URL that pertains to your deployment location in the address bar of your browser.
  1. Select the account that has the [Cloud Application Administrator role](/entra/identity/role-based-access-control/permissions-reference#cloud-application-administrator) if prompted.
  1. On the **Permissions** requested page, select **Accept**.

[!INCLUDE [Configure custom audience](../../includes/vpn-gateway-custom-audience.md)]

## Configure the gateway

After you've completed the steps in the previous sections, continue to [Configure Virtual WAN User VPN for Microsoft Entra ID authentication](point-to-site-entra-gateway.md).

## <a name="change"></a>Update to Microsoft-registered VPN app Client ID

> [!NOTE]
> While these steps can be used for any of the supported values associated with the Azure VPN Client app, we recommend that you associate the Microsoft-registered App ID value `c632b3df-fb67-4d84-bdcf-b95ad541b5c8` to your custom app.

[!INCLUDE [Change custom audience](../../includes/vpn-gateway-custom-audience-change.md)]

## Next steps

[Configure Virtual WAN P2S User VPN for Microsoft Entra ID authentication](point-to-site-entra-gateway.md).