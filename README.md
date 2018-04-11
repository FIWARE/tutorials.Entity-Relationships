This tutorial teaches FIWARE users about batch commands and entity relationships.

#Contents

TBD

# Understanding Entities

This tutorial builds on the data created in the previous store finder example and creates and associates a series of related data entities to create a simple stock management system.

## Entities within a stock management system

Within the FIWARE platform, an entity represents the state of a physical or conceptural object which exists in the real world.

For a simple stock management system, we will only need four types of entity. The relationship between our entities is defined as shown:

![](https://fiware.github.io//tutorials.Entity-Relationships/entities.png)

* A **Store** is a real world bricks and mortar building. Stores would have properties such as:
  + A name of the store e.g. "Checkpoint Markt"
  + An address "Friedrichstraße 44, 10969 Kreuzberg, Berlin"
  + A phyiscal location  e.g. *52.5075 N, 13.3903 E*
* A **Shelf** is a real world device to hold objects which we wish to sell. Each shelf would have properties such as:
  + A name of the shelf e.g. "Wall Unit"
  + A phyiscal location  e.g. *52.5075 N, 13.3903 E*
  + A maximum capacity
  + An association to the store in which the shelf is present
* A **Product** is defined as something that we sell - it is conceptural object. Products would have properties such as:
  + A name of the product e.g. "Vodka"
  + A price e.g. 13.99 Euros
  + A size e.g. Small
* An **Inventory Item** is another conceptural entity, used to assocate products, stores, shelves and physical objects. It would have properties such as:
  + An assocation to the product being sold
  + An association to the store in which the product is being sold
  + An association to the shelf where the product is being displayed
  + A stock count of the quantity of the product available in the warehouse
  + A stock count of the quantity of the product available on the shelf


As you can see, each of the entities defined above contain some properties which are liable to change. A product could change its price, stock could be sold and the shelf count of stock could be reduced and so on.


# Application Overview

## Architecture

This application will only make use of one FIWARE component - the [Orion Context Broker](https://catalogue.fiware.org/enablers/publishsubscribe-context-broker-orion-context-broker) . Usage of the Orion Context Broker is sufficient for an application to qualify as “Powered by FIWARE”.

Currently, the Orion Context Broker relies on open source [MongoDB](https://www.mongodb.com/) technology to keep persistence of the context data it holds. Therefore, the architecture will consist of two elements:

* The Orion Context Broker server which will receive requests using NGSI
* The underlying MongoDB database associated to the Orion Context Broker server

Since all interactions between the two elements are initiated by HTTP requests, the entities can be containerized and run from exposed ports. 

![](https://fiware.github.io//tutorials.Entity-Relationships/architecture.png)

## Prerequisites

### Docker and Docker Compose 

To keep things simple both components will be run using [Docker](https://www.docker.com). **Docker** is a container technology which allows to different components isolated into their respective environments. 

* To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
* To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
* To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A [YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Getting-Started/master/docker-compose.yml) is used configure the required
services for the application. This means all container sevices can be brought up in a single commmand. Docker Compose is installed by default as part of Docker for Windows and  Docker for Mac, however Linux users will need to follow the instructions found [here](https://docs.docker.com/compose/install/)

### Postman (Optional)

 **Postman** is a testing framework for REST APIs. The tool can be downloaded from [here](www.getpostman.com). 
 
The text in this tutorial uses `cUrl` commands to interact with the Orion Context Broker server, however a [Postman collection](https://raw.githubusercontent.com/Fiware/tutorials.Getting-Started/master/FIWARE%20Getting%20Started.postman_collection.json)  of commands is also available within this GitHub repository. The entire tutorial is also available directly as [Postman documentation](https://documenter.getpostman.com/view/513743/fiware-getting-started/RVu5kp1c).


	