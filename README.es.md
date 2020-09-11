[![FIWARE Banner](https://fiware.github.io/tutorials.Entity-Relationships/img/fiware.png)](https://www.fiware.org/developers)
[![NGSI v2](https://img.shields.io/badge/NGSI-v2-5dc0cf.svg)](https://fiware-ges.github.io/orion/api/v2/stable/)

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://github.com/FIWARE/catalogue/blob/master/core/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Entity-Relationships.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://nexus.lab.fiware.org/repository/raw/public/badges/stackoverflow/fiware.svg)](https://stackoverflow.com/questions/tagged/fiware)
 <br/>
[![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

Este tutorial enseña a los usuarios de FIWARE acerca de los comandos por lotes (batch processing) y las relaciones de entidad. El tutorial se basa en los datos creados en el ejemplo anterior [buscador de tiendas](https://github.com/FIWARE/tutorials.Getting-Started/blob/master/README.es.md), agrega y
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
> -   Los tipos de entidades se escriben con **texto en negrita**
> -   Los atributos de los datos se escriben en `texto de espaciado fijo`
> -   Los artículos en el mundo real es escriben en texto simple, sin estilo
>
> Por lo tanto, una tienda en el mundo real está representada en los datos de contexto por una entidad **Store**, y un estante del mundo
> real encontrado en una tienda está representado por una entidad **Shelf** con un atributo `refStore`.

# Arquitectura

Esta aplicación sólo hará uso de un componente FIWARE - el
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) (Corredor de Contexto de Orión). El uso del Orion Context Broker es suficiente
para una aplicación para calificar como _"Powered by FIWARE"_.

Actualmente, el Orion Context Broker depende de la tecnología de código abierto [MongoDB](https://www.mongodb.com/) para mantener
la persistencia de los datos de contexto que contiene. Por lo tanto, la arquitectura consistirá en dos elementos:

-   El [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) recibirá las solicitudes utilizando
    [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
-   La base de datos subyacente [MongoDB](https://www.mongodb.com/):
    -   Utilizada por el Orion Context Broker para guardar información de datos de contexto como entidades de datos, suscripciones e inscripciones

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
La inicialización por línea de comandos es autoexplicativa.

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
git checkout NGSI-v2

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

En el tutorial anterior, creamos cada entidad de **Store** individualmente.

Vamos a crear cinco unidades de estantes al mismo tiempo. Esta solicitud utiliza el endpoint del procesamiento por lotes para crear cinco entidades de estantes. El procesamiento por lotes utiliza el endpoint "v2/op/update" con un payload con dos atributos - `actionType=APPEND` significa que sobrescribiremos las entidades existentes si existiesen, mientras que el atributo `entities` contiene un conjunto de entidades que deseamos actualizar.

Para diferenciar las entidades **Shelf** de las entidades **Store**, a cada estante se le ha asignado "tipo=Shelf". Propiedades como "nombre" y "ubicación" han sido añadidas como propiedades a cada estante.

#### :one: Solicitud:

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

De manera similar, podemos crear una serie de entidades **Product** utilizando el `type=Product`.

#### :two: Solicitud:

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

En ambos casos hemos codificado cada entidad `id` de acuerdo con la especificació
[NGSI-LD](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.03.01_60/gs_cim009v010301p.pdf) - la propuesta es que cada `id` es una URN y sigue un formato estándar: `urn:ngsi-ld:<tipo-de-entidad>:<id-de-la-identidad>`.Esto significará que cada `id` en el sistema será único.

La información de Shelf puede ser solicitada haciendo una petición GET en el endpoint `v2/entities`. Por ejemplo, para devolver los datos de contexto de la entidad **Shelf** con el `id=urn:ngsi-ld:Shelf:unit001`.

#### :three: Solicitud:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=keyValues'
```

#### Respuesta:

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

Como puede ver, hay actualmente tres atributos de propiedad adicionales: `location`, `maxCapacity` y `name`.

## Creación de una Relación de uno-a-muchos

En bases de datos, las claves externas se utilizan a menudo para designar una relación de uno a muchos - por ejemplo, cada estante se encuentra en una sola tienda pero un almacén puede albergar muchos estantes. Para poder recordar esta información necesitamos añadir una
relación de asociación similar a una clave foránea. El procesamiento por lotes puede utilizarse de nuevo para modificar las
entidades **Shelf** para añadir un atributo `refStore` que mantiene la relación con cada tienda. De acuerdo con la guía de modelados de datos de FIWARE [datos vinculados](https://fiware-datamodels.readthedocs.io/en/latest/guidelines/index.html#modelling-linked-data), cuando se utilice un atributo de entidad como vínculo con otras entidades, deberá nombrarse con el prefijo `ref`más el nombre del tipo de entidad de destino (vinculado).

El valor del atributo `refStore` corresponde a una URN asociada a la propia entidad **Store**.

La URN sigue un formato estándar: `urn:ngsi-ld:<tipo-entidad>:<id-entidad>`

#### :four: Solicitud:

La siguiente solicitud asocia tres estantes a `urn:ngsi-ld:Store:001` y dos estantes a `urn:ngsi-ld:Store:002`.

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

Cuando la información del estante se solicita de nuevo, la respuesta ha cambiado e incluye una nueva propiedad `refStore`, añadida en el paso anterior.

#### :five: Solicitud:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=keyValues'
```

#### Respuesta:

La respuesta actualizada que incluye el atributo `refStore` se muestra a continuación:

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

También podemos hacer una solicitud para recuperar la información de la relación de atributos `refStore` de una entidad conocida `Shelf` usando el parámetro "options=values".

#### :six: Solicitud:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=values&attrs=refStore'
```

#### Respuesta:

```json
["urn:ngsi-ld:Store:001"]
```

Esto puede ser interpretado como "Estoy relacionado con la entidad **Store** con el `id=urn:ngsi-ld:Store:001`"

### Lectura desde la Entidad Padre a la Entidad Hijo

La lectura de un padre a un hijo puede hacerse usando el parámetro `options=count`.

#### :seven: Solicitud:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=count&attrs=type&type=Shelf'
```

Esta solicitud está pidiendo el `id` de todas las entidades de **Shelf** asociadas a la URN `urn:ngsi-ld:Store:001`, la respuesta es un arreglo JSON como se muestra.

#### Respuesta:

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

En lenguaje coloquial, esto puede interpretarse como "Hay tres estantes en `urn:ngsi-ld:Store:001`". La petición puede ser alterada usando los parámetros `options=values` y `attrs` para devolver propiedades específicas de las entidades asociadas relevantes. Por ejemplo, la solicitud:


#### :eight: Solicitud:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&type=Shelf&options=values&attrs=name'
```

Puede ser interpretada como una petición de _Dame los nombres de todos los estantes en `urn:ngsi-ld:Store:001`_.

#### Respuesta:

```json
[["Corner Unit"], ["Wall Unit 1"], ["Wall Unit 2"]]
```

## Creando Relaciones de muchos-a-muchos

Tablas puente se utilizan a menudo para relacionar las relaciones de muchos con muchos. Por ejemplo, cada tienda venderá una gama diferente de productos, y cada producto se vende en muchas tiendas diferentes.

A fin de mantener la información de contexto para "colocar un producto en un estante de una tienda determinada", será necesario crear una nueva entidad de datos **InventoryItem** que exista para asociar los datos de otras entidades. Tiene una relación clave ajena a las entidades **Store**, **Shelf** y **Product** y por lo tanto requiere atributos de relación llamados `refStore`, `refShelf` y `refProduct`.

La asignación de un producto a un estante se hace simplemente creando una entidad que contenga la información de la relación y cualquier otra propiedad adicional (como `StockCount` y `ShelfCount`)

#### :nine: Solicitud:

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

Cuando se lee de una entidad de la tabla puente, el `type` de la entidad debe ser conocido.

Después de crear al menos una entidad **InventoryItem** podemos consultar _¿Qué productos se venden en `urn:ngsi-ld:Store:001`?_ haciendo la siguiente petición

#### :one::zero: Solicitud:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=values&attrs=refProduct&type=InventoryItem'
```

#### Respuesta:

```json
[["urn:ngsi-ld:Product:prod001"]]
```

Del mismo modo, podemos consultar _¿Qué tiendas están vendiendo `urn:ngsi-ld:Product:001`?_ alterando la petición como se muestra:

#### :one::one: Solicitud:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refProduct==urn:ngsi-ld:Product:001&options=values&attrs=refStore&type=InventoryItem'
```

#### Respuesta:

```json
[["urn:ngsi-ld:Store:001"]]
```

## Integridad de los datos

Las relaciones de datos de contexto sólo deben establecerse y mantenerse entre entidades que existan - en otras palabras, la URN `urn:ngsi-ld:<tipo-entidad>:<id-entidad>` debe vincularse a otra entidad existente dentro del contexto. Por lo tanto, debemos tener cuidado al borrar una entidad de que no queden referencias colgantes. Imagina que se borra `urn:ngsi-ld:Store:001` - ¿qué debería pasar con las entidades **Shelf** asociadas?

Es posible hacer una solicitud para ver si existe alguna relación de entidad restante antes de la supresión, haciendo una solicitud como la siguiente

#### :one::two: Solicitud:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=count&attrs=type'
```

#### :one::three: Solicitud:

La respuesta enumera una serie de entidades de **Shelf** y **InventoryItem** - no hay entidades de **Product** ya que no hay una relación directa entre el producto y la tienda.

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

Si esta solicitud devuelve un arreglo vacío, la entidad no tiene objetos asociados.

# Próximos pasos

¿Quieres aprender a añadir más complejidad a tu aplicación añadiendo funciones avanzadas? Puedes averiguarlo leyendo
los otros [tutoriales de esta serie](https://fiware-tutorials.rtfd.io)

---

## Licencia

[MIT](LICENSE) © 2018-2020 FIWARE Foundation e.V.
