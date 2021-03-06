---
title: 'Inicio rápido: Creación de un índice de búsqueda en .NET'
titleSuffix: Azure Cognitive Search
description: En este inicio rápido de C#, se explica cómo crear un índice, cargar datos y ejecutar consultas mediante la biblioteca cliente Azure.Search.Documents.
manager: nitinme
author: HeidiSteen
ms.author: heidist
ms.service: cognitive-search
ms.devlang: dotnet
ms.topic: quickstart
ms.date: 10/05/2020
ms.custom: devx-track-csharp
ms.openlocfilehash: 53deb7dc853de969ad6b6679ee728a3f132b6309
ms.sourcegitcommit: 6a4687b86b7aabaeb6aacdfa6c2a1229073254de
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 10/06/2020
ms.locfileid: "91759104"
---
# <a name="quickstart-create-a-search-index-using-the-azuresearchdocuments-client-library"></a>Inicio rápido: Creación de un índice de búsqueda mediante la biblioteca cliente Azure.Search.Documents

Use la nueva [biblioteca cliente Azure.Search.Documents (versión 11)](/dotnet/api/overview/azure/search.documents-readme) para crear una aplicación de consola de .NET Core en C# que cree, cargue y consulte un índice de búsqueda.

[Descargue el código fuente](https://github.com/Azure-Samples/azure-search-dotnet-samples/tree/master/quickstart/v11) para empezar con un proyecto terminado o siga los pasos especificados en este artículo para crear el suyo propio.

> [!NOTE]
> ¿Busca una versión anterior? Consulte [Creación de un índice de búsqueda con Microsoft.Azure.Search v10](search-get-started-dotnet-v10.md).

## <a name="prerequisites"></a>Requisitos previos

Antes de empezar, tenga preparadas las herramientas y los servicios siguientes:

+ Una cuenta de Azure con una suscripción activa. [Cree una cuenta gratuita](https://azure.microsoft.com/free/).

+ Un servicio de Azure Cognitive Search. [Cree un servicio](search-create-service-portal.md) o [busque uno existente](https://ms.portal.azure.com/#blade/HubsExtension/BrowseResourceBlade/resourceType/Microsoft.Search%2FsearchServices). Puede usar un servicio gratuito para este inicio rápido. 

+ [Visual Studio](https://visualstudio.microsoft.com/downloads/), cualquier edición. El código de ejemplo se probó en la edición gratuita Community de Visual Studio 2019.

+ [Paquete NuGet Azure.Search.Documents](https://www.nuget.org/packages/Azure.Search.Documents/)

## <a name="set-up-your-project"></a>Configuración del proyecto

Ensamble la información de conexión de servicio e inicie Visual Studio para crear un proyecto de aplicación de consola que se pueda ejecutar en .NET Core.

<a name="get-service-info"></a>

### <a name="copy-a-key-and-endpoint"></a>Copia de una clave y un punto de conexión

Las llamadas al servicio requieren un punto de conexión de URL y una clave de acceso en cada solicitud. Como primer paso, busque la clave de API y la dirección URL que se vayan a agregar al proyecto. Deberá especificar ambos valores al crear el cliente en un paso posterior.

1. [Inicie sesión en Azure Portal](https://portal.azure.com/) y en la página **Introducción** del servicio de búsqueda, obtenga la dirección URL. Un punto de conexión de ejemplo podría ser similar a `https://mydemo.search.windows.net`.

2. En **Configuración** > **Claves**, obtenga una clave de administración para los derechos completos en el servicio, esta es necesaria si va a crear o eliminar objetos. Hay dos claves intercambiables una principal y otra secundaria. Puede usar cualquiera de ellas.

   ![Obtención de una clave de acceso y un punto de conexión HTTP](media/search-get-started-postman/get-url-key.png "Obtención de una clave de acceso y un punto de conexión HTTP")

Todas las solicitudes requieren una clave de API en cada solicitud enviada al servicio. Tener una clave válida genera la confianza, solicitud a solicitud, entre la aplicación que envía la solicitud y el servicio que se encarga de ella.

### <a name="install-the-nuget-package"></a>Instalación del paquete NuGet.

Una vez creado el proyecto, agregue la biblioteca de cliente. El [paquete Azure.Search.Documents](https://www.nuget.org/packages/Azure.Search.Documents/) consta de una biblioteca de cliente que proporciona todas las API que se usan para trabajar con un servicio de búsqueda en .NET.

1. En **Herramientas** > **Administrador de paquetes NuGet** seleccione **Administrar paquetes NuGet para la solución....** . 

1. Haga clic en **Examinar**.

1. Busque `Azure.Search.Documents` y seleccione la versión 11.0 ou otra posterior.

1. Haga clic a la derecha en **Instalar**para agregar el ensamblado al proyecto y la solución.

### <a name="create-a-search-client"></a>Creación de un cliente de búsqueda

1. En **Program.cs**, cambie el espacio de nombres a `AzureSearch.SDK.Quickstart.v11` y, a continuación, agregue las siguientes directivas `using`.

   ```csharp
   using Azure;
   using Azure.Search.Documents;
   using Azure.Search.Documents.Indexes;
   using Azure.Search.Documents.Indexes.Models;
   using Azure.Search.Documents.Models;
   ```

1. Cree dos clientes: [SearchIndexClient](/dotnet/api/azure.search.documents.indexes.searchindexclient) crea el índice y [SearchClient](/dotnet/api/azure.search.documents.searchclient) carga y consulta uno existente. Ambos necesitan el punto de conexión de servicio y una clave de API de administración para la autenticación con derechos de creación y eliminación.

   ```csharp
   static void Main(string[] args)
   {
       string serviceName = "<YOUR-SERVICE-NAME>";
       string indexName = "hotels-quickstart-v11";
       string apiKey = "<YOUR-ADMIN-API-KEY>";

       // Create a SearchIndexClient to send create/delete index commands
       Uri serviceEndpoint = new Uri($"https://{serviceName}.search.windows.net/");
       AzureKeyCredential credential = new AzureKeyCredential(apiKey);
       SearchIndexClient idxclient = new SearchIndexClient(serviceEndpoint, credential);

       // Create a SearchClient to load and query documents
       SearchClient srchclient = new SearchClient(serviceEndpoint, indexName, credential);
    ```

## <a name="1---create-an-index"></a>1 - Creación de un índice

En este inicio rápido se crea un índice de hoteles que se cargará con datos de hotel y en el que se ejecutarán consultas. En este paso, defina los campos en el índice. Cada definición de campo incluye un nombre, un tipo de datos y atributos que determinan cómo se usa el campo.

En este ejemplo, se usan métodos sincrónicos de la biblioteca Azure.Search.Documents para simplificar y mejorar la legibilidad. Sin embargo, para los escenarios de producción, debe usar métodos asincrónicos para mantener la aplicación escalable y receptiva. Por ejemplo, usaría [CreateIndexAsync](/dotnet/api/azure.search.documents.indexes.searchindexclient.createindexasync) en lugar de [CreateIndex](/dotnet/api/azure.search.documents.indexes.searchindexclient.createindex).

1. Agregue una definición de clase vacía al proyecto: **Hotel.cs**

1. En **Hotel.cs**, defina la estructura de un documento de hotel.

    ```csharp
    using System;
    using System.Text.Json.Serialization;

    namespace AzureSearch.SDK.Quickstart.v11
    {
        public class Hotel
        {
            [JsonPropertyName("hotelId")]
            public string Id { get; set; }

            [JsonPropertyName("hotelName")]
            public string Name { get; set; }

            [JsonPropertyName("hotelCategory")]
            public string Category { get; set; }

            [JsonPropertyName("baseRate")]
            public Int32 Rate { get; set; }

            [JsonPropertyName("lastRenovationDate")]
            public DateTime Updated { get; set; }
        }
    }
    ```

1. En **Program.cs**, cree un objeto [SearchIndex](/dotnet/api/azure.search.documents.indexes.models.searchindex) y llame al método [CreateIndex](/dotnet/api/azure.search.documents.indexes.searchindexclient.createindex) para expresar el índice en el servicio de búsqueda.

   ```csharp
    // Define an index schema using SearchIndex
    // Create the index using SearchIndexClient
    SearchIndex index = new SearchIndex(indexName)
    {
        Fields =
            {
                new SimpleField("hotelId", SearchFieldDataType.String) { IsKey = true, IsFilterable = true, IsSortable = true },
                new SearchableField("hotelName") { IsFilterable = true, IsSortable = true },
                new SearchableField("hotelCategory") { IsFilterable = true, IsSortable = true },
                new SimpleField("baseRate", SearchFieldDataType.Int32) { IsFilterable = true, IsSortable = true },
                new SimpleField("lastRenovationDate", SearchFieldDataType.DateTimeOffset) { IsFilterable = true, IsSortable = true }
            }
    };

    Console.WriteLine("{0}", "Creating index...\n");
    idxclient.CreateIndex(index);
   ```

Los atributos del campo determinan cómo se utilizan en una aplicación. Por ejemplo, el atributo `IsFilterable` se tiene que asignar a cada campo que admita una expresión de filtro.

En la biblioteca cliente Azure.Search.Documents puede usar [SearchableField](/dotnet/api/azure.search.documents.indexes.models.searchablefield) y [SimpleField](/dotnet/api/azure.search.documents.indexes.models.simplefield) para optimizar las definiciones de campo. Ambos son derivados de un [SearchField](/dotnet/api/azure.search.documents.indexes.models.searchfield) y pueden simplificar el código:

+ `SimpleField` puede ser cualquier tipo de datos, no permite búsquedas (se omite en las consultas de búsqueda de texto completo) y se puede recuperar (no está oculto). Otros atributos están desactivados de forma predeterminada, pero se pueden habilitar. Puede usar un `SimpleField` para los identificadores de documento o los campos que se usan solo en filtros, facetas o perfiles de puntuación. En caso de hacerlo, asegúrese de aplicar los atributos necesarios para el escenario, como `IsKey = true` para un identificador de documento. Para más información, consulte [SimpleFieldAttribute.cs](https://github.com/Azure/azure-sdk-for-net/blob/master/sdk/search/Azure.Search.Documents/src/Indexes/SimpleFieldAttribute.cs) en el código fuente.

+ `SearchableField` debe ser una cadena y siempre permite búsquedas y se puede recuperar. Otros atributos están desactivados de forma predeterminada, pero se pueden habilitar. Ya que este tipo de campo se puede buscar, admite sinónimos y el complemento completo de las propiedades del analizador. Para más información, consulte [SearchableFieldAttribute.cs](https://github.com/Azure/azure-sdk-for-net/blob/master/sdk/search/Azure.Search.Documents/src/Indexes/SearchableFieldAttribute.cs) en el código fuente.

Tanto si usa la API básica de `SearchField`, como uno de los modelos auxiliares, debe habilitar explícitamente los atributos de filtro, faceta y ordenación. Por ejemplo, [IsFilterable](/dotnet/api/azure.search.documents.indexes.models.searchfield.isfilterable), [IsSortable](/dotnet/api/azure.search.documents.indexes.models.searchfield.issortable)y [IsFacetable](/dotnet/api/azure.search.documents.indexes.models.searchfield.isfacetable) se tienen que atribuir explícitamente, como se muestra en el ejemplo anterior. 

<a name="load-documents"></a>

## <a name="2---load-documents"></a>2 - Carga de documentos

Azure Cognitive Search busca en el contenido almacenado en el servicio. En este paso, el usuario cargará los documentos JSON que se ajustan al índice de hotel que acaba de crear.

En Azure Cognitive Search, los documentos de búsqueda son estructuras de datos que se pueden usar como entradas para la indexación y como salidas de las consultas. Tal como se obtienen del origen de datos externo, las entradas de documento pueden ser las filas de una base de datos, los blobs en Blob Storage o documentos JSON en el disco. En este ejemplo vamos a atajar e insertaremos los documentos JSON para cinco hoteles en el propio código. 

Al cargar documentos, tiene que utilizar un objeto [IndexDocumentsBatch](/dotnet/api/azure.search.documents.models.indexdocumentsbatch-1). Un objeto `IndexDocumentsBatch` contiene una colección de [Actions](/dotnet/api/azure.search.documents.models.indexdocumentsbatch-1.actions) (Acciones), cada una de las cuales contiene un documento y una propiedad que indican a Azure Cognitive Search la acción que debe realizar ([carga, combinación, eliminación y mergeOrUpload](search-what-is-data-import.md#indexing-actions)).

1. En **Program.cs**, cree una matriz de documentos y las acciones de índice y pase la matriz a `IndexDocumentsBatch`. Los siguientes documentos se ajustan al índice hotels-quickstart-v11, tal como se define mediante la clase hotel.

    ```csharp
    // Load documents (using a subset of fields for brevity)
    IndexDocumentsBatch<Hotel> batch = IndexDocumentsBatch.Create(
        IndexDocumentsAction.Upload(new Hotel { Id = "78", Name = "Upload Inn", Category = "hotel", Rate = 279, Updated = new DateTime(2018, 3, 1, 7, 0, 0) }),
        IndexDocumentsAction.Upload(new Hotel { Id = "54", Name = "Breakpoint by the Sea", Category = "motel", Rate = 162, Updated = new DateTime(2015, 9, 12, 7, 0, 0) }),
        IndexDocumentsAction.Upload(new Hotel { Id = "39", Name = "Debug Motel", Category = "motel", Rate = 159, Updated = new DateTime(2016, 11, 11, 7, 0, 0) }),
        IndexDocumentsAction.Upload(new Hotel { Id = "48", Name = "NuGet Hotel", Category = "hotel", Rate = 238, Updated = new DateTime(2016, 5, 30, 7, 0, 0) }),
        IndexDocumentsAction.Upload(new Hotel { Id = "12", Name = "Renovated Ranch", Category = "motel", Rate = 149, Updated = new DateTime(2020, 1, 24, 7, 0, 0) }));

    IndexDocumentsOptions idxoptions = new IndexDocumentsOptions { ThrowOnAnyError = true };

    Console.WriteLine("{0}", "Loading index...\n");
    srchclient.IndexDocuments(batch, idxoptions);
    ```

    Una vez inicializado el objeto [IndexDocumentsBatch](/dotnet/api/azure.search.documents.models.indexdocumentsbatch-1), puede enviarlo al índice mediante una llamada a [IndexDocuments](/dotnet/api/azure.search.documents.searchclient.indexdocuments) en el objeto [SearchClient](/dotnet/api/azure.search.documents.searchclient).

1. Dado que se trata de una aplicación de consola que ejecuta todos los comandos secuencialmente, agregue un tiempo de espera de 2 segundos entre la indexación y las consultas.

    ```csharp
    // Wait 2 seconds for indexing to complete before starting queries (for demo and console-app purposes only)
    Console.WriteLine("Waiting for indexing...\n");
    System.Threading.Thread.Sleep(2000);
    ```

    El retraso de 2 segundos compensa por la indexación, que es asincrónica, por lo que todos los documentos se pueden indexar antes de que se ejecutan las consultas. La codificación en un retraso solo suele ser necesaria en las pruebas, demostraciones y aplicaciones de ejemplo.

## <a name="3---search-an-index"></a>3 - Búsqueda en un índice

Puede obtener resultados de consulta tan pronto como se indexe el primer documento, pero las pruebas reales del índice deben esperar hasta que todos los documentos estén indexados.

Esta sección agrega dos funcionalidades: la lógica de consulta y los resultados. En el caso de las consultas, use el método [Search](/dotnet/api/azure.search.documents.searchclient.search). Este método admite texto de búsqueda (la cadena de consulta) además de otras [opciones](/dotnet/api/azure.search.documents.searchoptions).

La clase [SearchResults](/dotnet/api/azure.search.documents.models.searchresults-1) representa los resultados.

1. En **Program.cs**, cree un método WriteDocuments que imprime los resultados de la búsqueda en la consola.

    ```csharp
    private static void WriteDocuments(SearchResults<Hotel> searchResults)
    {
        foreach (SearchResult<Hotel> response in searchResults.GetResults())
        {
            Hotel doc = response.Document;
            var score = response.Score;
            Console.WriteLine($"Name: {doc.Name}, Type: {doc.Category}, Rate: {doc.Rate}, Last-update: {doc.Updated}, Score: {score}");
        }

        Console.WriteLine();
    }
    ```

1. Cree un método RunQueries para ejecutar consultas y devolver los resultados. Los resultados son objetos de Hotel.

    ```csharp
    private static void RunQueries(SearchClient srchclient)
    {
        SearchOptions options;
        SearchResults<Hotel> response;

        Console.WriteLine("Query #1: Search on the term 'motel' and list the relevance score for each match...\n");

        options = new SearchOptions()
        {
            Filter = "",
            OrderBy = { "" }
        };

        response = srchclient.Search<Hotel>("motel", options);
        WriteDocuments(response);

        Console.WriteLine("Query #2: Find hotels where 'type' equals hotel...\n");

        options = new SearchOptions()
        {
            Filter = "hotelCategory eq 'hotel'",
        };

        response = srchclient.Search<Hotel>("*", options);
        WriteDocuments(response);

        Console.WriteLine("Query #3: Filter on rates less than $200 and sort by when the hotel was last updated...\n");

        options = new SearchOptions()
        {
            Filter = "baseRate lt 200",
            OrderBy = { "lastRenovationDate desc" }
        };

        response = srchclient.Search<Hotel>("*", options);
        WriteDocuments(response);
    }
    ```

1. Agregue RunQueries a `Main()`.

    ```csharp
    Console.WriteLine("Starting queries...\n");
    RunQueries(srchclient);
    ```

Este ejemplo muestra las dos [formas de hacer coincidir los términos en una consulta](search-query-overview.md#types-of-queries): búsqueda de texto completo y de filtros.

+ Las consultas de búsqueda de texto completo realiza búsquedas de uno o más términos en los campos en los que se permiten búsquedas en el índice. La primera consulta es una búsqueda de texto completo. Las búsquedas de texto completo generan puntuaciones de relevancia que se usan para clasificar los resultados.

+ Un filtro es una expresión booleana que se evalúa en campos [IsFilterable](/dotnet/api/azure.search.documents.indexes.models.searchfield.isfilterable) en un índice. Las consultas de filtro incluyen o excluyen valores. Como tal, no hay puntuaciones de relevancia asociadas a una consulta de filtro. Las dos últimas consultas muestran la búsqueda de filtros.

Puede usar la búsqueda de texto completo y los filtros conjuntamente o por separado.

Las búsquedas y los filtros se realizan mediante el método [SearchClient.Search](/dotnet/api/azure.search.documents.searchclient.search). Se puede pasar una consulta de búsqueda en la cadena `searchText`, mientras que una expresión de filtro se puede pasar en la propiedad [Filter](/dotnet/api/azure.search.documents.searchoptions.filter) de la clase [SearchOptions](/dotnet/api/azure.search.documents.searchoptions). Para filtrar sin buscar, solo tiene que pasar `"*"` al parámetro `searchText` del método [Search](/dotnet/api/azure.search.documents.searchclient.search). Para buscar sin filtrar, deje la propiedad `Filter` sin establecer, o no pase ninguna instancia de `SearchOptions`.

## <a name="run-the-program"></a>Ejecución del programa

Presione F5 para recompilar la aplicación y ejecutar el programa en su totalidad. 

La salida incluye mensajes de [Console.WriteLIne](/dotnet/api/system.console.writeline), con la incorporación de la información de la consulta y los resultados.

## <a name="clean-up-resources"></a>Limpieza de recursos

Cuando trabaje con su propia suscripción, es una buena idea al final de un proyecto identificar si todavía se necesitan los recursos que ha creado. Los recursos que se dejan en ejecución pueden costarle mucho dinero. Puede eliminar los recursos de forma individual o eliminar el grupo de recursos para eliminar todo el conjunto de recursos.

Puede encontrar y administrar recursos en el portal, mediante el vínculo **Todos los recursos** o **Grupos de recursos** en el panel de navegación izquierdo.

Si está usando un servicio gratuito, recuerde que está limitado a tres índices, indexadores y orígenes de datos. Puede eliminar elementos individuales en el portal para mantenerse por debajo del límite. 

## <a name="next-steps"></a>Pasos siguientes

En este inicio rápido de C# ha realizado un conjunto de tareas para crear un índice, cargar documentos en él y ejecutar consultas. En diferentes fases, hemos atajado para simplificar el código y mejorar la legibilidad y la comprensión. Si ya se ha familiarizado con los conceptos básicos, se recomienda el siguiente artículo para realizar una exploración de conceptos y enfoques alternativos que le permitirán profundizar en sus conocimientos. 

> [!div class="nextstepaction"]
> [Procedimientos de desarrollo en .NET](search-howto-dotnet-sdk.md)

¿Quiere optimizar y ahorrar en el gasto en la nube?

> [!div class="nextstepaction"]
> [Comience a analizar los costos con Cost Management](../cost-management-billing/costs/quick-acm-cost-analysis.md?WT.mc_id=costmanagementcontent_docsacmhorizontal_-inproduct-learn)