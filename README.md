[![FIWARE Banner](https://fiware.github.io/tutorials.CRUD-Operations/img/fiware.png)](https://www.fiware.org/developers)
[![NGSI LD](https://img.shields.io/badge/NGSI-LD-d6604d.svg)](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.03.01_60/gs_cim009v010301p.pdf)

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://github.com/FIWARE/catalogue/blob/master/core/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.CRUD-Operations.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
[![JSON LD](https://img.shields.io/badge/JSON--LD-1.1-f06f38.svg)](https://w3c.github.io/json-ld-syntax/) <br/>
[![Documentation](https://img.shields.io/readthedocs/ngsi-ld-tutorials.svg)](https://ngsi-ld-tutorials.rtfd.io)

This tutorial teaches **NGSI-LD** users about batch commands and entity relationships. The tutorial builds on the data
created in the previous [Smart Farm example](https://github.com/FIWARE/tutorials.Getting-Started/tree/NGSI-LD) and
creates and associates a series of related data entities to create add sensors and farm workers to the farm.

The tutorial uses [cUrl](https://ec.haxx.se/) commands throughout, but is also available as
[Postman documentation](https://fiware.github.io/tutorials.Entity-Relationships/ngsi-ld.html).

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/d0f2b74c4beb8434595f)

-   このチュートリアルは[日本語](README.ja.md)でもご覧いただけます。

## Contents

<details>
<summary><strong>Details</strong></summary>

-   [Understanding Entities and Relationships](#understanding-entities-and-relationships)
    -   [Entities within a Farm Management Information System (FMIS)](#entities-within-a-farm-management-information-system-fmis)
-   [Architecture](#architecture)
-   [Prerequisites](#prerequisites)
    -   [Docker and Docker Compose](#docker-and-docker-compose)
    -   [Cygwin for Windows](#cygwin-for-windows)
-   [Start Up](#start-up)
-   [Creating and Associating Data Entities](#creating-and-associating-data-entities)
    -   [Creating Several Entities at Once](#creating-several-entities-at-once)
    -   [Creating one-to-one or one-to-many Relationships](#creating-one-to-one-or-one-to-many-relationships)
    -   [Reading a Foreign Key Relationship](#reading-a-foreign-key-relationship)
        -   [Reading from Child Entity to Parent Entity](#reading-from-child-entity-to-parent-entity)
        -   [Reading from Parent Entity to Child Entity](#reading-from-parent-entity-to-child-entity)
    -   [Creating many-to-many Relationships](#creating-many-to-many-relationships)
    -   [Reading from a bridge table](#reading-from-a-bridge-table)
    -   [Relationships of Properties](#relationships-of-properties)
        -   [Retrieving the Temperature of a Barn](#retriving-the-temperature-of-a-barn)
-   [Next Steps](#next-steps)

</details>

# Understanding Entities and Relationships

Within the FIWARE platform, the context of an entity represents the state of a physical or conceptual object which
exists in the real world.

## Entities within a Farm Management Information System (FMIS)

To illustrate entity relationships within an FMIS system based on NGSI-LD, we will need to create a series of entities.
For this simplified FMIS, we will only need a small number entities. The relationship between our entities is defined as
shown:

![](https://fiware.github.io/tutorials.Entity-Relationships/img/ngsi-ld-entities.png)

-   A building, such as a barn, is a real world bricks and mortar construct. **Building** entities would have properties
    such as:
    -   A name of the building e.g. "The Big Red Barn"
    -   The category of the building (e.g. "barn")
    -   An address "Friedrichstraße 44, 10969 Kreuzberg, Berlin"
    -   A physical location e.g. _52.5075 N, 13.3903 E_
    -   A filling level - the degree to which the building is full.
    -   A temperature - e.g. _21 °C_
    -   An association to the owner of the building (a real person)
-   Smart devices such as **TemperatureSensors** or **FillingLevelSensors** would extend a common **Device** data model.
    Each **Device** entity would have properties such as:
    -   A description of the device
    -   The category of device (e.g. _sensor_, _actuator_, _both_)
    -   The name of the property they are measuring (e.g. _temperature_)
    -   An association to the asset (e.g. building) they are measuring
-   A **person** is an entity representing a farmer or farm labourer. Each **Person** entity would have properties such
    as:
    -   A name of the person e.g. "Mr. Jones"
    -   A job title
    -   An association to the farm buildings they own.
-   A task something we do down on the farm. It is a conceptual entity, used to associate workers, agricultural products
    and locations **Task** entities would have properties such as:
    -   The name of the task (e.g. _Spray Herbicide XXX on field Y_)
    -   The status of the task (e.g. _scheduled_, _in progress_, _completed_)
    -   An association to the worker (i.e. a **Person** entity) who performs the task
    -   An association to the product (e.g. **Herbicide** entity) to be used.
    -   An association to the location (e.g. **PartField** entity) to be used.

As you can see, each of the entities defined above contain a mixture of static and dynamic data. Some properties are
liable to change. A **Herbicide** could change its `formula`, hay could be sold and the `fillingLevel` of the barn could
be reduced and so on.

> **Note** this tutorial uses the following typographic styling :
>
> -   Entity types have been made **bold text**
> -   Data attributes are written in `monospace text`
> -   Items in the real world use plain text
>
> Therefore a person in the real world is represented in the context data by a **Person** entity, and a real world barn
> owned by a person is represented in the context data by a **Building** entity which has a `owner` attribute.

# Architecture

The demo FMIS application will send and receive NGSI-LD calls to a compliant context broker. Since the standardized
NGSI-LD interface is available across multiple context brokers, so we only need to pick one - for example the
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/). The application will therefore only make use of
one FIWARE component.

Currently, the Orion Context Broker relies on open source [MongoDB](https://www.mongodb.com/) technology to keep
persistence of the context data it holds.

To promote interoperability of data exchange, NGSI-LD context brokers explicitly expose a
[JSON-LD `@context` file](https://json-ld.org/spec/latest/json-ld/#the-context) to define the data held within the
context entities. This defines a unique URI for every entity type and every attribute so that other services outside of
the NGSI domain are able to pick and choose the names of their data structures. Every `@context` file must be available
on the network. In our case the tutorial application will be used to host a series of static files.

Therefore, the architecture will consist of three elements:

-   The [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which will receive requests using
    [NGSI-LD](https://forge.etsi.org/swagger/ui/?url=https://forge.etsi.org/gitlab/NGSI-LD/NGSI-LD/raw/master/spec/updated/full_api.json)
-   The underlying [MongoDB](https://www.mongodb.com/) database :
    -   Used by the Orion Context Broker to hold context data information such as data entities, subscriptions and
        registrations
-   The **Tutorial Application** does the following:
    -   Offers static `@context` files defining the context entities within the system.

Since all interactions between the two elements are initiated by HTTP requests, the entities can be containerized and
run from exposed ports.

![](https://fiware.github.io/tutorials.Entity-Relationships/img/architecture-ld.png)

The necessary configuration information can be seen in the services section of the associated `docker-compose.yml` file.
It has been described in a [previous tutorial](https://github.com/FIWARE/tutorials.Getting-Started/tree/NGSI-LD)

# Prerequisites

## Docker and Docker Compose

To keep things simple both components will be run using [Docker](https://www.docker.com). **Docker** is a container
technology which allows to different components isolated into their respective environments.

-   To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
-   To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
-   To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A
[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Entity-Relationships/master/docker-compose.yml) is used
configure the required services for the application. This means all container services can be brought up in a single
command. Docker Compose is installed by default as part of Docker for Windows and Docker for Mac, however Linux users
will need to follow the instructions found [here](https://docs.docker.com/compose/install/)

You can check your current **Docker** and **Docker Compose** versions using the following commands:

```console
docker-compose -v
docker version
```

Please ensure that you are using Docker version 18.03 or higher and Docker Compose 1.21 or higher and upgrade if
necessary.

## Cygwin for Windows

We will start up our services using a simple Bash script. Windows users should download [cygwin](http://www.cygwin.com/)
to provide a command-line functionality similar to a Linux distribution on Windows.

# Start Up

All services can be initialised from the command-line by running the
[services](https://github.com/FIWARE/tutorials.Entity-Relationships/blob/NGSI-LD/services) Bash script provided within
the repository. Please clone the repository and create the necessary images by running the commands as shown:

```console
git clone https://github.com/FIWARE/tutorials.Entity-Relationships.git
cd tutorials.Entity-Relationships
git checkout NGSI-LD

./services start
```

This command will also import seed data (**Building**, **Person**, **TemperatureSensor**, **FillingLevelSensor**,
**Herbicide** and **PartField**) on startup.

> :information_source: **Note:** If you want to clean up and start over again you can do so with the following command:
>
> ```console
> ./services stop
> ```

# Creating and Associating Data Entities

## Creating Several Entities at Once

In the previous tutorial, we created each entity individually,

Lets create several sensors at the same time. This request uses the convenience batch processing endpoint to create five
entities. Batch processing uses the `/ngsi-ld/v1/entityOperations/`endpoints and the `upsert` endpoints means we will
create new entities if they are not present and overwrite existing entities if they exist.

To differentiate different **Device**, each temperature sensor has been assigned `type=TemperatureSensor`. Real-world
properties such as `category` have been added as properties to each device.

#### :one: Request:

```console
curl -X POST 'http://locahost:1026/ngsi-ld/v1/entityOperations/upsert' \
-H 'Content-Type: application/json' \
-H 'Link: <'http://context-provider:3000/data-models/ngsi-context.jsonld'>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/ld+json' \
--data-raw '[
    {
      "id": "urn:ngsi-ld:TemperatureSensor:001",
      "type": "TemperatureSensor",
      "description" : {"type": "Property", "value": "Temperature Gauge 1"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "temperature"},
      "temperature": {"type": "Property", "value": 20, "unitCode": "CEL"}
    },
    {
      "id": "urn:ngsi-ld:TemperatureSensor:002",
      "type": "TemperatureSensor",
      "description" : {"type": "Property", "value": "Temperature Gauge 2"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "temperature"},
      "temperature": {"type": "Property", "value": 21, "unitCode": "CEL"}
    },
    {
      "id": "urn:ngsi-ld:TemperatureSensor:003",
      "type": "TemperatureSensor",
      "description" : {"type": "Property", "value": "Temperature Gauge 3"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "temperature"},
      "temperature": {"type": "Property", "value": 27, "unitCode": "CEL"}
    }
]'
```

Similarly, we can create a series of **FillingLevelSensors** entities by using the `type=FillingLevelSensor`.

#### :two: Request:

```console
curl -X POST 'http://locahost:1026/ngsi-ld/v1/entityOperations/upsert' \
-H 'Content-Type: application/json' \
-H 'Link: <'http://context-provider:3000/data-models/ngsi-context.jsonld'>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/ld+json' \
--data-raw '[
    {
      "id": "urn:ngsi-ld:FillingLevelSensor:001",
      "type": "FillingLevelSensor",
      "description" : {"type": "Property", "value": "Filling Level Sensor 1"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "fillingLevel"},
      "fillingLevel": {"type": "Property", "value": 1, "unitCode": "C62"}
    },
    {
      "id": "urn:ngsi-ld:FillingLevelSensor:002",
      "type": "FillingLevelSensor",
      "description" : {"type": "Property", "value": "Filling Level Sensor 2"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "fillingLevel"},
      "fillingLevel": {"type": "Property", "value": 0.9, "unitCode": "C62"}
    },
    {
      "id": "urn:ngsi-ld:FillingLevelSensor:003",
      "type": "FillingLevelSensor",
      "description" : {"type": "Property", "value": "Filling Gauge 3"},
      "category": {"type": "Property", "value": "sensor"},
      "controlledProperty" : {"type": "Property", "value": "fillingLevel"},
      "fillingLevel": {"type": "Property", "value": 0.8, "unitCode": "C62"}
    }
]'
```

In both cases we have encoded each entity `id` according to the NGSI-LD
[specification](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.03.01_60/gs_cim009v010301p.pdf) - the proposal
is that each `id` is a URN follows a standard format: `urn:ngsi-ld:<entity-type>:<entity-id>`. This will mean that every
`id` in the system will be unique.

Device information can be requested by making a GET request on the `/ngsi-ld/v1/entities` endpoint. For example to
return the context data of the devices

#### :three: Request:

```console
curl -X GET 'http://localhost:1026/ngsi-ld/v1/entities/?type=TemperatureSensor,FillingSensor&options=keyValues' \
-H 'Link: <http://context-provider:3000/data-models/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
```

#### Response:

```jsonld
[
    {
        "@context": "http://context-provider:3000/data-models/ngsi-context.jsonld",
        "id": "urn:ngsi-ld:TemperatureSensor:001",
        "type": "TemperatureSensor",
        "description": "Temperature Gauge 1",
        "category": "sensor",
        "controlledProperty": "temperature",
        "temperature": 20
    },
    {
        "@context": "http://context-provider:3000/data-models/ngsi-context.jsonld",
        "id": "urn:ngsi-ld:TemperatureSensor:002",
        "type": "TemperatureSensor",
        "description": "Temperature Gauge 2",
        "category": "sensor",
        "controlledProperty": "temperature",
        "temperature": 21
    },
    ... etc
]
```

As you can see there are currently three additional property attributes present `description`, `category` and
`controlledProperty`

## Creating one-to-one or one-to-many Relationships

In databases, foreign keys are often used to designate one-to-one or  one-to-many relationships - for example a building has a single owner but can hold many
devices. In order to remember this information we need to add an association relationship similar to a foreign key.
Batch processing can again be used to amend the existing the **TemperatureSensor** and **FillingLevelSensor** entities
to add a `controllingAsset` attribute holding the one-to-one relationship to each building controlled by the device. According to
the Smart Data Model [Device](https://swagger.lab.fiware.org/?url=https://smart-data-models.github.io/dataModel.Device/Device/swagger.yaml)
definition `https://uri.fiware.org/ns/data-models#controllingAsset` is the URI long name to be used for this
relationship, and the value of the `controllingAsset` attribute corresponds to a URN associated to a **Building** entity
itself.

The URN follows a standard format: `urn:ngsi-ld:<entity-type>:<entity-id>`

#### :four: Request:

The following request associates six devices to `urn:ngsi-ld:Building:farm001`, `urn:ngsi-ld:Building:barn002` and
`urn:ngsi-ld:Building:farm002`

```console
curl -G -iX POST 'http://localhost:1026/ngsi-ld/v1/entityOperations/upsert' \
-d 'options=update' \
-H 'Content-Type: application/json' \
-H 'Link: <http://context-provider:3000/data-models/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
--data-raw '[
    {
        "id": "urn:ngsi-ld:TemperatureSensor:001",
        "type": "TemperatureSensor",
        "controllingAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:farm001"}
    },
    {
        "id": "urn:ngsi-ld:TemperatureSensor:002",
        "type": "TemperatureSensor",
        "controllingAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:barn002"}
    },
    {
        "id": "urn:ngsi-ld:FillingLevelSensor:003",
        "type": "FillingLevelSensor",
        "controllingAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:farm002"}
    },
    {
        "id": "urn:ngsi-ld:FillingLevelSensor:001",
        "type": "FillingLevelSensor",
        "controllingAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:farm001"}
    },
    {
        "id": "urn:ngsi-ld:FillingLevelSensor:002",
        "type": "FillingLevelSensor",
        "controllingAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:barn002"}
    },
    {
        "id": "urn:ngsi-ld:TemperatureSensor:003",
        "type": "TemperatureSensor",
        "controllingAsset": {"type": "Relationship", "object": "urn:ngsi-ld:Building:farm002"}
    }
]'
```

Now when the devcie information is requested again, the response has changed and includes a new property
`controllingAsset`, which has been added in the previous step.

#### :five: Request:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:TemperatureSensor001' \
-d 'options=keyValues' \
-H 'Link: <http://context-provider:3000/data-models/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
```

#### Response:

The updated response including the `controllingAsset` attribute is shown below:

```jsonld
{
    "@context": "http://context-provider:3000/data-models/ngsi-context.jsonld",
    "id": "urn:ngsi-ld:TemperatureSensor:001",
    "type": "TemperatureSensor",
    "description": "Temperature Gauge 1",
    "category": "sensor",
    "controlledProperty": "temperature",
    "temperature": 20,
    "controllingAsset": "urn:ngsi-ld:Building:farm001"
}
```

## Reading a Foreign Key Relationship

### Reading from Child Entity to Parent Entity

We can also make a request to retrieve the `controllingAsset` attribute relationship information from a known **Device**
entity by using the `options=keyValues` setting

#### :six: Request:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:TemperatureSensor:001' \
-d 'options=keyValues' \
-d 'attrs=controllingAsset' \
-H 'Link: <http://context-provider:3000/data-models/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/json'
```

#### Response:

```json
{
    "id": "urn:ngsi-ld:TemperatureSensor:001",
    "type": "TemperatureSensor",
    "controllingAsset": "urn:ngsi-ld:Building:farm001"
}
```

This can be interpreted as _"I am making sensor readings inside the **Building** entity with the
`id=urn:ngsi-ld:Building:farm001`"_

### Reading from Parent Entity to Child Entity

Reading from a parent to a child can be done using the following query:

#### :seven: Request:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities' \
-d 'q=controllingAsset==%22urn:ngsi-ld:Building:farm001%22' \
-d 'attrs=controllingAsset' \
-d 'options=keyValues' \
-H 'Link: <http://context-provider:3000/data-models/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
```

This request is asking for the `id` of all **Device** entities associated to the URN `urn:ngsi-ld:Building:farm001`, the
response is a JSON array as shown.

#### Response:

```jsonld
[
    {
        "@context": "http://context-provider:3000/data-models/ngsi-context.jsonld",
        "id": "urn:ngsi-ld:TemperatureSensor:001",
        "type": "TemperatureSensor",
    },
    {
        "@context": "http://context-provider:3000/data-models/ngsi-context.jsonld",
        "id": "urn:ngsi-ld:FillingLevelSensor:001",
        "type": "FillingLevelSensor",
    }
]
```

In plain English, this can be interpreted as _"There are two devices in `urn:ngsi-ld:Building:farm001`"_. The request can
be altered use the `count=true` to return the number of entities which fulfill the criteria.

#### :eight: Request:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities' \
-d 'q=controllingAsset==%22urn:ngsi-ld:Building:farm001%22' \
-d 'attrs=controllingAsset' \
-d 'options=keyValues' \
-d 'count=true' \
-d 'limit=0' \
-H 'Link: <http://context-provider:3000/data-models/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"'
```

Returns an HTTP Header as part of the response which indicates the number of affected entities:

#### Response:

```text
NGSILD-Results-Count: 2
```

### Creating many-to-many Relationships

Bridge Tables are often used to relate many-to-many relationships. For example, every spraying activity within the FMIS
will need to associate a farm worker, a product to apply, and a location to apply the treatment (known as a
**PartField**)

In order to hold the context information to "direct a worker to spray a herbicide onto a field" we will need to create a
new data entity **Task** which exists to associate data from other entities. It has a foreign key relationship to the
**Person**, **Herbicide** and **PartField** entities and therefore requires relationship attributes called `field`,
`herbicide` and `worker`.

Assigning a task is simply done by creating an entity holding the relationship information and any other additional
properties (such as `description` and `status`)

#### :nine: Request:

```console
curl -L -X POST 'http://localhost:1026/ngsi-ld/v1/entities/' \
-H 'Content-Type: application/json' \
-H 'Link: <http://context-provider:3000/data-models/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
--data-raw '{
    "id": "urn:ngsi-ld:Task:001",
    "type": "Task",
    "worker": {"type": "Relationship", "object": "urn:ngsi-ld:Person:001"},
    "field": {"type": "Relationship", "object": "urn:ngsi-ld:PartField:002"},
    "product": {"type": "Relationship", "object": "urn:ngsi-ld:Herbicide:001"},
    "description": {"type": "Property", "value": "Spray the North Field with Agent Orange"},
    "status": {"type": "Property", "value": "scheduled"},
    "dueDate": {"type": "Property", "value": "2021-07-16"}
}'
```

## Reading from a bridge table

When reading from a bridge table entity, the `type` of the entity must be known.

After creating at least one **Task** entity we can query _Which workers are assigned activities in field
`urn:ngsi-ld:PartField:002`?_ by making the following request

#### :one::zero: Request:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities' \
-d 'q=field==%22urn:ngsi-ld:PartField:002%22' \
-d 'options=keyValues' \
-d 'attrs=worker' \
-d 'type=Task' \
-H 'Link: <http://context-provider:3000/data-models/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/json'
```

#### Response:

```jsonld
[
    {
        "id": "urn:ngsi-ld:Task:001",
        "type": "Task",
        "worker": "urn:ngsi-ld:Person:001"
    }
]
```

Similarly we can request _Which fields are treated using `urn:ngsi-ld:Herbicide:001`?_ by altering the request as shown:

#### :one::one: Request:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities' \
-d 'q=product==%22urn:ngsi-ld:Herbicide:001%22' \
-d 'options=keyValues' \
-d 'attrs=field' \
-d 'type=Task' \
-H 'Link: <http://context-provider:3000/data-models/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/json'
```

#### Response:

```json
[
    {
        "id": "urn:ngsi-ld:Task:001",
        "type": "Task",
        "field": "urn:ngsi-ld:PartField:002"
    }
]
```

## Relationships of Properties

_Properties-of-Properties_ and _Relationships of Properties_ are metadata. The addition of metadata entities such as
these into the context data allows users to navigate the graph of entities relationships and gain further insights about
the state of the system.

### Retrieving the Temperature of a Barn

The temperature readings from a temperature sensor have already been discussed. It may also be necessary to duplicate
this data into another entity. For example, the temperature reading of a sensor in the barn is also the temperature
reading of the barn itself. A dummy reading has already been added into the `urn:ngsi-ld:Building:farm001` Entity and
can be retrieved with a GET request:

#### :one::two: Request:

```console
curl -G -iX GET 'http://localhost:1026/ngsi-ld/v1/entities/urn:ngsi-ld:Building:farm001' \
-d 'attrs=temperature' \
-H 'Link: <http://context-provider:3000/data-models/ngsi-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"; type="application/ld+json"' \
-H 'Accept: application/json'
```

#### Response:

```json
{
    "id": "urn:ngsi-ld:Building:farm001",
    "type": "Building",
    "temperature": {
        "type": "Property",
        "value": 20,
        "unitCode": "CEL",
        "providedBy": {
            "type": "Relationship",
            "object": "urn:ngsi-ld:TemperatureSensor:001"
        }
    }
}
```

As you can see the `temperature` Property holds additional information regarding the provider of the measurement, this
will allow additional inferences to be made.

# Next Steps

Want to learn how to add more complexity to your application by adding advanced features? You can find out by reading
the other [tutorials in this series](https://ngsi-ld-tutorials.rtfd.io)

---

## License

[MIT](LICENSE) © 2020 FIWARE Foundation e.V.
