---
title: Configuración de la identidad de nube híbrida para aplicaciones de Azure y Azure Stack Hub
description: Aprenda a configurar la identidad de nube híbrida para aplicaciones de Azure y Azure Stack Hub.
author: BryanLa
ms.topic: article
ms.date: 11/05/2019
ms.author: bryanla
ms.reviewer: anajod
ms.lastreviewed: 11/05/2019
ms.openlocfilehash: 650eef0f144ecafab4586d93f72e1defdf4a61ce
ms.sourcegitcommit: d2def847937178f68177507be151df2aa8e25d53
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 07/20/2020
ms.locfileid: "86477259"
---
# <a name="configure-hybrid-cloud-identity-for-azure-and-azure-stack-hub-apps"></a>Configuración de la identidad de nube híbrida para aplicaciones de Azure y Azure Stack Hub

Aprenda a configurar la identidad de nube híbrida para sus aplicaciones de Azure y Azure Stack Hub.

Tiene dos opciones para conceder acceso a sus aplicaciones tanto en Azure global como en Azure Stack Hub.

 * Si Azure Stack Hub tiene conexión continua a Internet, puede usar Azure Active Directory (Azure AD).
 * Si Azure Stack Hub no tiene conexión a Internet, puede usar los Servicios de federación de Active Directory (AD FS).

Las entidades de servicio se usan para conceder acceso a las aplicaciones de Azure Stack Hub para la implementación o configuración mediante Azure Resource Manager en Azure Stack Hub.

En esta solución, creará un entorno de ejemplo para:

> [!div class="checklist"]
> - Establecer una identidad híbrida en Azure global y Azure Stack Hub
> - Recuperar un token para acceder a la API de Azure Stack Hub.

Para seguir los pasos de esta solución, debe tener permisos de operador de Azure Stack Hub.

> [!Tip]  
> ![hybrid-pillars.png](./media/solution-deployment-guide-cross-cloud-scaling/hybrid-pillars.png)  
> Microsoft Azure Stack Hub es una extensión de Azure. Azure Stack Hub aporta la agilidad e innovación de la informática en la nube a su entorno local, ya que habilita la única nube híbrida que permite crear e implementar aplicaciones híbridas en cualquier parte.  
> 
> En el artículo [Consideraciones de diseño de aplicaciones híbridas](overview-app-design-considerations.md) se examinan los pilares de la calidad del software (selección de ubicación, escalabilidad, disponibilidad, resistencia, manejabilidad y seguridad) para diseñar, implementar y usar aplicaciones híbridas. Las consideraciones de diseño ayudan a optimizar el diseño de aplicaciones híbridas y reducen los desafíos en los entornos de producción.

## <a name="create-a-service-principal-for-azure-ad-in-the-portal"></a>Crear a una entidad de servicio para Azure AD en el portal

Si implementó Azure Stack Hub con Azure AD como el almacén de identidades, puede crear entidades de servicio como lo hace para Azure. La sección [Uso de una identidad de aplicación para acceder a recursos](/azure-stack/operator/azure-stack-create-service-principals.md#manage-an-azure-ad-app-identity) muestra cómo realizar los pasos desde el portal. Asegúrese de que dispone de los [permisos de Azure AD necesarios](/azure/azure-resource-manager/resource-group-create-service-principal-portal#required-permissions) antes de comenzar.

## <a name="create-a-service-principal-for-ad-fs-using-powershell"></a>Creación de una entidad de servicio para AD FS mediante PowerShell

Si implementó Azure Stack Hub con AD FS, puede usar PowerShell para crear una entidad de servicio, asignar un rol para el acceso e iniciar sesión en PowerShell con dicha identidad. La sección [Uso de una identidad de aplicación para acceder a recursos](/azure-stack/operator/azure-stack-create-service-principals.md#manage-an-ad-fs-app-identity) muestra cómo realizar los pasos necesarios mediante PowerShell.

## <a name="using-the-azure-stack-hub-api"></a>Uso de la API de Azure Stack Hub

La solución [API de Azure Stack Hub](/azure-stack/user/azure-stack-rest-api-use.md) le guía por el proceso de recuperación de un token de acceso a la API de Azure Stack Hub.

## <a name="connect-to-azure-stack-hub-using-powershell"></a>Conexión a Azure Stack Hub mediante PowerShell

La guía de inicio rápido para [empezar a trabajar con PowerShell en Azure Stack Hub](/azure-stack/operator/azure-stack-powershell-install.md) le guía por los pasos necesarios para instalar Azure PowerShell y conectarse a su instalación de Azure Stack Hub.

### <a name="prerequisites"></a>Prerrequisitos

Necesita una instalación de Azure Stack Hub conectada a Azure AD con una suscripción a la que puede acceder. Si no tiene una instalación de Azure Stack Hub, puede usar estas instrucciones para configurar un [Kit de desarrollo de Azure Stack (ASDK)](/azure-stack/asdk/asdk-install.md).

#### <a name="connect-to-azure-stack-hub-using-code"></a>Conexión a Azure Stack Hub mediante código

Para conectarse a Azure Stack Hub mediante código, use la API de los puntos de conexión de Azure Resource Manager para obtener los puntos de conexión de autenticación y Graph para la instalación de Azure Stack Hub. Luego, autentíquese mediante solicitudes de REST. Puede encontrar una aplicación cliente de ejemplo en [GitHub](https://github.com/shriramnat/HybridARMApplication).

>[!Note]
>A menos que el SDK de Azure correspondiente al lenguaje seleccionado admita perfiles de API de Azure, puede que el SDK no funcione con Azure Stack Hub. Para más información acerca de los perfiles de la API de Azure, consulte el artículo [Administración de los perfiles de la versión de API](/azure-stack/user/azure-stack-version-profiles.md).

## <a name="next-steps"></a>Pasos siguientes

- Para más información sobre cómo se trata la identidad en Azure Stack Hub, consulte [Arquitectura de identidad para Azure Stack Hub](/azure-stack/operator/azure-stack-identity-architecture.md).
- Para más información sobre los patrones de nube de Azure, consulte [Patrones de diseño en la nube](/azure/architecture/patterns).
