---
title: Azure Service Fabric - Configuración de credenciales de repositorio de contenedor
description: Configuración de credenciales de repositorio para descargar imágenes desde el registro de contenedor
ms.topic: conceptual
ms.date: 12/09/2019
ms.custom: sfrev
ms.openlocfilehash: 142ede6fcc59063d83854712a966a90c7472923b
ms.sourcegitcommit: 9c262672c388440810464bb7f8bcc9a5c48fa326
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 09/03/2020
ms.locfileid: "89421431"
---
# <a name="configure-repository-credentials-for-your-application-to-download-container-images"></a>Configuración de credenciales de repositorio para que la aplicación descargue imágenes de contenedor

Configure la autenticación del registro de contenedor mediante la adición de `RepositoryCredentials` a la sección `ContainerHostPolicies` del manifiesto de aplicación. Agregue la cuenta y contraseña del registro de contenedor (*myregistry.azurecr.io*), lo que permite al servicio descargar la imagen del contenedor del repositorio.

```xml
<ServiceManifestImport>
    ...
    <Policies>
        <ContainerHostPolicies CodePackageRef="Code">
            <RepositoryCredentials AccountName="myregistry" Password="=P==/==/=8=/=+u4lyOB=+=nWzEeRfF=" PasswordEncrypted="false"/>
            <PortBinding ContainerPort="80" EndpointRef="Guest1TypeEndpoint"/>
        </ContainerHostPolicies>
    </Policies>
    ...
</ServiceManifestImport>
```

Se recomienda cifrar la contraseña del repositorio con un certificado de cifrado que se implemente en todos los nodos del clúster. Cuando Service Fabric implementa el paquete de servicio en el clúster, el certificado de cifrado se utiliza para descifrar el texto cifrado. El cmdlet Invoke-ServiceFabricEncryptText se usa para crear el texto cifrado para la contraseña, que se agrega al archivo ApplicationManifest.xml.
Vea [Administración de secretos](service-fabric-application-secret-management.md) para más información sobre los certificados y la semántica de cifrado.

## <a name="configure-cluster-wide-credentials"></a>Configuración de las credenciales en todo el clúster

Service Fabric permite configurar credenciales en todo el clúster que las aplicaciones pueden usar como credenciales del repositorio predeterminadas.

Para habilitar o deshabilitar esta característica, hay que agregar el atributo `UseDefaultRepositoryCredentials` a `ContainerHostPolicies` en ApplicationManifest.xml con un valor de `true` o `false`.

```xml
<ServiceManifestImport>
    ...
    <Policies>
        <ContainerHostPolicies CodePackageRef="Code" UseDefaultRepositoryCredentials="true">
            <PortBinding ContainerPort="80" EndpointRef="Guest1TypeEndpoint"/>
        </ContainerHostPolicies>
    </Policies>
    ...
</ServiceManifestImport>
```

Después, Service Fabric usa las credenciales del repositorio predeterminadas, que se pueden especificar en el archivo ClusterManifest de la sección `Hosting`.  Si `UseDefaultRepositoryCredentials` es `true`, Service Fabric lee los valores siguientes de ClusterManifest:

* DefaultContainerRepositoryAccountName (cadena)
* DefaultContainerRepositoryPassword (cadena)
* IsDefaultContainerRepositoryPasswordEncrypted (valor booleano)
* DefaultContainerRepositoryPasswordType (cadena)

Este es un ejemplo de lo que se puede agregar dentro de la sección `Hosting` en el archivo ClusterManifestTemplate.json. La sección `Hosting` puede agregarse al crear el clúster o posteriormente en una actualización de la configuración. Para obtener más información, consulte [Personalización de la configuración de un clúster de Service Fabric](service-fabric-cluster-fabric-settings.md) y [Administración de los secretos en aplicaciones de Azure Service Fabric](service-fabric-application-secret-management.md).

```json
"fabricSettings": [
    ...,
    {
        "name": "Hosting",
        "parameters": [
          {
            "name": "EndpointProviderEnabled",
            "value": "true"
          },
          {
            "name": "DefaultContainerRepositoryAccountName",
            "value": "someusername"
          },
          {
            "name": "DefaultContainerRepositoryPassword",
            "value": "somepassword"
          },
          {
            "name": "IsDefaultContainerRepositoryPasswordEncrypted",
            "value": "false"
          },
          {
            "name": "DefaultContainerRepositoryPasswordType",
            "value": "PlainText"
          },
          {
        "name": "DefaultMSIEndpointForTokenAuthentication",
        "value": "URI"
          }
        ]
      },
]
```

## <a name="use-tokens-as-registry-credentials"></a>Uso de tokens como credenciales del registro

Service Fabric permite usar tokens como credenciales para descargar imágenes relativas a los contenedores.  Esta característica aprovecha la *identidad administrada* del conjunto de escalado de máquinas virtuales subyacente para autenticarse en el registro, lo que elimina la necesidad de administrar las credenciales de usuario.  Para más información, consulte [Identidades administradas para los recursos de Azure](../active-directory/managed-identities-azure-resources/overview.md).  El uso de esta característica requiere los siguientes pasos:

1. Asegúrese de que la *identidad administrada asignada por el sistema* está habilitada para la máquina virtual.

    ![Azure Portal: Opción para crear una identidad de un conjunto de escalado de máquinas virtuales](./media/configure-container-repository-credentials/configure-container-repository-credentials-acr-iam.png)

2. Conceda al conjunto de escalado de máquinas virtuales los permisos necesarios para extraer o leer imágenes del registro. En la hoja Access Control (IAM) de Azure Container Registry de Azure Portal, agregue una *asignación de roles*  para la máquina virtual:

    ![Agregar una entidad de seguridad de máquina virtual a ACR](./media/configure-container-repository-credentials/configure-container-repository-credentials-vmss-identity.png)

3. A continuación, modifique el manifiesto de aplicación. En la sección `ContainerHostPolicies`, agregue el atributo `‘UseTokenAuthenticationCredentials=”true”`.

    ```xml
      <ServiceManifestImport>
          <ServiceManifestRef ServiceManifestName="NodeServicePackage" ServiceManifestVersion="1.0"/>
      <Policies>
        <ContainerHostPolicies CodePackageRef="NodeService.Code" Isolation="process" UseTokenAuthenticationCredentials="true">
          <PortBinding ContainerPort="8905" EndpointRef="Endpoint1"/>
        </ContainerHostPolicies>
        <ResourceGovernancePolicy CodePackageRef="NodeService.Code" MemoryInMB="256"/>
      </Policies>
      </ServiceManifestImport>
    ```

    > [!NOTE]
    > Si la marca `UseDefaultRepositoryCredentials` está establecida en true cuando `UseTokenAuthenticationCredentials` es true, se producirá un error durante la implementación.

### <a name="using-token-credentials-outside-of-azure-global-cloud"></a>Uso de credenciales de token fuera de la nube global de Azure

Al usar credenciales de registro basadas en tokens, Service Fabric captura un token en nombre de la máquina virtual para presentarlo a ACR. De manera predeterminada, Service Fabric solicita un token cuyo público es el punto de conexión de nube de Azure global. Si va a realizar implementaciones en otra instancia de nube, como Azure Germany o Azure Government, deberá invalidar el valor predeterminado del parámetro `DefaultMSIEndpointForTokenAuthentication`. Si no va a realizar la implementación en un entorno especial, no invalide este parámetro. Si realizará la implementación, reemplazará el valor predeterminado, que es

```
http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.core.windows.net/
```

con el punto de conexión de recursos adecuado para su entorno. Por ejemplo, para [Azure Germany](https://docs.microsoft.com/azure/germany/germany-developer-guide#endpoint-mapping), la invalidación sería 

```json
{
    "name": "DefaultMSIEndpointForTokenAuthentication",
    "value": "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.core.cloudapi.de/"
}
```

[Más información sobre la captura de tokens de conjunto de escalado de máquinas virtuales](https://docs.microsoft.com/azure/active-directory/managed-identities-azure-resources/how-to-use-vm-token).

## <a name="next-steps"></a>Pasos siguientes

* Obtenga más información sobre la [autenticación del registro de contenedor](../container-registry/container-registry-authentication.md).
