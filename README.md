This tutorial teaches FIWARE users about batch commands and entity relationships.

This tutorial builds on the data created in the previous [store finder example](https://github.com/Fiware/tutorials.Getting-Started) and creates and associates a series of related data entities to create a simple stock management system.

The tutorial is also available as [Postman documentation](http://fiware.github.io/tutorials.Entity-Relationships/).

[![Run in Postman](https://run.pstmn.io/button.svg)](https://www.getpostman.com/collections/0671934f64958d3200b3)
 

# Contents

- [Understanding Entities and Relationships](#understanding-entities-and-relationships)
  * [Entities within a stock management system](#entities-within-a-stock-management-system)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
  * [Docker and Docker Compose](#docker-and-docker-compose)
  * [Cygwin for Windows](#cygwin-for-windows)
- [Start Up](#start-up)
- [Creating and Associating Data Entities](#creating-and-associating-data-entities)
  * [Creating Several Entities at Once](#creating-several-entities-at-once)
  * [Creating a one-to-many Relationship](#creating-a-one-to-many-relationship)
  * [Reading a Foreign Key Relationship](#reading-a-foreign-key-relationship)
    + [Reading from Child Entity to Parent Entity](#reading-from-child-entity-to-parent-entity)
    + [Reading from Parent Entity to Child Entity](#reading-from-parent-entity-to-child-entity)
  * [Creating many-to-many Relationships](#creating-many-to-many-relationships)
  * [Reading from a bridge table](#reading-from-a-bridge-table)
  * [Data Integrity](#data-integrity)

# Understanding Entities and Relationships

Within the FIWARE platform, an entity represents the state of a physical or conceptural object which exists in the real world.

## Entities within a stock management system

For a simple stock management system, we will only need four types of entity. The relationship between our entities is defined as shown:

![](https://fiware.github.io/tutorials.Entity-Relationships/img/entities.png)

* A store is a real world bricks and mortar building. **Store** entities would have properties such as:
  + A name of the store e.g. "Checkpoint Markt"
  + An address "Friedrichstraße 44, 10969 Kreuzberg, Berlin"
  + A phyiscal location  e.g. *52.5075 N, 13.3903 E*
* A shelf is a real world device to hold objects which we wish to sell. Each **Shelf** entity would have properties such as:
  + A name of the shelf e.g. "Wall Unit"
  + A phyiscal location  e.g. *52.5075 N, 13.3903 E*
  + A maximum capacity
  + An association to the store in which the shelf is present
* A product is defined as something that we sell - it is conceptural object. **Product** entities would have properties such as:
  + A name of the product e.g. "Vodka"
  + A price e.g. 13.99 Euros
  + A size e.g. Small
* An inventory item is another conceptural entity, used to assocate products, stores, shelves and physical objects. **Inventory Item** entities would have properties such as:
  + An assocation to the product being sold
  + An association to the store in which the product is being sold
  + An association to the shelf where the product is being displayed
  + A stock count of the quantity of the product available in the warehouse
  + A stock count of the quantity of the product available on the shelf


As you can see, each of the entities defined above contain some properties which are liable to change. A product could change its price, stock could be sold and the shelf count of stock could be reduced and so on.


> **Note** this tutorial uses the following typographic styling :
> 
> * Entity types have been made **bold text** 
> * Data attributes are written in `monospace text`  
> * Items in the real world use plain text
>
> Therefore a store in the real world  is represented in the context data by a **Store** 
> entity, and a real world shelf found in a store is represented in the context data by 
> a **Shelf** entity which has a `refStore` attribute.
>


# Architecture

This application will only make use of one FIWARE component - the [Orion Context Broker](https://catalogue.fiware.org/enablers/publishsubscribe-context-broker-orion-context-broker). Usage of the Orion Context Broker is sufficient for an application to qualify as *“Powered by FIWARE”*.

Currently, the Orion Context Broker relies on open source [MongoDB](https://www.mongodb.com/) technology to keep persistence of the context data it holds. Therefore, the architecture will consist of two elements:

* The Orion Context Broker server which will receive requests using NGSI
* The underlying MongoDB database associated to the Orion Context Broker server

Since all interactions between the two elements are initiated by HTTP requests, the entities can be containerized and run from exposed ports. 

![](https://fiware.github.io/tutorials.Entity-Relationships/img/architecture.png)

# Prerequisites

## Docker and Docker Compose 

To keep things simple both components will be run using [Docker](https://www.docker.com). **Docker** is a container technology which allows to different components isolated into their respective environments. 

* To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
* To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
* To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A [YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Entity-Relationships/master/docker-compose.yml) is used configure the required
services for the application. This means all container sevices can be brought up in a single commmand. Docker Compose is installed by default as part of Docker for Windows and  Docker for Mac, however Linux users will need to follow the instructions found [here](https://docs.docker.com/compose/install/)

## Cygwin for Windows

We will start up our services using a simple Bash script. Windows users should download [cygwin](www.cygwin.com) to provide a command line functionality similar to a Linux distribution on Windows.


# Start Up

All services can be initialised from the command line by running the [services](https://github.com/Fiware/tutorials.Entity-Relationships/blob/master/services) Bash script provided within the repository:

```bash
./services start
``` 

This command will also import seed data from the previous [Store Finder tutorial](https://github.com/Fiware/tutorials.Getting-Started) on startup.

>**Note:** If you want to clean up and start over again you can do so with the following command:
>
>```bash
>./services stop
>``` 
>


#  Creating and Associating Data Entities

## Creating Several Entities at Once

In the previous tutorial, we created each **Store** entity individually, 

Lets create five shelf units at the same time. This request uses the convenience batch processing endpoint to create five shelf entities. Batch processing uses the `/v2/op/update` endpoint with a payload with two attributes - `actionType=APPEND` means we will overwrite existing entities if they exist whereas the  `entities` attribute holds an array of entities we wish to update.

To differenciate **Shelf** Entities from **Store** Entities, each shelf has been assigned `type=Shelf`. 
Real-world properties such as `name` and `location` have been addded as properties to each shelf.

#### Request:

```bash
curl -X POST \
  'http://localhost:1026/v2/op/update' \
  -H 'Content-Type: application/json' \
  -d '{
  "actionType":"APPEND",
  "entities":[
{
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

#### Request:

```bash
curl -X POST \
  'http://localhost:1026/v2/op/update' \
  -H 'Content-Type: application/json' \
  -d '{
  "actionType":"APPEND",
  "entities":[
{
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

In both cases we have encoded each entity `id` according to the NGSI-LD [draft recommendation](https://docbox.etsi.org/ISG/CIM/Open/ISG_CIM_NGSI-LD_API_Draft_for_public_review.pdf) -  the proposal is that each `id` is a URN follows a standard format: `urn:ngsi-ld:<entity-type>:<entity-id>`. This will mean that every `id` in the system will be unique.

Shelf information can be requested by making a GET request on the `/v2/entities` endpoint. For example to return the context data of the **Shelf** entity with the `id=urn:ngsi-ld:Shelf:unit001`.

#### Request:

```bash
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
        "coordinates": [
            13.3986112,
            52.554699
        ]
    },
    "maxCapacity": 50,
    "name": "Corner Unit"
}
```

As you can see there are currently three additional property attributes present `location`, `maxCapacity` and `name`

## Creating a one-to-many Relationship

In databases, foreign keys are often used to designate a one-to-many relationship - for example every shelf is found in a single store and a single store can hold many shelving units. In order to remember this information we need to add an association relationship similar to a foreign key. Batch processing can again be used to amend the existing the **Shelf** entities to add a `refStore` attribute holding the relationship to each shelf.  According to the FIWARE Data Modelling Guidelines on [linked data](http://fiware-datamodels.readthedocs.io/en/latest/guidelines/index.html#modelling-linked-data), when an entity attribute is used as a link to other entities  it should be named with the prefix `ref` plus the name of the target (linked) entity type. 

The value of the `refStore` attribute corresponds to a URN associated to a **Store** entity itself. 

The URN follows a standard format: `urn:ngsi-ld:<entity-type>:<entity-id>`

#### Request:

The following request associates three shelves to `urn:ngsi-ld:Store:001` and two shelves to `urn:ngsi-ld:Store:002`

```bash
curl -X POST \
  'http://localhost:1026/v2/op/update' \
  -H 'Content-Type: application/json' \
  -d '{
  "actionType":"APPEND",
  "entities":[
    {
      "id":"unit001", "type":"Shelf",
      "refStore": { 
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
      }
    },
    {
      "id":"unit002", "type":"Shelf",
      "refStore": { 
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
      }
    },
    {
      "id":"unit003", "type":"Shelf",
      "refStore": { 
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:001"
      }
    },
    {
      "id":"unit004", "type":"Shelf",
      "refStore": { 
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:002"
      }
    },
    {
      "id":"unit005", "type":"Shelf",
      "refStore": { 
        "type": "Relationship",
        "value": "urn:ngsi-ld:Store:002"
      }
    }
  ]
}'
```

Now when the shelf information is requested again, the response has changed and includes a new property `refStore`, which has been added in the previous step.

#### Request:

```bash
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
        "coordinates": [
            13.3986112,
            52.554699
        ]
    },
    "maxCapacity": 50,
    "name": "Corner Unit",
    "refStore": "urn:ngsi-ld:Store:001"
}
```


## Reading a Foreign Key Relationship

### Reading from Child Entity to Parent Entity

We can also make a request to retrieve the `refStore` attribute relationship information from a known **Shelf** entity by using the `options=values` setting

#### Request:

```bash
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Shelf:unit001/?type=Shelf&options=values&attrs=refStore
```

#### Response:

```json
[
    "urn:ngsi-ld:Store:001"
]
```

This can be interpreted as "I am related to the **Store** entity with the `id=urn:ngsi-ld:Store:001`"

### Reading from Parent Entity to Child Entity

Reading from a parent to a child can be done using the  `options=count` setting

#### Request:

```bash
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=count&attrs=type&type=Shelf' 
```

This request is asking for the `id` of all **Shelf** entities associated to the URN `urn:ngsi-ld:Store:001`, the response is a JSON array as shown.

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

In plain English, this can be interpreted as "There are three shelves in `urn:ngsi-ld:Store:001`".  The request can be altered use the `options=values` and `attrs` parameters to return specific properties of the relevant associated entities. For example the request:

#### Request:

```bash
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&type=Shelf&options=values&attrs=name'
```

Can be interpreted as request for *Give me the names of all shelves in `urn:ngsi-ld:Store:001`*.

#### Response:

```json
[
    [
        "Corner Unit"
    ],
    [
        "Wall Unit 1"
    ],
    [
        "Wall Unit 2"
    ]
]
```


## Creating many-to-many Relationships

Bridge Tables are often used to relate many-to-many relationships. For example, every store will sell a different range of products, and each product is sold in many different stores. 

In order to hold the context information to "place a product onto a shelf in a given store" we will need to create a new data entity **InventoryItem** which exists to associate data from other entities. It has a foreign key relationship to 
the **Store**, **Shelf** and **Product** entities and therefore requires relationship attributes called `refStore`, `refShelf` and `refProduct`.

Assigning a product to a shelf is simply done by creating an entity holding the relationship information and any other additional properties (such as `stockCount` and `shelfCount`)

#### Request:

```bash
curl -X POST \
  'http://localhost:1026/v2/entities/' \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/json' \
  -H 'Postman-Token: 0588ef62-6b5c-4d1b-8066-172d63b516fd' \
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



## Reading from a bridge table

When reading from a bridge table entity, the `type` of the entity must be known.

After creating at least one **InventoryItem** entity we can query *Which products are sold in `urn:ngsi-ld:Store:001`?* by making the following request

#### Request:

```bash
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refProduct==urn:ngsi-ld:Product:001&options=values&attrs=refStore&type=InventoryItem' 
```

#### Response:

```json
[
    [
        "urn:ngsi-ld:Store:001"
    ]
]
```


Similarly we can request *Which stores are selling `urn:ngsi-ld:Product:001`?* by altering the request as shown: 

#### Request:

```bash
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=values&attrs=refProduct&type=InventoryItem' 
```

#### Response:

```json
[
    [
        "urn:ngsi-ld:Product:prod001"
    ]
]
```



## Data Integrity

Context data relationships should only be set up and maintained between entities that exist - in other words the URN `urn:ngsi-ld:<entity-type>:<entity-id>` should link to another existing entity within the context. Therefore we must take care when deleting an entity that no dangling references remain. Imagine `urn:ngsi-ld:Store:001` is deleted - what should happen to the associated the **Shelf** entities?

It is possible to make a request to see if any remaining entity relationship exists prior to deletion by making a request as follows

#### Request:

```bash
curl -X GET \
  'http://localhost:1026/v2/entities/?q=refStore==urn:ngsi-ld:Store:001&options=count&attrs=type'
```


#### Request:

The response lists a series of **Shelf** and **InventoryItem** entities - there are no **Product** entities since there is no direct relationship between product and store.

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