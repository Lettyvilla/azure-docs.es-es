---
title: Configuración de la característica Start/Stop VMs during off-hours de Azure Automation
description: En este artículo se describe cómo configurar la característica Start/Stop VMs during off-hours para admitir diferentes casos de uso o escenarios.
services: automation
ms.subservice: process-automation
ms.date: 06/01/2020
ms.topic: conceptual
ms.openlocfilehash: b0bc23d515bebdd0d943bbad33c5ebba35a35605
ms.sourcegitcommit: bdd5c76457b0f0504f4f679a316b959dcfabf1ef
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 09/22/2020
ms.locfileid: "90987213"
---
# <a name="configure-startstop-vms-during-off-hours"></a>Configuración de la solución Start/Stop VMs during off-hours

En este artículo se describe cómo configurar la característica [Start/Stop VMs during off-hours](automation-solution-vm-management.md) para admitir los escenarios descritos. También puede obtener información sobre cómo:

* [Configurar notificaciones por correo electrónico](#configure-email-notifications)
* [Agregar una VM](#add-a-vm)
* [Excluir una VM](#exclude-a-vm)
* [Modificar las programaciones de inicio y apagado](#modify-the-startup-and-shutdown-schedules)

## <a name="scenario-1-startstop-vms-on-a-schedule"></a><a name="schedule"></a>Escenario 1: Iniciar o detener las máquinas virtuales según una programación

Este escenario es la configuración predeterminada cuando se implementa la característica Start/Stop VMs during off-hours por primera vez. Por ejemplo, puede configurar la característica para que detenga todas las máquinas virtuales de una suscripción por la noche, cuando salga del trabajo, e iniciarlas por la mañana cuando vuelva a la oficina. Al configurar las programaciones **Scheduled-StartVM** y **Scheduled-StopVM** durante la implementación, inician y detienen las máquinas virtuales de destino. 

La característica puede configurarse para detener solo las máquinas virtuales. Consulte [Modificación de las programaciones de inicio y apagado](#modify-the-startup-and-shutdown-schedules) para obtener información sobre cómo configurar una programación personalizada.

> [!NOTE]
> La zona horaria que usa la característica es su zona horaria actual cuando configura el parámetro de hora de la programación. Sin embargo, Azure Automation la guarda en formato UTC. No es necesario realizar ninguna conversión de zona horaria, ya que esto se realiza durante la implementación de la máquina.

Para controlar las máquinas virtuales que están en el ámbito, configure las variables `External_Start_ResourceGroupNames`, `External_Stop_ResourceGroupNames` y `External_ExcludeVMNames`.

Puede habilitar que el destino de la acción sea una suscripción y un grupo de recursos, o una lista específica de máquinas virtuales, pero no ambas cosas.

### <a name="target-the-start-and-stop-actions-against-a-subscription-and-resource-group"></a>Destino de las acciones de inicio y detención a un grupo de recursos y una suscripción

1. Configure las variables `External_Stop_ResourceGroupNames` y `External_ExcludeVMNames` para especificar las VM de destino.

2. Habilite y actualice las programaciones **Scheduled-StartVM** y **Scheduled-StopVM**.

3. Ejecute el runbook **ScheduledStartStop_Parent** con el campo de parámetro **ACTION** establecido en **start** y el campo de parámetro **WHATIF** establecido en True para obtener una vista previa de los cambios.

### <a name="target-the-start-and-stop-action-by-vm-list"></a>Destino de las acciones de inicio y detención por lista de máquinas virtuales

1. Ejecute el runbook **ScheduledStartStop_Parent** con **ACTION** establecido en **start**.

2. Agregue una lista separada por comas de máquinas virtuales (sin espacios) en el campo de parámetro **VMList**. Una lista de ejemplo es `vm1,vm2,vm3`.

3. Establezca el campo de parámetro **WHATIF** en True.

4. Configure la variable `External_ExcludeVMNames` con una lista separada por comas de máquinas virtuales (VM1, VM2, VM3), sin espacios entre los valores separados por comas.

5. En este escenario no se respetan las variables `External_Start_ResourceGroupNames` y `External_Stop_ResourceGroupnames`. Para este escenario, es preciso que cree su propia programación de Automation. Para más información, consulte [Programación de un runbook en Azure Automation](shared-resources/schedules.md).

    > [!NOTE]
    > El valor de **Target ResourceGroup Names** (Nombres de ResourceGroup de destino) se almacena como valor tanto para `External_Start_ResourceGroupNames` como para `External_Stop_ResourceGroupNames`. A fin de lograr un mayor detalle, puede modificar cada una de estas variables para distintos grupos de recursos de destino. Para la acción de inicio, use `External_Start_ResourceGroupNames`; y para la acción de detención, use `External_Stop_ResourceGroupNames`. Las máquinas virtuales se agregan automáticamente a las programaciones de inicio y detención.

## <a name="scenario-2-startstop-vms-in-sequence-by-using-tags"></a><a name="tags"></a>Escenario 2: Iniciar o detener las máquinas virtuales en secuencia mediante el uso de etiquetas

En un entorno que incluya dos, o más, componentes de varias máquinas virtuales que admitan una carga de trabajo distribuida, es importante que se pueda decidir la secuencia en que los componentes se inician o detienen en orden. 

### <a name="target-the-start-and-stop-actions-against-a-subscription-and-resource-group"></a>Destino de las acciones de inicio y detención a un grupo de recursos y una suscripción

1. Agregue las etiquetas `sequencestart` y `sequencestop` con un valor entero positivo a las máquinas virtuales de destino en las variables `External_Start_ResourceGroupNames` y `External_Stop_ResourceGroupNames`. Las acciones de inicio y detención se realizan en orden ascendente. Para aprender a etiquetar una máquina virtual, consulte [Etiquetado de una máquina virtual Windows en Azure](../virtual-machines/windows/tag.md) y [Etiquetado de una máquina virtual Linux en Azure](../virtual-machines/linux/tag.md).

2. Modifique las programaciones de **Sequenced-StartVM** y **Sequenced-StopVM** a una fecha y hora que cumplan sus requisitos, y habilite la programación.

3. Ejecute el runbook **SequencedStartStop_Parent** con **ACTION** establecido en **start** y **WHATIF** establecido en True para obtener una vista previa de los cambios.

4. Obtenga una vista previa de la acción y realice los cambios necesarios antes de implementarla en las máquinas virtuales de producción. Cuando esté preparado, ejecute manualmente el runbook con el parámetro establecido en **False** o deje que la programación de Automation **Sequenced-StartVM** y **Sequenced-StopVM** se ejecuten automáticamente siguiendo su programación prescrita.

### <a name="target-the-start-and-stop-actions-by-vm-list"></a>Destino de las acciones de inicio y detención por lista de máquinas virtuales

1. Agregue las etiquetas `sequencestart` y `sequencestop` con un valor entero positivo a las máquinas virtuales que pretende agregar al parámetro `VMList`.

2. Ejecute el runbook **SequencedStartStop_Parent** con **ACTION** establecido en **start**.

3. Agregue una lista separada por comas de máquinas virtuales (sin espacios) en el campo de parámetro **VMList**. Una lista de ejemplo es `vm1,vm2,vm3`.

4. Establezca **WHATIF** en True. 

5. Configure la variable `External_ExcludeVMNames` con una lista separada por comas de máquinas virtuales, sin espacios entre los valores separados por comas.

6. En este escenario no se respetan las variables `External_Start_ResourceGroupNames` y `External_Stop_ResourceGroupnames`. Para este escenario, es preciso que cree su propia programación de Automation. Para más información, consulte [Programación de un runbook en Azure Automation](shared-resources/schedules.md).

7. Obtenga una vista previa de la acción y realice los cambios necesarios antes de implementarla en las máquinas virtuales de producción. Cuando esté listo, ejecute manualmente **monitoring-and-diagnostics/monitoring-action-groupsrunbook** con el parámetro establecido en **False**. Como alternativa, deje que las programaciones de Automation **Sequenced-StartVM** y **Sequenced-StopVM** se ejecuten automáticamente siguiendo la programación prescrita.

## <a name="scenario-3-start-or-stop-automatically-based-on-cpu-utilization"></a><a name="cpuutil"></a>Escenario 3: Iniciar o detener automáticamente según el uso de CPU

La característica Start/Stop VMs during off-hours puede ayudarle a administrar el costo de la ejecución de máquinas virtuales de Azure Resource Manager y clásicas en su suscripción mediante la evaluación de las máquinas virtuales de Azure que no se utilizan durante los períodos de poca actividad, por ejemplo, las horas no laborables, y apagarlas automáticamente si el uso del procesador es menor de un porcentaje especificado.

De forma predeterminada, la característica está preconfigurada para evaluar la métrica de CPU en porcentaje, con el fin de ver si la utilización media es del 5 %, o menos. Este escenario se controla mediante las siguientes variables, y puede modificarse si los valores predeterminados no satisfacen sus requisitos:

* `External_AutoStop_MetricName`
* `External_AutoStop_Threshold`
* `External_AutoStop_TimeAggregationOperator`
* `External_AutoStop_TimeWindow`
* `External_AutoStop_Frequency`
* `External_AutoStop_Severity`

Puede habilitar que el destino de la acción sea una suscripción y un grupo de recursos, o una lista específica de máquinas virtuales.

Cuando se ejecuta el runbook **AutoStop_CreateAlert_Parent**, este comprueba que existen la suscripción, los grupos de recursos y las máquinas virtuales de destino. Si las máquinas virtuales existen, el runbook llama al runbook **AutoStop_CreateAlert_Child** de cada máquina virtual verificada mediante el runbook primario. Este runbook secundario:

* Crea una regla de alertas de métricas para cada máquina virtual comprobada.
* Desencadena el runbook **AutoStop_VM_Child** para una VM determinada si la CPU caiga por debajo del umbral configurado durante el intervalo de tiempo especificado. 
* Intenta detener la máquina virtual.

### <a name="target-the-autostop-action-against-all-vms-in-a-subscription"></a>Destino de la acción de detención automática a todas las máquinas virtuales de una suscripción

1. Asegúrese de que la variable `External_Stop_ResourceGroupNames` esté vacía o establecida en * (carácter comodín).

2. De manera opcional, si desea excluir algunas máquinas virtuales de la acción de detención automática, puede agregar una lista separada por comas de nombres de máquinas virtuales a la variable `External_ExcludeVMNames`.

3. Habilite la programación de **Schedule_AutoStop_CreateAlert_Parent** para que se ejecute para crear las reglas de alertas de métricas de detención de máquina virtual necesarias para todas las máquinas virtuales de la suscripción. La ejecución de este tipo de programación le permite crear nuevas reglas de alertas de métricas a medida que se agregan nuevas VM a la suscripción.

### <a name="target-the-autostop-action-against-all-vms-in-a-resource-group-or-multiple-resource-groups"></a>Destino de la acción de detención automática a todas las máquinas virtuales de un grupo de recursos o de varios grupos de recursos

1. Agregue una lista separada por comas de nombres de grupos de recursos a la variable `External_Stop_ResourceGroupNames`.

2. Si desea excluir algunas de las máquinas virtuales de la detención automática, puede agregar una lista separada por comas de nombres de máquinas virtuales a la variable `External_ExcludeVMNames`.

3. Habilite la programación de **Schedule_AutoStop_CreateAlert_Parent** para que se ejecute para crear las reglas de alertas de métricas de detención de VM necesarias para todas las VM de los grupos de recursos. La ejecución de esta operación según una programación le permitirá crear nuevas reglas de alertas de métricas a medida que se agregan nuevas VM a los grupos de recursos.

### <a name="target-the-autostop-action-to-a-list-of-vms"></a>Destino de la acción de detención automática a una lista de máquinas virtuales

1. Cree una nueva [programación](shared-resources/schedules.md#create-a-schedule) y vincúlela al runbook **AutoStop_CreateAlert_Parent**. Para ello, agregue una lista separada por comas con los nombres de las máquinas virtuales al parámetro `VMList`.

2. De manera opcional, si desea excluir algunas máquinas virtuales de la acción de detención automática, puede agregar una lista separada por comas de nombres de máquinas virtuales (sin espacios) a la variable `External_ExcludeVMNames`.

## <a name="configure-email-notifications"></a>Configuración de notificaciones de correo electrónico

Para cambiar las notificaciones de correo electrónico después de implementar la característica Start/Stop VMs during off-hours, puede modificar el grupo de acciones creado durante la implementación.  

> [!NOTE]
> Las suscripciones en la nube de Azure Government no admiten la funcionalidad de correo electrónico de esta característica.

1. En Azure Portal, vaya a **Supervisar** y, a continuación, **Grupos de acciones**. Seleccione el grupo de acciones denominado **StartStop_VM_Notication**.

    :::image type="content" source="media/automation-solution-vm-management/azure-monitor.png" alt-text="Captura de pantalla de la página Supervisar: grupos de acciones.":::

2. En la página StartStop_VM_Notification, haga clic en **Editar detalles** en **Detalles**. Se abrirá la página Correo electrónico/SMS/Push/Voz. Actualice la dirección de correo electrónico y haga clic en **Aceptar** para guardar los cambios.

    :::image type="content" source="media/automation-solution-vm-management/change-email.png" alt-text="Captura de pantalla de la página Supervisar: grupos de acciones.":::

    También puede agregar acciones adicionales al grupo de acciones; para más información sobre grupos de acciones, consulte [grupos de acciones](../azure-monitor/platform/action-groups.md)

El siguiente es un correo electrónico de ejemplo que se envía cuando la característica cierra las máquinas virtuales.

:::image type="content" source="media/automation-solution-vm-management/email.png" alt-text="Captura de pantalla de la página Supervisar: grupos de acciones.":::

## <a name="add-or-exclude-vms"></a><a name="add-exclude-vms"></a>Agregar o excluir máquinas virtuales

La característica permite agregar máquinas virtuales como destino o para excluirlas. 

### <a name="add-a-vm"></a>Agregar una VM

Hay dos maneras de asegurarse de que se incluya una máquina virtual cuando se ejecute la característica:

* Cada uno de los [runbooks](automation-solution-vm-management.md#runbooks) primarios de la característica tiene un parámetro `VMList`. Puede pasar una lista separada por comas de nombres de máquinas virtuales (sin espacios) a este parámetro al programar el runbook principal adecuado para su situación, y estas máquinas virtuales se incluirán cuando se ejecute la característica.

* Para seleccionar varias VM, establezca `External_Start_ResourceGroupNames` y `External_Stop_ResourceGroupNames` en los nombres de los grupos de recursos que contienen las VM que desea iniciar o detener. También se pueden establecer las variables en un valor de `*` para que la característica se ejecute en todos los grupos de recursos de la suscripción.

### <a name="exclude-a-vm"></a>Excluir una VM

Para excluir una máquina virtual de la característica Start/Stop VMs during off-hours, puede agregar su nombre a la variable `External_ExcludeVMNames`. Esta variable es una lista separada por comas de máquinas virtuales específicas (sin espacios) que se excluirán de la característica. Esta lista se limita a 140 VM. Si agrega más de 140 máquinas virtuales a esta lista separada por comas, las establecidas para excluirse pueden iniciarse o detenerse accidentalmente.

## <a name="modify-the-startup-and-shutdown-schedules"></a>Modificación de las programaciones de inicio y apagado

La administración de las programaciones de inicio y apagado en esta característica sigue los mismos pasos descritos en [Programación de un runbook en Azure Automation](shared-resources/schedules.md). Se requieren programaciones independientes para iniciar y detener las máquinas virtuales.

La característica puede configurarse para únicamente detener las máquinas virtuales en un momento determinado. En este escenario, simplemente creará una programación Stop y ninguna programación Start correspondiente. 

1. Asegúrese de que ha agregado los grupos de recursos para que las máquinas virtuales se apaguen en la variable `External_Stop_ResourceGroupNames`.

2. Cree su propia programación para el momento en el que desea apagar las máquinas virtuales.

3. Navegue hasta el runbook **ScheduledStartStop_Parent** y haga clic en **Programación**. Esto le permite seleccionar la programación que creó en el paso anterior.

4. Seleccione **Configuración de ejecución y parámetros** y establezca el campo **ACTION** en **Stop**.

5. Seleccione **Aceptar** para guardar los cambios.

## <a name="next-steps"></a>Pasos siguientes

* Para supervisar la característica durante la operación, vea [Consulta de registros de Start/Stop VMs during off-hours](automation-solution-vm-management-logs.md).
* Para solucionar problemas durante la administración de máquinas virtuales, consulte [Solución de problemas de la característica Start/Stop VMs during off-hours](troubleshoot/start-stop-vm.md).
