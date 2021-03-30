---
title: Implementación de una aplicación híbrida con datos locales que se escalan en la nube
description: Aprenda a implementar aplicaciones que usan datos locales y que se escalan en la nube mediante Azure y Azure Stack Hub.
author: BryanLa
ms.topic: article
ms.date: 11/05/2019
ms.author: bryanla
ms.reviewer: anajod
ms.lastreviewed: 11/05/2019
ms.openlocfilehash: 0989859fd68847932d3e69defee59740a2bffd44
ms.sourcegitcommit: 962334135b63ac99c715e7bc8fb9282648ba63c9
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 03/23/2021
ms.locfileid: "104895404"
---
# <a name="deploy-hybrid-app-with-on-premises-data-that-scales-cross-cloud"></a>Implementación de una aplicación híbrida con datos locales que se escalan en la nube

En esta guía de soluciones se muestra cómo implementar una aplicación híbrida que abarca Azure y Azure Stack Hub, y que usa un solo origen de datos local.

Mediante el uso de una solución en la nube híbrida, puede combinar las ventajas de cumplimiento de una nube privada con la escalabilidad de la nube pública. Los desarrolladores pueden aprovechar el ecosistema de desarrolladores de Microsoft y aplicar sus aptitudes a los entornos en la nube y locales.

## <a name="overview-and-assumptions"></a>Introducción y supuestos

Siga este tutorial para configurar un flujo de trabajo que permita a los desarrolladores implementar una aplicación web idéntica en una nube pública y en una privada. Esta aplicación puede acceder a una red enrutable sin conexión a Internet hospedada en la nube privada. Estas aplicaciones web se supervisan y, cuando hay un pico de tráfico, un programa modifica los registros del DNS para redirigir el tráfico a la nube pública. Cuando el tráfico disminuye hasta el nivel anterior al pico, el tráfico se enruta de vuelta a la nube privada.

En este tutorial se describen las tareas siguientes:

> [!div class="checklist"]
> - Implementar un servidor de base de datos de SQL Server con conexión híbrida.
> - Conectar una aplicación web de Azure global a una red híbrida.
> - Configurar DNS para el escalado entre nubes.
> - Configurar certificados SSL para el escalado entre nubes.
> - Configurar e implementar la aplicación web.
> - Crear un perfil de Traffic Manager y configurarlo para el escalado entre nubes.
> - Configurar la supervisión y las alertas de Application Insights para un aumento del tráfico.
> - Configuración de la conmutación automática del tráfico entre Azure global y Azure Stack Hub.

> [!Tip]  
> ![Diagrama de fundamentos de las aplicaciones híbridas](./media/solution-deployment-guide-cross-cloud-scaling/hybrid-pillars.png)  
> Microsoft Azure Stack Hub es una extensión de Azure. Azure Stack Hub aporta la agilidad y la innovación de la informática en la nube a su entorno local y hace posible la única nube híbrida que le permite crear e implementar aplicaciones híbridas en cualquier parte.  
> 
> En el artículo [Consideraciones de diseño de aplicaciones híbridas](overview-app-design-considerations.md) se examinan los fundamentos de calidad del software (selección de ubicación, escalabilidad, disponibilidad, resistencia, manejabilidad y seguridad) para diseñar, implementar y usar aplicaciones híbridas. Las consideraciones de diseño ayudan a optimizar el diseño de aplicaciones híbridas y reducen los desafíos en los entornos de producción.

### <a name="assumptions"></a>Supuestos

En este tutorial se da por supuesto que tiene conocimientos básicos de Azure global y Azure Stack Hub. Para más información antes de iniciar el tutorial, consulte los siguientes artículos:

- [Introducción a Azure](https://azure.microsoft.com/overview/what-is-azure/)
- [Conceptos clave de Azure Stack Hub](/azure-stack/operator/azure-stack-overview)

En este tutorial también se supone que tiene una suscripción de Azure. Si no tiene una suscripción, [cree una cuenta gratuita](https://azure.microsoft.com/free/) antes de empezar.

## <a name="prerequisites"></a>Prerrequisitos

Antes de comenzar esta solución, asegúrese de que cumple los requisitos siguientes:

- Un Kit de desarrollo de Azure Stack (ASDK) o una suscripción a un sistema integrado de Azure Stack Hub. Para implementar el ASDK, siga las instrucciones que encontrará en el artículo acerca de la [implementación del Kit de desarrollo de Azure Stack mediante el instalador](/azure-stack/asdk/asdk-install).
- Para la instalación de Azure Stack Hub es necesario tener instalado lo siguiente:
  - Azure App Service. Trabaje con su operador de Azure Stack Hub para implementar y configurar Azure App Service en su entorno. Este tutorial requiere que App Service tenga al menos un (1) rol de trabajo dedicado disponible.
  - Una imagen de Windows Server 2016.
  - Una imagen de Windows Server 2016 con Microsoft SQL Server.
  - Los planes y ofertas adecuados.
  - Un nombre de dominio para la aplicación web. Si no tiene un nombre de dominio, puede comprarlo a un proveedor de dominios como GoDaddy, Bluehost o InMotion.
- Un certificado SSL para el dominio de una entidad de certificación de confianza como LetsEncrypt.
- Una aplicación web que se comunica con una base de datos de SQL Server y es compatible con Application Insights. Puede descargar la aplicación de ejemplo [dotnetcore-sqldb-tutorial](https://github.com/Azure-Samples/dotnetcore-sqldb-tutorial) de GitHub.
- Una red híbrida entre una red virtual de Azure y la red virtual de Azure Stack Hub. Para instrucciones detalladas, consulte [Configuración de la conectividad de la nube híbrida con Azure y Azure Stack Hub](solution-deployment-guide-connectivity.md).

- Una canalización híbrida de integración continua e implementación continua (CI/CD) con un agente de compilación privado en Azure Stack Hub. Para instrucciones detalladas, consulte [Configuración de la identidad de nube híbrida con aplicaciones de Azure y Azure Stack Hub](solution-deployment-guide-identity.md).

## <a name="deploy-a-hybrid-connected-sql-server-database-server"></a>Implementación de un servidor de base de datos de SQL Server con conexión híbrida

1. Inicie sesión en el portal de usuarios de Azure Stack Hub.

2. En **Dashboard** (Panel), seleccione **Marketplace**.

    ![Marketplace de Azure Stack Hub](media/solution-deployment-guide-hybrid/image1.png)

3. En **Marketplace**, seleccione **Compute** (Proceso) y, a continuación, elija **More** (Más). En **Más**, seleccione **Free SQL Server License: SQL Server 2017 Developer en Windows Server**.

    ![Seleccione una imagen de máquina virtual en el portal de usuarios de Azure Stack Hub](media/solution-deployment-guide-hybrid/image2.png)

4. En **Free SQL Server License: SQL Server 2017 Developer en Windows Server**, seleccione **Crear**.

5. En **Aspectos básicos > Configurar opciones básicas**, proporcione un **Nombre** para la máquina virtual (VM), un **Nombre de usuario**  para el usuario SA de SQL Server y una **Contraseña** para el usuario SA.  En la lista desplegable **Suscripción**, seleccione la suscripción en la que se va a realizar la implementación. En **Resource group** (Grupo de recursos), use **Choose existing** (Elegir existente) y coloque la máquina virtual en el mismo grupo de recursos que la aplicación web de Azure Stack Hub.

    ![Configuración de los valores básicos para la máquina virtual en el portal de usuarios de Azure Stack Hub](media/solution-deployment-guide-hybrid/image3.png)

6. En **Size** (Tamaño), elija un tamaño de máquina virtual. Para este tutorial se recomienda A2_Standard o DS2_V2_Standard.

7. En **Configuración > Configurar características opcionales**, configure las siguientes opciones:

   - **Cuenta de almacenamiento**: Si necesita una, cree una nueva cuenta.
   - **Red virtual**:

     > [!Important]  
     > Asegúrese de que la máquina virtual de SQL Server se implementa en la misma red virtual que las puertas de enlace de VPN.

   - **Dirección IP pública**: Use la configuración predeterminada.
   - **Grupo de seguridad de red**: (NSG). Cree un nuevo grupo de seguridad de red.
   - **Extensiones y Supervisión**: Mantenga la configuración predeterminada.
   - **Cuenta de almacenamiento de diagnóstico**: Si necesita una, cree una nueva cuenta.
   - Seleccione **OK** (Aceptar) para guardar la configuración.

     ![Configuración de las características opcionales de la máquina virtual en el portal de usuarios de Azure Stack Hub](media/solution-deployment-guide-hybrid/image4.png)

8. En **SQL Server settings** (Configuración de SQL Server), configure las siguientes opciones:

   - En **SQL connectivity** (Conectividad SQL), seleccione **Public (Internet)** (Pública [Internet]).
   - En **Port** (Puerto), mantenga el valor predeterminado, **1433**.
   - En **SQL authentication** (Autenticación SQL), seleccione **Enable** (Habilitar).

     > [!Note]  
     > Al habilitar la autenticación SQL, se rellena de forma automática con la información de "SQLAdmin" que configuró en **Aspectos básicos**.

   - Para el resto de la configuración, conserve los valores predeterminados. Seleccione **Aceptar**.

     ![Configuración de SQL Server en el portal de usuarios de Azure Stack Hub](media/solution-deployment-guide-hybrid/image5.png)

9. En **Summary** (Resumen), revise la configuración de la máquina virtual y seleccione **OK** (Aceptar) para iniciar la implementación.

    ![Resumen de la configuración en el portal de usuarios de Azure Stack Hub](media/solution-deployment-guide-hybrid/image6.png)

10. La nueva VM tarda algún tiempo en crearse. Puede ver el estado de las máquinas virtuales en **Virtual machines** (Máquinas virtuales).

    ![Estado de las máquinas virtuales en el portal de usuarios de Azure Stack Hub](media/solution-deployment-guide-hybrid/image7.png)

## <a name="create-web-apps-in-azure-and-azure-stack-hub"></a>Creación de aplicaciones web en Azure y Azure Stack Hub

Azure App Service simplifica la ejecución y administración de una aplicación web. Dado que Azure Stack Hub es coherente con Azure, App Service se puede ejecutar en ambos entornos. Usará App Service para hospedar la aplicación.

### <a name="create-web-apps"></a>Creación de aplicaciones web

1. Cree una aplicación web en Azure siguiendo las instrucciones de [Administración de un plan de App Service en Azure](/azure/app-service/app-service-plan-manage#create-an-app-service-plan). Debe asegurarse de colocar la aplicación web en la misma suscripción y grupo de recursos que la red híbrida.

2. Repita el paso anterior (1) en Azure Stack Hub.

### <a name="add-route-for-azure-stack-hub"></a>Incorporación de una ruta para Azure Stack Hub

App Service en Azure Stack Hub debe ser enrutable desde la red pública de Internet para que los usuarios puedan acceder a la aplicación. Si la instancia de Azure Stack Hub es accesible desde Internet, anote la dirección IP pública o la dirección URL de la aplicación web de Azure Stack Hub.

Si usa un ASDK, puede [configurar una asignación NAT estática](/azure-stack/operator/azure-stack-create-vpn-connection-one-node#configure-the-nat-vm-on-each-asdk-for-gateway-traversal) para exponer App Service fuera del entorno virtual.

### <a name="connect-a-web-app-in-azure-to-a-hybrid-network"></a>Conexión de una aplicación web de Azure a una red híbrida

Para proporcionar conectividad entre el front-end web en Azure y la base de datos de SQL Server en Azure Stack Hub, la aplicación web debe estar conectada a la red híbrida entre Azure y Azure Stack Hub. Para habilitar la conectividad, tendrá que:

- Configurar la conectividad de punto a sitio.
- Configurar la aplicación web.
- Modificar la puerta de enlace de red local en Azure Stack Hub.

### <a name="configure-the-azure-virtual-network-for-point-to-site-connectivity"></a>Configuración de la red virtual de Azure para la conectividad de punto a sitio

La puerta de enlace de red virtual en el lado de Azure de la red híbrida debe permitir conexiones de punto a sitio para la integración con Azure App Service.

1. En Azure Portal, vaya a la página de la puerta de enlace de red virtual. En **Configuración**, seleccione **Configuración de punto a sitio**.

    ![Opción de punto a sitio en la puerta de enlace de red virtual de Azure](media/solution-deployment-guide-hybrid/image8.png)

2. Seleccione **Configurar ahora** para la configuración de punto a sitio.

    ![Iniciar configuración de punto a sitio en la puerta de enlace de red virtual de Azure](media/solution-deployment-guide-hybrid/image9.png)

3. En la página de configuración **De punto a sitio**, escriba el intervalo de direcciones IP privadas que desea usar en **Grupo de direcciones**.

   > [!Note]  
   > Asegúrese de que el intervalo que especifique no se superponga con ninguno de los intervalos de direcciones ya utilizados por las subredes de Azure global o los componentes de Azure Stack Hub de la red híbrida.

   En **Tipo de túnel**, desactive la opción **VPN IKEv2**. Seleccione **Guardar** para finalizar la configuración de punto a sitio.

   ![Configuración de punto a sitio en la puerta de enlace de red virtual de Azure](media/solution-deployment-guide-hybrid/image10.png)

### <a name="integrate-the-azure-app-service-app-with-the-hybrid-network"></a>Integración de la aplicación de Azure App Service con la red híbrida

1. Para conectar la aplicación a la red virtual de Azure, siga las instrucciones de [Gateway required VNet integration](/azure/app-service/web-sites-integrate-with-vnet#gateway-required-vnet-integration) (Integración de VNet requerida de la puerta de enlace).

2. Vaya a **Configuración** en el plan de App Service que hospeda la aplicación web. En **Configuración**, seleccione **Redes**.

    ![Configurar redes para el plan de App Service](media/solution-deployment-guide-hybrid/image11.png)

3. En **Integración de VNET**, seleccione **Haga clic aquí para administrar**.

    ![Administración de la integración de red virtual para el plan de App Service](media/solution-deployment-guide-hybrid/image12.png)

4. Seleccione la red virtual que desea configurar. En **DIRECCIONES IP ENRUTADAS A RED VIRTUAL**, escriba el intervalo de direcciones IP de la red virtual de Azure, la red virtual de Azure Stack Hub y los espacios de direcciones de punto a sitio. Seleccione **Guardar** para validar y guardar la configuración.

    ![Intervalos de direcciones IP que se enrutan en la integración de la red virtual](media/solution-deployment-guide-hybrid/image13.png)

Para más información acerca de cómo se integra App Service con las redes virtuales de Azure, consulte [Integración de la aplicación con una red virtual de Azure](/azure/app-service/web-sites-integrate-with-vnet).

### <a name="configure-the-azure-stack-hub-virtual-network"></a>Configuración de la red virtual de Azure Stack Hub

La puerta de enlace de red local de la red virtual de Azure Stack Hub debe estar configurada para enrutar el tráfico desde el intervalo de direcciones de punto a sitio de App Service.

1. En el portal de Azure Stack Hub, vaya a **Puerta de enlace de red local**. En **Settings** (Configuración), seleccione **Configuration** (Configuración).

    ![Opción de configuración de puerta de enlace en la puerta de enlace de red local de Azure Stack Hub](media/solution-deployment-guide-hybrid/image14.png)

2. En **Espacio de direcciones**, escriba el intervalo de direcciones de punto a sitio para la puerta de enlace de red virtual de Azure.

    ![Espacio de direcciones de punto a sitio en la puerta de enlace de red local de Azure Stack Hub](media/solution-deployment-guide-hybrid/image15.png)

3. Seleccione **Guardar** para validar y guardar la configuración.

## <a name="configure-dns-for-cross-cloud-scaling"></a>Configuración de DNS para el escalado entre nubes

Mediante la configuración adecuada de DNS en las aplicaciones de toda la nube, los usuarios pueden acceder a las instancias de Azure global y de Azure Stack Hub de la aplicación web. La configuración de DNS de este tutorial también permite que Azure Traffic Manager enrute el tráfico cuando la carga aumenta o disminuye.

En este tutorial, se usa Azure DNS para administrar el DNS porque los dominios de App Service no funcionarían.

### <a name="create-subdomains"></a>Creación de subdominios

Dado que Traffic Manager se basa en registros CNAME de DNS, se necesita un subdominio para enrutar correctamente el tráfico a los puntos de conexión. Para más información acerca de la asignación de dominios y registros DNS, consulte [Asignación de dominios con Traffic Manager](/azure/app-service/web-sites-traffic-manager-custom-domain-name).

Para el punto de conexión de Azure, creará un subdominio que los usuarios pueden utilizar para acceder a la aplicación web. Para este tutorial, puede usar **app.northwind.com**, pero este valor se debe personalizar en función de su propio dominio.

También deberá crear un subdominio con un registro A para el punto de conexión de Azure Stack Hub. Puede usar **azurestack.northwind.com**.

### <a name="configure-a-custom-domain-in-azure"></a>Configuración de un dominio personalizado en Azure

1. Agregue el nombre de host **app.northwind.com** a la aplicación web de Azure mediante la [Asignación de un registro CNAME a Azure App Service](/azure/app-service/app-service-web-tutorial-custom-domain#map-a-cname-record).

### <a name="configure-custom-domains-in-azure-stack-hub"></a>Configuración de dominios personalizados en Azure Stack Hub

1. Agregue el nombre de host **azurestack.northwind.com** a la aplicación web de Azure Stack Hub mediante [Asignación de un registro A para Azure App Service](/azure/app-service/app-service-web-tutorial-custom-domain#map-an-a-record). Use la dirección IP enrutable por Internet para la aplicación de App Service.

2. Agregue el nombre de host **app.northwind.com** a la aplicación web de Azure Stack Hub mediante [Asignación de un registro CNAME a Azure App Service](/azure/app-service/app-service-web-tutorial-custom-domain#map-a-cname-record). Utilice el nombre de host que configuró en el paso anterior (1) como destino para el registro CNAME.

## <a name="configure-ssl-certificates-for-cross-cloud-scaling"></a>Configuración de certificados SSL para el escalado entre nubes

Es importante asegurarse de que los datos confidenciales que recopila la aplicación web están seguros en reposo y en tránsito cuando se almacenan en la base de datos SQL.

Configurará las aplicaciones web de Azure y Azure Stack Hub para que usen certificados SSL con todo el tráfico entrante.

### <a name="add-ssl-to-azure-and-azure-stack-hub"></a>Adición de SSL en Azure y Azure Stack Hub

Para agregar SSL en Azure:

1. Asegúrese de que el certificado SSL que obtiene es válido para el subdominio que ha creado. (Es correcto usar certificados con caracteres comodín).

2. En Azure Portal, siga las instrucciones de las secciones **Preparar la aplicación web** y **Enlace su certificado SSL** del artículo en el que se explica cómo [enlazar un certificado SSL personalizado existente con Azure Web Apps](/azure/app-service/app-service-web-tutorial-custom-ssl). Seleccione **SSL basado en SNI** como el **Tipo de SSL**.

3. Redirija todo el tráfico al puerto HTTPS. Siga las instrucciones de la sección **Aplicación de HTTPS** del artículo [Enlace de un certificado SSL personalizado existente con Azure Web Apps](/azure/app-service/app-service-web-tutorial-custom-ssl).

Para agregar SSL a Azure Stack Hub:

1. Repita los pasos 1-3 que usó para Azure, pero ahora use el portal de Azure Stack Hub.

## <a name="configure-and-deploy-the-web-app"></a>Configurar e implementar la aplicación web

Deberá configurar el código de la aplicación para notificar los datos de telemetría a la instancia de Application Insights correcta y configurar las aplicaciones web con las cadenas de conexión apropiadas. Para más información sobre Application Insights, consulte [¿Qué es Application Insights?](/azure/application-insights/app-insights-overview)

### <a name="add-application-insights"></a>Adición de Application Insights

1. Abra la aplicación web en Microsoft Visual Studio.

2. [Agregue Application Insights](/azure/azure-monitor/app/asp-net-core#enable-client-side-telemetry-for-web-applications) al proyecto para transmitir los datos de telemetría que utiliza Application Insights para crear alertas cuando el tráfico web aumenta o disminuye.

### <a name="configure-dynamic-connection-strings"></a>Configuración de cadenas de conexión dinámicas

Cada instancia de la aplicación web usará un método diferente para conectarse a la base de datos SQL. La aplicación de Azure usa la dirección IP privada de la máquina virtual de SQL Server y la aplicación de Azure Stack Hub usa la dirección IP pública de la máquina virtual con SQL Server.

> [!Note]  
> En un sistema integrado de Azure Stack Hub, la dirección IP pública no debería ser enrutable en Internet. En un Kit de desarrollo de Azure Stack, la dirección IP pública no se puede enrutar fuera del ASDK.

Puede usar variables de entorno de App Service para pasar una cadena de conexión distinta a cada instancia de la aplicación.

1. Abra la aplicación en Visual Studio.

2. Abra Startup.cs y busque el bloque de código siguiente:

    ```C#
    services.AddDbContext<MyDatabaseContext>(options =>
        options.UseSqlite("Data Source=localdatabase.db"));
    ```

3. Reemplace el bloque de código anterior por el código siguiente, que usa una cadena de conexión definida en el archivo *appsettings.json*:

    ```C#
    services.AddDbContext<MyDatabaseContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("MyDbConnection")));
     // Automatically perform database migration
     services.BuildServiceProvider().GetService<MyDatabaseContext>().Database.Migrate();
    ```

### <a name="configure-app-service-app-settings"></a>Configurar los parámetros de la aplicación en App Service

1. Cree cadenas de conexión para Azure y Azure Stack Hub. Las cadenas deben ser iguales, excepto las direcciones IP utilizadas.

2. En Azure y Azure Stack Hub, agregue la cadena de conexión adecuada [como una configuración de la aplicación](/azure/app-service/web-sites-configure) en la aplicación web y use `SQLCONNSTR\_` como prefijo en el nombre.

3. **Guarde** la configuración de la aplicación web y reinicie la aplicación.

## <a name="enable-automatic-scaling-in-global-azure"></a>Habilitación del escalado automático en Azure global

Cuando se crea la aplicación web en un entorno de App Service, esta se inicia con una instancia. Puede escalar horizontalmente de manera automática para agregar instancias y proporcionar más recursos de proceso a la aplicación. De forma similar, puede reducir horizontalmente de manera automática y reducir el número de instancias que necesita la aplicación.

> [!Note]  
> Debe tener un plan de App Service para configurar el escalado y la reducción horizontales. Si no tiene un plan, cree uno antes de iniciar los pasos siguientes.

### <a name="enable-automatic-scale-out"></a>Habilitación de la escalabilidad horizontal automática

1. En Azure Portal, busque el plan de App Service de los sitios que desea escalar horizontalmente y, después, seleccione **Escalar horizontalmente (plan de App Service)** .

    ![Escalar horizontalmente Azure App Service](media/solution-deployment-guide-hybrid/image16.png)

2. Seleccione **Habilitar escalado automático**.

    ![Habilitar la escalabilidad automática en Azure App Service](media/solution-deployment-guide-hybrid/image17.png)

3. Escriba un nombre para **Nombre de la configuración de escalado automático**. Para la regla de escalado automático **Predeterminada**, seleccione **Escalado basado en una métrica**. Establezca el valor de **Límites de instancia** en **Mínimo: 1**, **Máximo: 10** y **Predeterminado: 1**.

    ![Configurar la escalabilidad automática en Azure App Service](media/solution-deployment-guide-hybrid/image18.png)

4. Seleccione **+Agregar una regla**.

5. En **Origen de métrica**, seleccione **Recurso actual**. Utilice los siguientes criterios y acciones para la regla.

#### <a name="criteria"></a>Criterios

1. En **Agregación de tiempo**, seleccione **Media**.

2. En **Nombre de métrica**, seleccione **Porcentaje de CPU**.

3. En **Operador**, seleccione **Mayor que**.

   - Establezca **Umbral** en **50**.
   - Establezca **Duración** en **10**.

#### <a name="action"></a>Acción

1. En **Operación**, seleccione **Aumentar recuento en**.

2. Establezca **Recuento de instancias** en **2**.

3. Establezca **Tiempo de finalización** en **5**.

4. Seleccione **Agregar**.

5. Seleccione **+Agregar una regla**.

6. En **Origen de métrica**, seleccione **Recurso actual**.

   > [!Note]  
   > El recurso actual contendrá el nombre o el identificador único del plan de App Service y las listas desplegables **Tipo de recurso** y **Recurso** no estarán disponibles.

### <a name="enable-automatic-scale-in"></a>Habilitación de la reducción horizontal automática

Cuando el tráfico disminuye, la aplicación web de Azure puede reducir automáticamente el número de instancias activas para reducir los costos. Esta acción es menos agresiva que la escalabilidad horizontal y reduce el impacto sobre los usuarios de la aplicación.

1. Vaya a la condición del escalado horizontal **Predeterminada** y seleccione **+ Agregar una regla**. Utilice los siguientes criterios y acciones para la regla.

#### <a name="criteria"></a>Criterios

1. En **Agregación de tiempo**, seleccione **Media**.

2. En **Nombre de métrica**, seleccione **Porcentaje de CPU**.

3. En **Operador**, seleccione **Menor que**.

   - Establezca **Umbral** en **30**.
   - Establezca **Duración** en **10**.

#### <a name="action"></a>Acción

1. En **Operación**, seleccione **Reducir recuento en**.

   - Establezca **Recuento de instancias** en **1**.
   - Establezca **Tiempo de finalización** en **5**.

2. Seleccione **Agregar**.

## <a name="create-a-traffic-manager-profile-and-configure-cross-cloud-scaling"></a>Creación de un perfil de Traffic Manager y configuración del escalado entre nubes

Cree un perfil de Traffic Manager desde Azure Portal y, después, configure puntos de conexión para habilitar el escalado entre nubes.

### <a name="create-traffic-manager-profile"></a>Creación de un perfil de Traffic Manager

1. Seleccione **Crear un recurso**.
2. Seleccionar **Redes**.
3. Seleccione **Perfil de Traffic Manager** y configure las opciones siguientes:

   - En **Nombre**, escriba un nombre para el perfil. Este nombre **debe** ser único en la zona trafficmanager.net y se utiliza para crear un nuevo nombre de DNS (por ejemplo, northwindstore.trafficmanager.net).
   - En **Método de enrutamiento**, seleccione **Ponderado**.
   - En **Suscripción**, seleccione la suscripción en la que desea crear este perfil.
   - En **Grupo de recursos**, cree un grupo de recursos nuevo para este perfil.
   - En **Ubicación del grupo de recursos**, seleccione la ubicación del grupo de recursos. Este valor hace referencia a la ubicación del grupo de recursos y no tiene efecto alguno sobre el perfil de Traffic Manager que se implementa globalmente.

4. Seleccione **Crear**.

    ![Creación de un perfil de Traffic Manager](media/solution-deployment-guide-hybrid/image19.png)

   Una vez que se completa la implementación global del perfil de Traffic Manager, aparece en la lista de recursos del grupo de recursos en el que se creó.

### <a name="add-traffic-manager-endpoints"></a>Incorporación de puntos de conexión de Traffic Manager

1. Busque el perfil de Traffic Manager que creó. Si estaba en el grupo de recursos del perfil, seleccione el perfil.

2. En **Perfil de Traffic Manager**, en **Configuración**, seleccione **Puntos de conexión**.

3. Seleccione **Agregar**.

4. En **Agregar punto de conexión**, use la siguiente configuración para Azure Stack Hub:

   - En **Tipo**, seleccione **Punto de conexión externo**.
   - Escriba el **Nombre** del punto de conexión.
   - En **Nombre de dominio completo (FQDN) o dirección IP**, escriba la dirección URL externa de la aplicación web de Azure Stack Hub.
   - En **Peso**, mantenga el valor predeterminado, **1**. Este peso hace que todo el tráfico vaya a este punto de conexión si funciona correctamente.
   - No active la opción **Agregar como deshabilitado**.

5. Seleccione **Aceptar** para guardar el punto de conexión de Azure Stack Hub.

A continuación, configurará el punto de conexión de Azure.

1. En **Perfil de Traffic Manager**, seleccione **Puntos de conexión**.
2. Seleccione **+Agregar**.
3. En **Agregar punto de conexión**, utilice la siguiente configuración para Azure:

   - En **Tipo**, seleccione **Punto de conexión de Azure**.
   - Escriba el **Nombre** del punto de conexión.
   - En **Tipo de recurso de destino**, seleccione **App Service**.
   - En **Recurso de destino**, seleccione **Elegir un servicio de aplicaciones** para mostrar la lista de Web Apps en la misma suscripción.
   - En **Recursos**, elija el servicio de aplicación que quiere agregar como el primer punto de conexión.
   - En **Peso**, seleccione **2**. Esta opción hace que todo el tráfico vaya a este punto de conexión si el punto de conexión principal está en mal estado o si tiene una regla o alerta que redirige el tráfico cuando se desencadena.
   - No active la opción **Agregar como deshabilitado**.

4. Seleccione **Aceptar** para guardar el punto de conexión de Azure.

Después de que ambos puntos de conexión están configurados, aparecerán en el **Perfil de Traffic Manager** al seleccionar **Puntos de conexión**. El ejemplo de la captura de pantalla siguiente muestra dos puntos de conexión con la información de estado y configuración de cada uno de ellos.

![Puntos de conexión del perfil de Traffic Manager](media/solution-deployment-guide-hybrid/image20.png)

## <a name="set-up-application-insights-monitoring-and-alerting-in-azure"></a>Configuración de la supervisión y generación de alertas de Application Insights en Azure

Azure Application Insights permite supervisar la aplicación y enviar alertas basadas en las condiciones que configure. Algunos ejemplos son: la aplicación no está disponible, experimenta errores o muestra problemas de rendimiento.

Para crear alertas, usará las métricas de Azure Application Insights. Cuando se desencadenen estas alertas, la instancia de la aplicación web cambiará automáticamente de Azure Stack Hub a Azure para escalarse horizontalmente y volverá a Azure Stack Hub para reducirse horizontalmente.

### <a name="create-an-alert-from-metrics"></a>Creación de una alerta a partir de métricas

En Azure Portal, vaya al grupo de recursos de este tutorial y seleccione la instancia de Application Insights para abrir **Application Insights**.

![Application Insights](media/solution-deployment-guide-hybrid/image21.png)

Esta vista se usará para crear una alerta de escalabilidad horizontal y una alerta de reducción horizontal.

### <a name="create-the-scale-out-alert"></a>Creación de la alerta de escalabilidad horizontal

1. En **Configurar**, seleccione **Alertas (clásica)** .
2. Seleccione **Agregar una alerta de métrica (clásica)** .
3. En **Agregar regla**, configure las opciones siguientes:

   - En **Nombre**, escriba **Burst into Azure Cloud**.
   - La **Descripción** es opcional.
   - En **Origen** > **Alerta sobre**, seleccione **Métricas**.
   - En **Criterios**, seleccione la suscripción, el grupo de recursos del perfil de Traffic Manager y el nombre del perfil de Traffic Manager para el recurso.

4. En **Métrica**, seleccione **Velocidad de solicitudes de**.
5. En **Condición**, seleccione **Mayor que**.
6. En **Umbral**, escriba **2**.
7. En **Período**, seleccione **En los últimos 5 minutos**.
8. En **Notificar mediante**:
   - Active la casilla de verificación para **Correo electrónico a propietarios, colaboradores y lectores**.
   - En **Correos electrónicos adicionales del administrador**, escriba su dirección de correo electrónico.

9. En la barra de menús, seleccione **Guardar**.

### <a name="create-the-scale-in-alert"></a>Creación de la alerta de reducción horizontal

1. En **Configurar**, seleccione **Alertas (clásica)** .
2. Seleccione **Agregar una alerta de métrica (clásica)** .
3. En **Agregar regla**, configure las opciones siguientes:

   - En **Nombre**, escriba **Scale back into Azure Stack Hub**.
   - La **Descripción** es opcional.
   - En **Origen** > **Alerta sobre**, seleccione **Métricas**.
   - En **Criterios**, seleccione la suscripción, el grupo de recursos del perfil de Traffic Manager y el nombre del perfil de Traffic Manager para el recurso.

4. En **Métrica**, seleccione **Velocidad de solicitudes de**.
5. En **Condición**, seleccione **Menor que**.
6. En **Umbral**, escriba **2**.
7. En **Período**, seleccione **En los últimos 5 minutos**.
8. En **Notificar mediante**:
   - Active la casilla de verificación para **Correo electrónico a propietarios, colaboradores y lectores**.
   - En **Correos electrónicos adicionales del administrador**, escriba su dirección de correo electrónico.

9. En la barra de menús, seleccione **Guardar**.

En la captura de pantalla siguiente se muestran las alertas de escalabilidad horizontal y reducción horizontal.

   ![Alertas de Application Insights (clásico)](media/solution-deployment-guide-hybrid/image22.png)

## <a name="redirect-traffic-between-azure-and-azure-stack-hub"></a>Redirección del tráfico entre Azure y Azure Stack Hub

Puede configurar la conmutación manual o automática del tráfico de la aplicación web entre Azure y Azure Stack Hub.

### <a name="configure-manual-switching-between-azure-and-azure-stack-hub"></a>Configuración de la conmutación manual entre Azure y Azure Stack Hub

Cuando el sitio web llegue a los umbrales que configuró, recibirá una alerta. Utilice los pasos siguientes para redirigir el tráfico a Azure manualmente.

1. En Azure Portal, seleccione el perfil de Traffic Manager.

    ![Puntos de conexión de Traffic Manager en Azure Portal](media/solution-deployment-guide-hybrid/image20.png)

2. Seleccione **Puntos de conexión**.
3. Seleccione el **punto de conexión de Azure**.
4. En **Estado**, seleccione **Habilitado** y, a continuación, seleccione **Guardar**.

    ![Habilitar punto de conexión de Azure en Azure Portal](media/solution-deployment-guide-hybrid/image23.png)

5. En **Puntos de conexión** del perfil de Traffic Manager, seleccione **Punto de conexión externo**.
6. En **Estado**, seleccione **Deshabilitado** y, a continuación, seleccione **Guardar**.

    ![Deshabilitar punto de conexión de Azure Stack Hub en Azure Portal](media/solution-deployment-guide-hybrid/image24.png)

Después de configurar los puntos de conexión, el tráfico de la aplicación se dirige a la aplicación web de escalabilidad horizontal de Azure, y no a la aplicación web de Azure Stack Hub.

 ![Puntos de conexión cambiados en el tráfico de aplicaciones web de Azure](media/solution-deployment-guide-hybrid/image25.png)

Para invertir el flujo de vuelta a Azure Stack Hub, use los pasos anteriores para:

- Habilitar el punto de conexión de Azure Stack Hub.
- Deshabilitar el punto de conexión de Azure.

### <a name="configure-automatic-switching-between-azure-and-azure-stack-hub"></a>Configurar la conmutación automática entre Azure y Azure Stack Hub.

También puede usar la supervisión de Application Insights si la aplicación se ejecuta en un entorno [sin servidor](https://azure.microsoft.com/overview/serverless-computing/) proporcionado por Azure Functions.

En este escenario, puede configurar Application Insights para que use un webhook que llama a una aplicación de función. Esta aplicación habilita o deshabilita automáticamente un punto de conexión en respuesta a una alerta.

Use los pasos siguientes como guía para configurar la conmutación automática del tráfico.

1. Cree una aplicación de función de Azure.
2. Cree una función desencadenada por HTTP.
3. Importe los SDK de Azure para Resource Manager, Web Apps y Traffic Manager.
4. Desarrolle código para:

   - Autenticarse en la suscripción de Azure.
   - Usar un parámetro que alterne los puntos de conexión de Traffic Manager para dirigir el tráfico a Azure o Azure Stack Hub.

5. Guarde el código y agregue la dirección URL de la aplicación de función con los parámetros adecuados en la sección **Webhook** de la configuración de la regla de alertas de Application Insights.
6. El tráfico se redirige automáticamente cuando se desencadena una alerta de Application Insights.

## <a name="next-steps"></a>Pasos siguientes

- Para más información sobre los patrones de nube de Azure, consulte [Patrones de diseño en la nube](/azure/architecture/patterns).
