---
title: Implementar un grupo de disponibilidad de SQL Server 2016 en Azure y Azure Stack Hub
description: Aprenda a implementar un grupo de disponibilidad de SQL Server 2016 en Azure y Azure Stack Hub.
author: BryanLa
ms.topic: article
ms.date: 11/05/2019
ms.author: bryanla
ms.reviewer: anajod
ms.lastreviewed: 11/05/2019
ms.openlocfilehash: 85b859457b9b54a973c5fc23329b927212b60a07
ms.sourcegitcommit: d2def847937178f68177507be151df2aa8e25d53
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 07/20/2020
ms.locfileid: "86477089"
---
# <a name="deploy-a-sql-server-2016-availability-group-to-azure-and-azure-stack-hub"></a>Implementar un grupo de disponibilidad de SQL Server 2016 en Azure y Azure Stack Hub

Este artículo le guiará a lo largo de una implementación automatizada de un clúster de SQL Server 2016 Enterprise de alta disponibilidad (HA) con un sitio de recuperación ante desastres (DR) asincrónica entre dos entornos de Azure Stack Hub. Para obtener más información acerca de SQL Server 2016 y la alta disponibilidad, vea [Grupos de disponibilidad Always On: una solución de alta disponibilidad y recuperación ante desastres](/sql/database-engine/availability-groups/windows/always-on-availability-groups-sql-server?view=sql-server-2016).

En esta solución, creará un entorno de ejemplo para:

> [!div class="checklist"]
> - Organizar una implementación en dos instancias de Azure Stack Hub.
> - Usar Docker para minimizar los problemas de dependencia con los perfiles de la API de Azure.
> - Implementar un clúster de SQL Server 2016 Enterprise de alta disponibilidad básico con un sitio de recuperación ante desastres.

> [!Tip]  
> ![hybrid-pillars.png](./media/solution-deployment-guide-cross-cloud-scaling/hybrid-pillars.png)  
> Microsoft Azure Stack Hub es una extensión de Azure. Azure Stack Hub aporta la agilidad e innovación de la informática en la nube a su entorno local, ya que habilita la única nube híbrida que permite crear e implementar aplicaciones híbridas en cualquier parte.  
> 
> En el artículo [Consideraciones de diseño de aplicaciones híbridas](overview-app-design-considerations.md) se examinan los pilares de la calidad del software (selección de ubicación, escalabilidad, disponibilidad, resistencia, manejabilidad y seguridad) para diseñar, implementar y usar aplicaciones híbridas. Las consideraciones de diseño ayudan a optimizar el diseño de aplicaciones híbridas y reducen los desafíos en los entornos de producción.

## <a name="architecture-for-sql-server-2016"></a>Arquitectura para SQL Server 2016

![Azure Stack Hub de SQL de alta disponibilidad de SQL Server 2016](media/solution-deployment-guide-sql-ha/image1.png)

## <a name="prerequisites-for-sql-server-2016"></a>Requisitos previos para SQL Server 2016

- Dos sistemas integrados de Azure Stack Hub (Azure Stack Hub). Este desarrollo no funciona en el Kit de desarrollo de Azure Stack (ASDK). Para más información sobre Azure Stack Hub, consulte la [introducción a Azure Stack Hub](https://azure.microsoft.com/overview/azure-stack/).
- Una suscripción de inquilino en cada instancia de Azure Stack Hub.
  - **Tome nota de cada identificador de suscripción y del punto de conexión de Azure Resource Manager para cada instancia de Azure Stack Hub.**
- Una entidad de servicio de Azure Active Directory (Azure AD) que tenga permisos para la suscripción del inquilino en cada instancia de Azure Stack Hub. Es posible que deba crear dos entidades de servicio si las instancias de Azure Stack Hub se implementan en diferentes inquilinos de Azure AD. Para aprender a crear una entidad de servicio para Azure Stack Hub, consulte [Creación de entidades de servicio para otorgar a las aplicaciones acceso a los recursos de Azure Stack Hub](/azure-stack/user/azure-stack-create-service-principals).
  - **Tome nota del id. de aplicación, el secreto de cliente y el nombre de inquilino (xxxxx.onmicrosoft.com) de cada entidad de servicio.**
- SQL Server 2016 Enterprise sindicado en el Marketplace de cada instancia de Azure Stack Hub. Para más información acerca de la redifusión de Marketplace, consulte [Descarga de elementos de Marketplace en Azure Stack Hub](/azure-stack/operator/azure-stack-download-azure-marketplace-item).
    **Asegúrese de que su organización disponga de las licencias de SQL correspondientes.**
- [Docker para Windows](https://docs.docker.com/docker-for-windows/) instalado en la máquina local.

## <a name="get-the-docker-image"></a>Obtener la imagen de Docker

Las imágenes de Docker para cada implementación eliminan los problemas de dependencia entre las distintas versiones de Azure PowerShell.

1. Asegúrese de que Docker para Windows utiliza contenedores de Windows.
2. Ejecute el siguiente script en un símbolo del sistema con privilegios elevados para obtener el contenedor de Docker con los scripts de implementación.

    ```powershell  
    docker pull intelligentedge/sqlserver2016-hadr:1.0.0
    ```

## <a name="deploy-the-availability-group"></a>Implementar el grupo de disponibilidad

1. Una vez extraída correctamente la imagen de contenedor, inicie la imagen.

      ```powershell  
      docker run -it intelligentedge/sqlserver2016-hadr:1.0.0 powershell
      ```

2. Una vez iniciado el contenedor, se le ofrecerá un terminal de PowerShell con privilegios elevados en el contenedor. Cambie los directorios para acceder al script de implementación.

      ```powershell  
      cd .\SQLHADRDemo\
      ```

3. Ejecute la implementación. Proporcione las credenciales y los nombres de los recursos cuando sea necesario. Alta disponibilidad hace referencia a la instancia de Azure Stack Hub en la que se instalará el clúster de alta disponibilidad. Recuperación ante desastres hace referencia a la instancia de Azure Stack Hub en la que se instalará el clúster de recuperación ante desastres.

      ```powershell
      > .\Deploy-AzureResourceGroup.ps1 `
      -AzureStackApplicationId_HA "applicationIDforHAServicePrincipal" `
      -AzureStackApplicationSercet_HA "clientSecretforHAServicePrincipal" `
      -AADTenantName_HA "hatenantname.onmicrosoft.com" `
      -AzureStackResourceGroup_HA "haresourcegroupname" `
      -AzureStackArmEndpoint_HA "https://management.haazurestack.com" `
      -AzureStackSubscriptionId_HA "haSubscriptionId" `
      -AzureStackApplicationId_DR "applicationIDforDRServicePrincipal" `
      -AzureStackApplicationSercet_DR "ClientSecretforDRServicePrincipal" `
      -AADTenantName_DR "drtenantname.onmicrosoft.com" `
      -AzureStackResourceGroup_DR "drresourcegroupname" `
      -AzureStackArmEndpoint_DR "https://management.drazurestack.com" `
      -AzureStackSubscriptionId_DR "drSubscriptionId"
      ```

4. Escriba `Y` para permitir que el proveedor de NuGet se instale, lo que iniciará los módulos "2018-03-01-hybrid" del perfil de la API que se van a instalar.

5. Espere a que la implementación de recursos se complete.

6. Una vez completada la implementación de recursos de DR, salga del contenedor.

      ```powershell
      exit
      ```

7. Para inspeccionar la implementación, observe los recursos del portal de cada instancia de Azure Stack Hub. Conéctese a una de las instancias de SQL en el entorno de alta disponibilidad e inspeccione el grupo de disponibilidad mediante SQL Server Management Studio (SSMS).

    ![Alta disponibilidad de SQL de SQL Server 2016](media/solution-deployment-guide-sql-ha/image2.png)

## <a name="next-steps"></a>Pasos siguientes

- Use SQL Server Management Studio para conmutar por error el clúster. Consulte [Realización de una conmutación por error manual forzada de un grupo de disponibilidad Always On (SQL Server)](/sql/database-engine/availability-groups/windows/perform-a-forced-manual-failover-of-an-availability-group-sql-server?view=sql-server-2017)
- Más información sobre las aplicaciones en la nube híbrida. Consulte [Soluciones de nube híbrida.](https://aka.ms/azsdevtutorials)
- Use sus propios datos o modifique el código según este ejemplo en [GitHub](https://github.com/Azure-Samples/azure-intelligent-edge-patterns).
