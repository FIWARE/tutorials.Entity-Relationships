[![FIWARE Banner](https://fiware.github.io/tutorials.Entity-Relationships/img/fiware.png)](https://www.fiware.org/developers)

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://github.com/FIWARE/catalogue/blob/master/core/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Entity-Relationships.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://nexus.lab.fiware.org/repository/raw/public/badges/stackoverflow/fiware.svg)](https://stackoverflow.com/questions/tagged/fiware)
[![NGSI v2](https://img.shields.io/badge/NGSI-v2-5dc0cf.svg)](https://fiware-ges.github.io/orion/api/v2/stable/) <br/>
[![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

Este tutorial enseña a los usuarios de FIWARE acerca de los comandos por lotes (batch processing) y las relaciones de entidad. El tutorial se basa en los datos creados en el anterior [ejemplo del buscador de tiendas](https://github.com/FIWARE/tutorials.Getting-Started/blob/master/README.es.md), agrega y
asocia una serie de entidades de datos relacionadas para crear un sistema sencillo de gestión inventarios.

A lo largo de este tutorial se utilizan comandos [cUrl](https://ec.haxx.se/), pero también está disponible como
[documentación de Postman](https://fiware.github.io/tutorials.Entity-Relationships/).

[![Ejecutar en Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/0671934f64958d3200b3)

-   このチュートリアルは[日本語](https://github.com/FIWARE/tutorials.Entity-Relationships/blob/master/README.ja.md)でも
    ご覧いただけます。<br/>🇪n This tutorial is also available in [english](README.md)

## Contenido

<details>
<summary><strong>Detalle</strong></summary>

-   [Comprensión de entidades y relaciones](#comprension-de-entidades-y-relaciones)
    -   [Entidades en un sistema de gestión de inventarios](#entidades-en-un-sistema-de-gestion-de-inventarios)
-   [Arquitectura](#arquitectura)
-   [Pre requisitos](#pre-requisitos)
    -   [Docker y Docker Compose](#docker-y-docker-compose)
    -   [Cygwin para Windows](#cygwin-para-windows)
-   [Inicio](#inicio)
-   [Creación y asociación de entidades de datos](#creacion-y-asociacion-de-entidades-de-datos)
    -   [Creación de varias entidades a la vez](#creacion-de-varias-entidades-a-la-vez)
    -   [Creación de una Relación de uno-a-muchos](#creacion-de-una-relacion-de-uno-a-muchos)
    -   [Lectura de una Relación de Clave Foránea](#lectura-de-una-relacion-de-clave-foranea)
        -   [Lectura desde la Entidad Hijo a la Entidad Padre](#lectura-desde-la-entidad-hijo-a-la-entidad-padre)
        -   [Lectura desde la Entidad Padre a la Entidad Hijo](#lectura-desde-la-entidad-padre-a-la-entidad-hijo)
    -   [Creando Relaciones de muchos-a-muchos](#creando-relaciones-de-muchos-a-muchos)
    -   [Leyendo desde una tabla puente](#leyendo-desde-una-tabla-puente)
    -   [Integridad de los datos](#Integridad-de-los-datos)
-   [Próximos pasos](#proximos-pasos)

</details>

# Comprensión de entidades y relaciones

En la plataforma FIWARE, el contexto de una entidad representa el estado de un objeto físico o conceptual que
existe en el mundo real.

## Entidades en un sistema de gestión de inventarios

Para un sistema simple de gestión de inventarios, sólo necesitaremos cuatro tipos de entidades. La relación entre nuestras entidades se define a continuación:

![](https://fiware.github.io/tutorials.Entity-Relationships/img/entities.png)

-   Una tinda (store) es un edificio concreto (de ladrillos) del mundo real. Entidades **Store** pueden tener propiedades como:
    -   Nombre de la tienda, por ejemplo: "Checkpoint Markt"
    -   Una dirección, por ejemplo: "Friedrichstraße 44, 10969 Kreuzberg, Berlin"
    -   Una ubicación física, por ejemplo: _52.5075 N, 13.3903 E_
-   Un estante (shelf) es una entidad del mundo real para guardar los objetos que queremos vender. Cada entidad **Shelf** puede tener propiedades como:
    -   Nombre del estante, por ejemplo: "Wall Unit"
    -   Una ubicación física, por ejemplo: _52.5075 N, 13.3903 E_
    -   Una capacidad máxima
    -   Una asociación a la store (tienda) a la que pertenece
-   Un producto (product) es algo que se desea vender - es un objeto conceptual. Entidades **Product** pueden tener propiedades como:
    -   Nombre del producto, por ejemplo "Vodka"
    -   Su precio, por ejemplo: 13.99 Euros
    -   Un tamaño, por ejemplo: Pequeño
-   Un artículo de inventario (inventory) es otra entidad conceptual, utilizada para asociar productos, tiendas, estantes y objetos físicos.
    Entidades **Inventory Item** pueden tener propiedades como:
    -   Una asociación con el producto que se vende
    -   Una asociación a la tienda en la que se vende el producto
    -   Una asociación con el estante donde se exhibe el producto
    -   Un recuento de la cantidad de producto disponible en el almacén
    -   Un recuento de la cantidad de producto disponible en el estante

Como puede ver, cada una de las entidades definidas anteriormente contienen propiedades que pueden cambiar. Un producto podría
cambiar su precio, se podrían vender las existencias y reducir el número de existencias en las estanterías, etc.

> **Nota** este tutorial utiliza el siguiente estilo tipográfico:
>
> -   Los tipos de entidades se se escriben con **texto en negrita**
> -   Los atributos de los datos se escriben en `texto de espaciado fijo`
> -   Los artículos en el mundo real es escriben en texto simple, sin estilo
>
> Por lo tanto, una tienda en el mundo real está representada en los datos de contexto por una entidad **Store**, y un estante del mundo
> real encontrado en una tienda está representado por una entidad **Estante** con un atributo `refStore`.

# Arquitectura

Esta aplicación sólo hará uso de un componente FIWARE - el
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) (Corredor de Contexto de Orión). El uso del Orion Context Broker es suficiente
para una aplicación para calificar como _"Powered by FIWARE"_.

Actualmente, el Orion Context Broker depende de la tecnología de código abierto [MongoDB](https://www.mongodb.com/) para mantener
la persistencia de los datos de contexto que contiene. Por lo tanto, la arquitectura consistirá en dos elementos:

-   El [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) recibirá las solicitudes utilizando
    [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
-   La base de datos subyacente [MongoDB](https://www.mongodb.com/):
    -   Utilizado por el Orion Context Broker para guardar información de datos de contexto como entidades de datos, suscripciones e inscripciones

Dado que todas las interacciones entre los dos elementos se inician mediante solicitudes HTTP, las entidades pueden ser aisladas en contenedores y corridas desde puertos expuestos.

![](https://fiware.github.io/tutorials.Entity-Relationships/img/architecture.png)

La información de configuración necesaria se puede ver en la sección de servicios del archivo asociado `docker-compose.yml`:

```yaml
orion:
    image: fiware/orion:latest
    hostname: orion
    container_name: fiware-orion
    depends_on:
        - mongo-db
    networks:
        - default
    expose:
        - "1026"
    ports:
        - "1026:1026"
    command: -dbhost mongo-db -logLevel DEBUG
```

```yaml
mongo-db:
    image: mongo:3.6
    hostname: mongo-db
    container_name: db-mongo
    expose:
        - "27017"
    ports:
        - "27017:27017"
    networks:
        - default
    command: --bind_ip_all --smallfiles
```

Ambos contenedores residen en la misma red - Orion Context Broker escucha en el puerto `1026` y MongoDB lo hace por defecto en el puerto `27017`. Ambos contenedores también están exponiendo los mismos puertos externamente - esto es puramente para
el acceso al tutorial - para que cUrl o Postman puedan acceder a ellos sin ser parte de la misma red.  
La inicialización línea de comandos es autoexplicativa.

# Pre requisitos

## Docker y Docker Compose

Para mantener las cosas simples ambos componentes se ejecutarán usando [Docker](https://www.docker.com). **Docker** es una tecnología de contenedor que permite aislar diferentes componentes en sus respectivos entornos.
-   Para instalar Docker en Windows siga las instrucciones [aquí](https://docs.docker.com/docker-for-windows/)
-   Para instalar Docker en Mac siga las instrucciones [aquí](https://docs.docker.com/docker-for-mac/)
-   Para instalar Docker en Linux siga las instrucciones [aquí](https://docs.docker.com/install/)

**Docker Compose** es una herramienta para definir y ejecutar aplicaciones Docker multi-contenedores. Un
[archivo YAML](https://raw.githubusercontent.com/Fiware/tutorials.Entity-Relationships/master/docker-compose.yml) se utiliza para configurar los servicios necesarios para la aplicación. Esto significa que todos los servicios de los contenedores pueden ser ejecutados en un solo comando. Docker Compose está instalado por defecto como parte de Docker para Windows y Docker para Mac, sin embargo los usuarios de Linux tendrán que seguir las instrucciones que se encuentran [aquí](https://docs.docker.com/compose/install/)

Puede comprobar sus versiones actuales de **Docker** y **Docker Compose** usando los siguientes comandos:

```console
docker-compose -v
docker version
```

Por favor, asegúrese de que esté usando la versión 18.03 o superior de Docker y la versión 1.21 o superior de Docker Compose y actualice si es necesario.

## Cygwin para Windows

Iniciaremos nuestros servicios usando un simple script de Bash. Los usuarios de Windows deben descargar [cygwin](http://www.cygwin.com/)
para proporcionar una funcionalidad de línea de comandos similar a la de una distribución de Linux en Windows.

# Inicio

Todos los servicios pueden ser inicializados desde la línea de comando ejecutando el script Bash proporcionado en el repositorio
[services](https://github.com/FIWARE/tutorials.Entity-Relationships/blob/master/services). Por favor, clone el repositorio y cree las imágenes necesarias ejecutando los comandos como se muestra:

```console
git clone https://github.com/FIWARE/tutorials.Entity-Relationships.git
cd tutorials.Entity-Relationships

./services start
```

Este comando también importará datos iniciales del ejemplo anterior
[Tutorial del Buscador de Tiendas](https://github.com/FIWARE/tutorials.Getting-Started/blob/master/README.es.md).


> :information_source: **Nota:** Si quiere limpiar y empezar de nuevo puedes hacerlo con el siguiente comando:
>
> ```console
> ./services stop
> ```

# Creación y asociación de entidades de datos

## Creación de varias entidades a la vez

In the previous tutorial, we created each **Store** entity individually,

Lets create five shelf units at the same time. This request uses the convenience batch processing endpoint to create
five shelf entities. Batch processing uses the `/v2/op/update` endpoint with a payload with two attributes -
`actionType=APPEND` means we will overwrite existing entities if they exist whereas the `entities` attribute holds an
array of entities we wish to update.

To differenciate **Shelf** Entities from **Store** Entities, each shelf has been assigned `type=Shelf`. Real-world
properties such as `name` and `location` have been added as properties to each shelf.

#### :one: Request:

```console
curl -iX POST \
  'http://localhost:1026/v2/op/update' \
  -H 'Content-Type: application/json' \
  -d '{
  "actionType":"APPEND",
  "entities":[
    {
      "id":"urn:ngsi-ld:Shelf:unit001", "type":"Shelf",
      "location":{
        "type":"geo:json", "value":{ "type":"Point","coordinates":[13.3986112, 52.554699]}
      },
      "name":{
        "type":"Text", "value":"Corner Unit"
      },
      "maxCapacity":{
        "type":"Integer", "value":50
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit002", "type":"Shelf",
      "location":{
        "type":"geo:json","value":{"type":"Point","coordinates":[13.3987221, 52.5546640]}
      },
      "name":{
        "type":"Text", "value":"Wall Unit 1"
      },
      "maxCapacity":{
        "type":"Integer", "value":100
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit003", "type":"Shelf",
      "location":{
        "type":"geo:json", "value":{"type":"Point","coordinates":[13.3987221, 52.5546640]}
      },
      "name":{
        "type":"Text", "value":"Wall Unit 2"
      },
      "maxCapacity":{
        "type":"Integer", "value":100
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit004", "type":"Shelf",
      "location":{
        "type":"geo:json", "value":{"type":"Point","coordinates":[13.390311, 52.507522]}
      },
      "name":{
        "type":"Text", "value":"Corner Unit"
      },
      "maxCapacity":{
        "type":"Integer", "value":50
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit005", "type":"Shelf",
      "location":{
        "type":"geo:json","value":{"type":"Point","coordinates":[13.390309, 52.50751]}
      },
      "name":{
        "type":"Text", "value":"Long Wall Unit"
      },
      "maxCapacity":{
        "type":"Integer", "value":200
      }
    }
  ]
}'
```

Similarly, we can create a series of **Product** entities by using the `type=Product`.

#### :two: Request:

```console
curl -iX POST \
  'http://localhost:1026/v2/op/update' \
  -H 'Content-Type: application/json' \
  -d '{
  "actionType":"APPEND",
  "entities":[
    {
      "id":"urn:ngsi-ld:Product:001", "type":"Product",
      "name":{
        "type":"Text", "value":"Beer"
      },
      "size":{
        "type":"Text", "value": "S"
      },
      "price":{
        "type":"Integer", "value": 99
      }
    },
    {
      "id":"urn:ngsi-ld:Product:002", "type":"Product",
      "name":{
        "type":"Text", "value":"Red Wine"
      },
      "size":{
        "type":"Text", "value": "M"
      },
      "price":{
        "type":"Integer", "value": 1099
      }
    },
    {
      "id":"urn:ngsi-ld:Product:003", "type":"Product",
      "name":{
        "type":"Text", "value":"White Wine"
      },
      "size":{
        "type":"Text", "value": "M"
      },
      "price":{
        "type":"Integer", "value": 1499
      }
    },
    {
      "id":"urn:ngsi-ld:Product:004", "type":"Product",
      "name":{
        "type":"Text", "value":"Vodka"
      },
      "size":{
        "type":"Text", "value": "XL"
      },
      "price":{
        "type":"Integer", "value": 5000
      }
    }
  ]
}'
```

In both cases we have encoded each entity `id` according to the NGSI-LD
[specification](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.01.01_60/gs_CIM009v010101p.pdf) - the proposal
is that each `id` is a URN follows a standard format: `urn:ngsi-ld:<entity-type>:<entity-id>`. This will mean that every
`id` in the system will be unique.

Shelf information can be requested by making a GET request on the `/v2/entities` endpoint. For example to return the
context data of the **Shelf** entity with the `id=urn:ngsi-ld:Shelf:unit001`.

#### :three: Request:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=keyValues'
```

#### Response:

```json
{
    "id": "urn:ngsi-ld:Shelf:unit001",
    "type": "Shelf",
    "location": {
        "type": "Point",
        "coordinates": [13.3986112, 52.554699]
    },
    "maxCapacity": 50,
    "name": "Corner Unit"
}
```

As you can see there are currently three additional property attributes present `location`, `maxCapacity` and `name`

## Creación de una Relación de uno-a-muchos

In databases, foreign keys are often used to designate a one-to-many relationship - for example every shelf is found in
a single store and a single store can hold many shelving units. In order to remember this information we need to add an
association relationship similar to a foreign key. Batch processing can again be used to amend the existing the
**Shelf** entities to add a `refStore` attribute holding the relationship to each store. According to the FIWARE Data
Modelling Guidelines on
[linked data](https://fiware-datamodels.readthedocs.io/en/latest/guidelines/index.html#modelling-linked-data), when an
entity attribute is used as a link to other entities it should be named with the prefix `ref` plus the name of the
target (linked) entity type.

The value of the `refStore` attribute corresponds to a URN associated to a **Store** entity itself.

The URN follows a standard format: `urn:ngsi-ld:<entity-type>:<entity-id>`

#### :four: Request:

The following request associates three shelves to `urn:ngsi-ld:Store:001` and two shelves to `urn:ngsi-ld:Store:002`

```console
curl -iX POST \
  'http://localhost:1026/v2/op/update' \
  -H 'Content-Type: application/json' \
  -d '{
  "actionType":"APPEND",
  "entities":[
    {
      "id":"urn:ngsi-ld:Shelf:unit001", "type":"Shelf",
      "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit002", "type":"Shelf",
      "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit003", "type":"Shelf",
      "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit004", "type":"Shelf",
      "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:002"
      }
    },
    {
      "id":"urn:ngsi-ld:Shelf:unit005", "type":"Shelf",
      "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:002"
      }
    }
  ]
}'
```

Now when the shelf information is requested again, the response has changed and includes a new property `refStore`,
which has been added in the previous step.

#### :five: Request:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=keyValues'
```

#### Response:

The updated response including the `refStore` attribute is shown below:

```json
{
    "id": "urn:ngsi-ld:Shelf:unit001",
    "type": "Shelf",
    "location": {
        "type": "Point",
        "coordinates": [13.3986112, 52.554699]
    },
    "maxCapacity": 50,
    "name": "Corner Unit",
    "refStore": "urn:ngsi-ld:Store:001"
}
```

## Lectura de una Relación de Clave Foránea

### Lectura desde la Entidad Hijo a la Entidad Padre

We can also make a request to retrieve the `refStore` attribute relationship information from a known **Shelf** entity
by using the `options=values` setting

#### :six: Request:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=values&attrs=refStore'
```

#### Response:

```json
["urn:ngsi-ld:Store:001"]
```

This can be interpreted as "I am related to the **Store** entity with the `id=urn:ngsi-ld:Store:001`"

### Lectura desde la Entidad Padre a la Entidad Hijo

Reading from a parent to a child can be done using the `options=count` setting

#### :seven: Request:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=count&attrs=type&type=Shelf'
```

This request is asking for the `id` of all **Shelf** entities associated to the URN `urn:ngsi-ld:Store:001`, the
response is a JSON array as shown.

#### Response:

```json
[
    {
        "id": "urn:ngsi-ld:Shelf:unit001",
        "type": "Shelf"
    },
    {
        "id": "urn:ngsi-ld:Shelf:unit002",
        "type": "Shelf"
    },
    {
        "id": "urn:ngsi-ld:Shelf:unit003",
        "type": "Shelf"
    }
]
```

In plain English, this can be interpreted as "There are three shelves in `urn:ngsi-ld:Store:001`". The request can be
altered use the `options=values` and `attrs` parameters to return specific properties of the relevant associated
entities. For example the request:

#### :eight: Request:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&type=Shelf&options=values&attrs=name'
```

Can be interpreted as request for _Give me the names of all shelves in `urn:ngsi-ld:Store:001`_.

#### Response:

```json
[["Corner Unit"], ["Wall Unit 1"], ["Wall Unit 2"]]
```

## Creando Relaciones de muchos-a-muchos

Bridge Tables are often used to relate many-to-many relationships. For example, every store will sell a different range
of products, and each product is sold in many different stores.

In order to hold the context information to "place a product onto a shelf in a given store" we will need to create a new
data entity **InventoryItem** which exists to associate data from other entities. It has a foreign key relationship to
the **Store**, **Shelf** and **Product** entities and therefore requires relationship attributes called `refStore`,
`refShelf` and `refProduct`.

Assigning a product to a shelf is simply done by creating an entity holding the relationship information and any other
additional properties (such as `stockCount` and `shelfCount`)

#### :nine: Request:

```console
curl -iX POST \
  'http://localhost:1026/v2/entities' \
  -H 'Content-Type: application/json' \
  -d '{
    "id": "urn:ngsi-ld:InventoryItem:001", "type": "InventoryItem",
    "refStore": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
    },
    "refShelf": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Shelf:unit001"
    },
    "refProduct": {
        "type": "Relationship",
        "value": "urn:ngsi-ld:Product:001"
    },
    "stockCount":{
        "type":"Integer", "value": 10000
    },
    "shelfCount":{
        "type":"Integer", "value": 50
    }
}'
```

## Leyendo desde una tabla puente

When reading from a bridge table entity, the `type` of the entity must be known.

After creating at least one **InventoryItem** entity we can query _Which products are sold in `urn:ngsi-ld:Store:001`?_
by making the following request

#### :one::zero: Request:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=values&attrs=refProduct&type=InventoryItem'
```

#### Response:

```json
[["urn:ngsi-ld:Product:prod001"]]
```

Similarly we can request _Which stores are selling `urn:ngsi-ld:Product:001`?_ by altering the request as shown:

#### :one::one: Request:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refProduct==urn:ngsi-ld:Product:001&options=values&attrs=refStore&type=InventoryItem'
```

#### Response:

```json
[["urn:ngsi-ld:Store:001"]]
```

## Integridad de los datos

Context data relationships should only be set up and maintained between entities that exist - in other words the URN
`urn:ngsi-ld:<entity-type>:<entity-id>` should link to another existing entity within the context. Therefore we must
take care when deleting an entity that no dangling references remain. Imagine `urn:ngsi-ld:Store:001` is deleted - what
should happen to the associated the **Shelf** entities?

It is possible to make a request to see if any remaining entity relationship exists prior to deletion by making a
request as follows

#### :one::two: Request:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=count&attrs=type'
```

#### :one::three: Request:

The response lists a series of **Shelf** and **InventoryItem** entities - there are no **Product** entities since there
is no direct relationship between product and store.

```json
[
    {
        "id": "urn:ngsi-ld:Shelf:unit001",
        "type": "Shelf"
    },
    {
        "id": "urn:ngsi-ld:Shelf:unit002",
        "type": "Shelf"
    },
    {
        "id": "urn:ngsi-ld:Shelf:unit003",
        "type": "Shelf"
    },
    {
        "id": "urn:ngsi-ld:InventoryItem:001",
        "type": "InventoryItem"
    }
]
```

If this request returns an empty array, the entity has no associates.

# Próximos pasos

Want to learn how to add more complexity to your application by adding advanced features? You can find out by reading
the other [tutorials in this series](https://fiware-tutorials.rtfd.io)

---

## Licencia

[MIT](LICENSE) © 2018-2020 FIWARE Foundation e.V.
