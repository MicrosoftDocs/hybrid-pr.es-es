---
title: Patrón DevOps en Azure Stack Hub
description: Más información sobre el patrón DevOps para que pueda garantizar la coherencia entre las implementaciones de Azure y Azure Stack Hub.
author: BryanLa
ms.topic: article
ms.date: 11/05/2019
ms.author: bryanla
ms.reviewer: anajod
ms.lastreviewed: 11/05/2019
ms.openlocfilehash: 306cc9604a8e919724f9f76b7e5122d534d2d1ae
ms.sourcegitcommit: bb3e40b210f86173568a47ba18c3cc50d4a40607
ms.translationtype: MT
ms.contentlocale: es-ES
ms.lasthandoff: 06/17/2020
ms.locfileid: "84911957"
---
# <a name="devops-pattern"></a>Patrón DevOps

Programe desde una única ubicación y realice la implementación en varios destinos en entornos de desarrollo, pruebas y producción que pueden estar ubicados en el centro de datos local, en nubes privadas o en la nube pública.

## <a name="context-and-problem"></a>Contexto y problema

La continuidad, la seguridad y la confiabilidad de la implementación de las aplicaciones resultan fundamentales para las organizaciones y vitales para los equipos de desarrollo.

A menudo, las aplicaciones requieren la refactorización de código para ejecutarse en cada entorno de destino. Esto significa que la aplicación no es totalmente portátil. Debe actualizarse, probarse y validarse a medida que se traslada a cada entorno. Por ejemplo, el código escrito en un entorno de desarrollo debe volver a escribirse para que funcione en un entorno de pruebas y debe volver a escribirse cuando finalmente llega a un entorno de producción. Además, este código está vinculado específicamente al host, lo que aumenta el costo y la complejidad del mantenimiento de la aplicación. Cada versión de la aplicación está vinculada a cada entorno. Esta mayor complejidad y la duplicación aumentan los riesgos relacionados con la seguridad y la calidad del código. Además, el código no se puede volver a implementar fácilmente cuando se eliminan los hosts con error de restauración o se implementan hosts adicionales para administrar los aumentos de la demanda.

## <a name="solution"></a>Solución

El patrón DevOps le permite compilar, probar e implementar una aplicación que se ejecuta en varias nubes. Este patrón une las prácticas de integración continua y de entrega continua. Con la integración continua, el código se compila y se prueba cada vez que un miembro del equipo confirma un cambio en el control de versiones. La entrega continua automatiza cada paso de una compilación a un entorno de producción. Juntos, estos procesos crean un proceso de versión que admite la implementación en diversos entornos. Con este patrón, puede preparar el código y, a continuación, implementarlo en un entorno local, en nubes privadas diferentes y en nubes públicas. Las diferencias en el entorno requieren realizar un cambio en un archivo de configuración en lugar de cambios en el código.

![Patrón DevOps](media/pattern-cicd-pipeline/hybrid-ci-cd.png)

Con un conjunto coherente de herramientas de desarrollo en entornos locales, de nube privada y de nube pública, puede implementar una práctica de integración continua y entrega continua. Las aplicaciones y los servicios implementados con el patrón DevOps son intercambiables y pueden ejecutarse en cualquiera de estas ubicaciones, aprovechando las características y funcionalidades de los entornos local y de nube pública.

El uso de una canalización de versión de DevOps le ayuda a realizar lo siguiente:

- Iniciar una nueva compilación basada en confirmaciones de código en un único repositorio.
- Implementar automáticamente el código recién compilado en la nube pública para llevar a cabo las pruebas de aceptación de usuario.
- Realizar la implementación automáticamente a una nube privada una vez que el código ha pasado las pruebas.

## <a name="issues-and-considerations"></a>Problemas y consideraciones

El patrón DevOps está pensado para garantizar la coherencia entre las implementaciones, independientemente del entorno de destino. Sin embargo, las funcionalidades varían en los entornos local y de nube. Considere los siguientes puntos:

- ¿Las funciones, los puntos de conexión, los servicios y los demás recursos de la implementación están disponibles en las ubicaciones de implementación de destino?
- ¿Los artefactos de configuración se almacenan en ubicaciones a las que se puede acceder a través de nubes?
- ¿Los parámetros de implementación funcionarán en todos los entornos de destino?
- ¿Las propiedades específicas de los recursos están disponibles en todas las nubes de destino?

Para obtener más información, consulte [Desarrollo de plantillas de Azure Resource Manager para mantener la coherencia en la nube](https://docs.microsoft.com/azure/azure-resource-manager/templates-cloud-consistency).

Asimismo, a la hora de decidir cómo implementar este patrón, debe tener en cuenta los siguientes puntos:

### <a name="scalability"></a>Escalabilidad

Los sistemas de automatización de la implementación son el punto de control clave en los patrones de DevOps. Las implementaciones pueden variar. La elección del tamaño correcto del servidor depende del tamaño de la carga de trabajo esperada. Las máquinas virtuales son más difíciles de escalar que los contenedores. Sin embargo, para utilizar contenedores para el escalado, el proceso de compilación debe ejecutarse con contenedores.

### <a name="availability"></a>Disponibilidad

La disponibilidad en el contexto de DevPattern significa poder recuperar cualquier información de estado asociada al flujo de trabajo como, por ejemplo, los resultados de las pruebas, las dependencias de código u otros artefactos. Para evaluar los requisitos de disponibilidad, tenga en cuenta dos métricas comunes:

- Objetivo de tiempo de recuperación (RTO) especifica cuánto tiempo se puede estar sin un sistema.

- Objetivo de punto de recuperación (RPO) indica la cantidad de datos que puede permitirse perder si una interrupción del servicio afecta al sistema.

En la práctica, las métricas RTO y el RPO implican redundancia y copia de seguridad. En la nube global de Azure, la disponibilidad no es una cuestión de recuperación de hardware, que forma parte de Azure, sino más bien de garantizar que mantenga el estado de sus sistemas DevOps. En Azure Stack Hub, es posible que deba tenerse en cuenta la recuperación de hardware.

Otro factor importante a tener en cuenta a la hora de diseñar el sistema que se usa para la automatización de la implementación es el control de acceso y la administración adecuada de los derechos necesarios para implementar servicios en entornos de nube. ¿Qué derechos son necesarios para crear, eliminar o modificar implementaciones? Por ejemplo, normalmente se necesita un conjunto de derechos para crear un grupo de recursos en Azure y otro para implementar servicios en el grupo de recursos.

### <a name="manageability"></a>Facilidad de uso

El diseño de cualquier sistema basado en el patrón DevOps debe tener en cuenta la automatización, el registro y las alertas de cada servicio de la cartera. Use servicios compartidos, un equipo de aplicaciones o ambos, y realice un seguimiento de las directivas de seguridad y de la gobernanza.

Implemente entornos de producción y entornos de desarrollo y pruebas en grupos de recursos independientes en Azure o Azure Stack Hub. De este modo, puede supervisar los recursos de cada entorno y acumular los costos de facturación por grupo de recursos. También se pueden eliminar recursos en conjunto, lo que resulta útil cuando se realizan implementaciones de prueba.

## <a name="when-to-use-this-pattern"></a>Cuándo usar este patrón

Use este patrón en los siguientes casos:

- Puede desarrollar código en un entorno que satisfaga las necesidades de los desarrolladores e implementarlo en un entorno específico de la solución en el que puede que sea difícil desarrollar código nuevo.
- Puede usar el código y las herramientas que los desarrolladores quieran, siempre y cuando puedan seguir el proceso de integración y entrega continuas del patrón de DevOps.

No se recomienda este patrón:

- Si no puede automatizar la infraestructura, el aprovisionamiento de recursos, la configuración, la identidad y las tareas de seguridad.
- Si los equipos no tienen acceso a recursos de nube híbrida para implementar un enfoque de integración continua/desarrollo continuo (CI/CD).

## <a name="next-steps"></a>Pasos siguientes

Para más información sobre los temas presentados en este artículo:

- Consulte la [documentación de Azure DevOps](/azure/devops) para más información sobre Azure DevOps y las herramientas relacionadas, como Azure Repos y Azure Pipelines.
- Consulte la información relativa a la [familia de productos y soluciones de Azure Stack](/azure-stack) para más información sobre toda la gama de productos y soluciones.

Cuando esté listo para probar la solución de ejemplo, continúe con la [guía de implementación de la solución de CI/CD híbrida de DevOps](https://aka.ms/hybriddevopsdeploy). La guía de implementación proporciona instrucciones paso a paso para implementar y probar sus componentes. Aprenda a implementar una aplicación en Azure y Azure Stack Hub mediante una canalización híbrida de integración y entrega continuas (CI/CD).
