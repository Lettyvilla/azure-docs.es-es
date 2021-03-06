---
title: Creación e implementación de una especificación de plantilla
description: Obtenga información sobre cómo crear una especificación de plantilla a partir de una plantilla de Resource Manager. A continuación, implemente la especificación de plantilla en un grupo de recursos de su suscripción.
author: tfitzmac
ms.date: 10/02/2020
ms.topic: quickstart
ms.author: tomfitz
ms.openlocfilehash: e1bd2f3da8204cf5947d0cf423cd57b023674608
ms.sourcegitcommit: 638f326d02d108cf7e62e996adef32f2b2896fd5
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 10/05/2020
ms.locfileid: "91728784"
---
# <a name="quickstart-create-and-deploy-template-spec-preview"></a>Inicio rápido: Creación e implementación de una especificación de plantilla (versión preliminar)

En esta guía de inicio rápido se muestra cómo empaquetar una plantilla de Azure Resource Manager (plantilla de ARM) como [especificación de plantilla](template-specs.md) y, a continuación, impleméntela. La especificación de plantilla contiene una plantilla de Resource Manager que implementa una cuenta de almacenamiento.

## <a name="prerequisites"></a>Requisitos previos

Una cuenta de Azure con una suscripción activa. [Cree una cuenta gratuita](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).

> [!NOTE]
> Las especificaciones de plantilla se encuentran actualmente en versión preliminar. Para usarlas, debe [registrarse para la lista de espera](https://aka.ms/templateSpecOnboarding).
>
> Después de obtener la aprobación de la lista de espera, recibirá instrucciones para instalar la versión preliminar del módulo de PowerShell y del módulo de la CLI.

## <a name="create-template-spec"></a>Creación de una especificación de plantilla

La especificación de plantilla es un tipo de recurso llamado **Microsoft.Resources/templateSpecs**. Para crear la especificación de plantilla, puede usar Azure PowerShell, la CLI de Azure o una plantilla de ARM. En todas las opciones, necesita una plantilla de Resource Manager que esté empaquetada en la especificación de la plantilla.

Con PowerShell y la CLI, la plantilla de ARM se pasa como un parámetro al comando. Con la plantilla de Resource Manager, la que se empaquetará dentro de la especificación de plantilla se inserta dentro de la definición de la especificación de plantilla.

Estas opciones se muestran a continuación.

# <a name="powershell"></a>[PowerShell](#tab/azure-powershell)

1. Al crear una especificación de plantilla con PowerShell, puede pasar una plantilla local. Copie la siguiente plantilla y guárdela en un archivo local llamado **azuredeploy.json**. En esta guía de inicio rápido se da por supuesto que se ha guardado en la ruta de acceso **c:\Templates\azuredeploy.json**, pero puede usar cualquier ruta de acceso.

    :::code language="json" source="~/quickstart-templates/101-storage-account-create/azuredeploy.json":::

1. Cree un nuevo grupo de recursos que contiene la especificación de plantilla.

    ```azurepowershell
    New-AzResourceGroup `
      -Name templateSpecRG `
      -Location westus2
    ```

1. A continuación, cree la especificación de plantilla en ese grupo de recursos. Asigne a la nueva especificación de plantilla el nombre **storageSpec**.

    ```azurepowershell
    New-AzTemplateSpec `
      -Name storageSpec `
      -Version "1.0" `
      -ResourceGroupName templateSpecRG `
      -Location westus2 `
      -TemplateFile "c:\Templates\azuredeploy.json"
    ```

# <a name="cli"></a>[CLI](#tab/azure-cli)

1. Al crear una especificación de plantilla con la CLI, puede pasar una plantilla local. Copie la siguiente plantilla y guárdela en un archivo local llamado **azuredeploy.json**. En esta guía de inicio rápido se da por supuesto que se ha guardado en la ruta de acceso **c:\Templates\azuredeploy.json**, pero puede usar cualquier ruta de acceso.

    :::code language="json" source="~/quickstart-templates/101-storage-account-create/azuredeploy.json":::

1. Cree un nuevo grupo de recursos que contiene la especificación de plantilla.

    ```azurecli
    az group create \
      --name templateSpecRG \
      --location westus2
    ```

1. A continuación, cree la especificación de plantilla en ese grupo de recursos. Asigne a la nueva especificación de plantilla el nombre **storageSpec**.

    ```azurecli
    az ts create \
      --name storageSpec \
      --version "1.0" \
      --resource-group templateSpecRG \
      --location "westus2" \
      --template-file "c:\Templates\azuredeploy.json"
    ```

# <a name="arm-template"></a>[Plantilla de ARM](#tab/azure-resource-manager)

1. Cuando se usa una plantilla de Resource Manager para crear la especificación de plantilla, la plantilla se incrusta en la definición de recurso. Copie la siguiente plantilla y guárdela localmente como **azuredeploy.json**. En esta guía de inicio rápido se da por supuesto que se ha guardado en la ruta de acceso **c:\Templates\azuredeploy.json**, pero puede usar cualquier ruta de acceso.

    > [!NOTE]
    > En la plantilla insertada, todos los corchetes de apertura se deben escapar con un segundo corchete de apertura. Use `[[` en lugar de `[`.

    ```json
    {
      "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
      "contentVersion": "1.0.0.0",
      "parameters": {},
      "functions": [],
      "variables": {},
      "resources": [
        {
          "type": "Microsoft.Resources/templateSpecs",
          "apiVersion": "2019-06-01-preview",
          "name": "storageSpec",
          "location": "westus2",
          "properties": {
            "displayName": "Storage template spec"
          },
          "tags": {},
          "resources": [
            {
              "type": "versions",
              "apiVersion": "2019-06-01-preview",
              "name": "1.0",
              "location": "westus2",
              "dependsOn": [ "storageSpec" ],
              "properties": {
                "template": {
                  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
                  "contentVersion": "1.0.0.0",
                  "parameters": {
                    "storageAccountType": {
                      "type": "string",
                      "defaultValue": "Standard_LRS",
                      "allowedValues": [
                        "Standard_LRS",
                        "Standard_GRS",
                        "Standard_ZRS",
                        "Premium_LRS"
                      ],
                      "metadata": {
                        "description": "Storage Account type"
                      }
                    },
                    "location": {
                      "type": "string",
                      "defaultValue": "[[resourceGroup().location]",
                      "metadata": {
                        "description": "Location for all resources."
                      }
                    }
                  },
                  "variables": {
                    "storageAccountName": "[[concat('store', uniquestring(resourceGroup().id))]"
                  },
                  "resources": [
                    {
                      "type": "Microsoft.Storage/storageAccounts",
                      "apiVersion": "2019-04-01",
                      "name": "[[variables('storageAccountName')]",
                      "location": "[[parameters('location')]",
                      "sku": {
                        "name": "[[parameters('storageAccountType')]"
                      },
                      "kind": "StorageV2",
                      "properties": {}
                    }
                  ],
                  "outputs": {
                    "storageAccountName": {
                      "type": "string",
                      "value": "[[variables('storageAccountName')]"
                    }
                  }
                }
              },
              "tags": {}
            }
          ]
        }
      ],
      "outputs": {}
    }
    ```

1. Use la CLI de Azure o PowerShell para crear un nuevo grupo de recursos.

    ```azurepowershell
    New-AzResourceGroup `
      -Name templateSpecRG `
      -Location westus2
    ```

    ```azurecli
    az group create \
      --name templateSpecRG \
      --location westus2
    ```

1. Implemente la plantilla con la CLI de Azure o PowerShell.

    ```azurepowershell
    New-AzResourceGroupDeployment `
      -ResourceGroupName templateSpecRG `
      -TemplateFile "c:\Templates\azuredeploy.json"
    ```

    ```azurecli
    az deployment group create \
      --name templateSpecRG \
      --template-file "c:\Templates\azuredeploy.json"
    ```

---

## <a name="deploy-template-spec"></a>Implementación de la especificación de plantilla

Ahora puede implementar la especificación de plantilla. La implementación de la especificación de plantilla es similar a la implementación de la plantilla que contiene, con la excepción de que se pasa el identificador de recurso de la especificación de plantilla. Puede usar los mismos comandos de implementación y, si es necesario, pasar los valores de parámetro para la especificación de plantilla.

# <a name="powershell"></a>[PowerShell](#tab/azure-powershell)

1. Cree un grupo de recursos que contenga la cuenta de almacenamiento nueva.

    ```azurepowershell
    New-AzResourceGroup `
      -Name storageRG `
      -Location westus2
    ```

1. Obtenga el id. de recurso de la especificación de plantilla.

    ```azurepowershell
    $id = (Get-AzTemplateSpec -ResourceGroupName templateSpecRG -Name storageSpec -Version "1.0").Versions.Id
    ```

1. Implemente la especificación de plantilla.

    ```azurepowershell
    New-AzResourceGroupDeployment `
      -TemplateSpecId $id `
      -ResourceGroupName storageRG
    ```

1. Proporcione los parámetros exactamente como lo haría para una plantilla de ARM. Vuelva a implementar la especificación de plantilla con un parámetro para el tipo de cuenta de almacenamiento.

    ```azurepowershell
    New-AzResourceGroupDeployment `
      -TemplateSpecId $id `
      -ResourceGroupName storageRG `
      -storageAccountType Standard_GRS
    ```

# <a name="cli"></a>[CLI](#tab/azure-cli)

1. Cree un grupo de recursos que contenga la cuenta de almacenamiento nueva.

    ```azurecli
    az group create \
      --name storageRG \
      --location westus2
    ```

1. Obtenga el id. de recurso de la especificación de plantilla.

    ```azurecli
    id = $(az ts show --name storageSpec --resource-group templateSpecRG --version "1.0" --query "id")
    ```

    > [!NOTE]
    > Existe un problema conocido al obtener el identificador de la especificación de plantilla y luego asignárselo a una variable en Windows PowerShell.

1. Implemente la especificación de plantilla.

    ```azurecli
    az deployment group create \
      --resource-group storageRG \
      --template-spec $id
    ```

1. Proporcione los parámetros exactamente como lo haría para una plantilla de ARM. Vuelva a implementar la especificación de plantilla con un parámetro para el tipo de cuenta de almacenamiento.

    ```azurecli
    az deployment group create \
      --resource-group storageRG \
      --template-spec $id \
      --parameters storageAccountType='Standard_GRS'
    ```

# <a name="arm-template"></a>[Plantilla de ARM](#tab/azure-resource-manager)

1. Copie la siguiente plantilla y guárdela en un archivo local llamado **storage.json**.

    ```json
       {
      "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
      "contentVersion": "1.0.0.0",
      "parameters": {},
      "functions": [],
      "variables": {},
      "resources": [
        {
          "type": "Microsoft.Resources/deployments",
          "apiVersion": "2020-06-01",
          "name": "demo",
          "properties": {
            "templateLink": {
              "id": "[resourceId('templateSpecRG', 'Microsoft.Resources/templateSpecs/versions', 'storageSpec', '1.0')]"
            },
            "parameters": {
            },
            "mode": "Incremental"
          }
        }
      ],
      "outputs": {}
    }
    ```

1. Use la CLI de Azure o PowerShell para crear un nuevo grupo de recursos para la cuenta de almacenamiento.

    ```azurepowershell
    New-AzResourceGroup `
      -Name storageRG `
      -Location westus2
    ```

    ```azurecli
    az group create \
      --name storageRG \
      --location westus2
    ```

1. Implemente la plantilla con la CLI de Azure o PowerShell.

    ```azurepowershell
    New-AzResourceGroupDeployment `
      -ResourceGroupName storageRG `
      -TemplateFile "c:\Templates\storage.json"
    ```

    ```azurecli
    az deployment group create \
      --name storageRG \
      --template-file "c:\Templates\storage.json"
    ```

---

## <a name="grant-access"></a>Conceder acceso

Si quiere permitir que otros usuarios de su organización implementen la especificación de plantilla, debe concederles acceso de lectura. Puede asignar el rol Lector a un grupo de Azure AD para el grupo de recursos que contiene las especificaciones de plantilla que quiere compartir. Para más información, consulte el [Tutorial: Concesión de acceso de grupo a recursos de Azure mediante Azure PowerShell](../../role-based-access-control/tutorial-role-assignments-group-powershell.md).

## <a name="clean-up-resources"></a>Limpieza de recursos

Para limpiar el recurso que implementó en esta guía de inicio rápido, elimine los dos grupos de recursos que creó.

1. En Azure Portal, seleccione el grupo de recursos en el menú de la izquierda.

1. Escriba el nombre del grupo de recursos (templateSpecRG y storageRG) en el campo Filtrar por nombre.

1. Seleccione el nombre del grupo de recursos.

1. Seleccione Eliminar grupo de recursos en el menú superior.

## <a name="next-steps"></a>Pasos siguientes

Para obtener información sobre cómo crear una especificación de plantilla que incluya plantillas vinculadas, consulte [Creación de una especificación de plantilla con plantillas vinculadas](template-specs-create-linked.md).
