# Amberdata Inbound API

Amberdata provides monitoring, alerting and analytics for blockchains.

The Amberdata platform can accept data from different blockchains.  In fact, after working with and researching a few different protocols and implementations (Ethreum, Aion, Stellar, etc), we have found some commonalities between them (and some differences!), and we have developed a global data model. 

A more complete documentation on the domain model can be found here: [Inbound domain model](https://github.com/amberdata/ingestion-domain-model/blob/master/README.md#getting-started).

The model defines the different entities and object models that 3rd parties can use to push data into the Amberdata platform via the Inbound API.

Once ingested into the Amberdata platform, these entities are processed by our backend pipeline, where we combine these data sources with off-chain data, extract metrics, and provide insights, analytics and monitoring into the ingested blockchain.

A full Swagger documentation of the REST API is available [here](https://blockchains.amberdata.io/api-explorer) and [here](https://blockchains.amberdata.io/api/v1/spec).

# Ingestion API Core

This `ingestion-core` project is a library that one can use when developing a new inbound module.  You can think of it as a set of utilities which simplify interacting with the Inbound API directly. 

It can be included in any SpringBoot 2.x application, and provides easier means to:
  - create valid requests for objects and sending them to the Inbound API 
  - automatically re-sends requests on error
  - provides a basic state storage to track which metrics have already be sent

The main java class is called `IngestionApiClient` and is used to publish metrics instead of interacting with an HTTP client directly.  More details available in the [Getting Started](#getting-started) section.

# Usage

> Here should be instructions of how to add maven repository which contains ingestion-domain-model artifact

## Use in another project

To add the Amberdata Inbound Domain Model to you Maven project, add this dependency to your `pom.xml`:
```xml
<dependency>
    <groupId>io.amberdata.ingestion</groupId>
    <artifactId>ingestion-core</artifactId>
    <version>0.0.4</version>
</dependency>
```

Or to your `build.gradle`:
```gradle
dependencies {
    implementation 'io.amberdata.ingestion:ingestion-core:0.0.4'
}
```

## Build locally

```sh
$ git clone https://github.com/amberdata/ingestion-core.git
$ cd ingestion-core
$ mvn clean install
```

# Getting Started

The `IngestionApiClient` can be instantiated as a Spring framework component, and needs to configured accordingly to your own blockchain.

> Note that you have to have register your blockhain before you start publishing metrics. To do that, follow the instructions: *put it here*

Three properties control where and how the the metrics are published to the Inbound API: 
```properties
ingestion.api.url=https://blockchains.amberdata.io/api/v1
ingestion.api.blockchain-id=<blockchain_id_goes_here>
ingestion.api.api-key=<api_key_goes_here>
```

Once you have implemented your own inbound module, you can either:
- set these properties in your own application configuration
- or pass them as parameters to your application at start time, for example: 
```bash
$ java -jar <application>.jar                                 \
  --ingestion.api.url=https://blockchains.amberdata.io/api/v1 \
  --ingestion.api.blockchain-id=<blockchain_id_goes_here>     \
  --ingestion.api.api-key=<api_key_goes_here>
```

## Application configuration

To help your SpringBoot application find the `IngestionApiClient` class, you need to specify the package.
The simplest way to do so is referring to `io.amberdata.ingestion.core.IngestionCore` marker interface in` @ComponentScan` annotation:
```java 
@SpringBootApplication
@ComponentScan(basePackageClasses = IngestionCore.class)
public class InboundModuleDemoApplication {
    public static void main (String[] arguments) {
        SpringApplication.run(Application.class, arguments);
    }
}
```

## Retrieving the Inbound Client

And with that, it is now possible to inject the configured `IngestionApiClient` instance as a dependency: 
```java 
@Component
public class BlocksPublisher {
    private final IngestionApiClient ingestionApiClient;
  
    @Autowired
    public BlocksPublisher (IngestionApiClient ingestionApiClient) {
        this.ingestionApiClient = ingestionApiClient;
    }
}
```

To send blockchain objects to the Inbound API, the `publish` method is called with the appropriate context, for example:
```java 
ingestionApiClient.publish("/blocks", entities); // sends a list of block entities to the Inbound API endpoint 
```
Note than you can send an entity (e.g. block) or a list of entities. 

## Working with state

### Saving state

In order to maintain state, each entity can (and probably should) be wrapped into a `BlockchainEntityWithState`, which contains information about the state:
```java 
Block block = new Block.Builder().number(12345L).build();

BlockchainEntityWithState blockWithState = BlockchainEntityWithState.from(
    block,
    ResourceState.from("Block", "12345")
);

ingestionApiClient.publish("/blocks", blockWithState);
```

The `BlockchainEntityWithState` automatically:
- publishes the block to the Inbound API
- stores its number to the internal storage 

The latter is for error handling, or in case of crashes, to guarantee that the collection can resume automatically where it left off.

In the example above, `ResourceState.from("Block", "12345")` provides a generic means to store state:
- the 1st argument (`Block`) can be any string which is used to identify the type of entity
- the 2nd argument (`12345`) can be any string which is used as the identifier of the entity

### Restoring state

Whenever you need information about the last published blockchain entity (e.g. block) you can get it from the internal storage, with an instance of the `io.amberdata.ingestion.core.state.ResourceStateStorage` component, which can be injected into any SpringBoot application:
```java 
@Component
public class BlocksPublisher {
    private final IngestionApiClient   ingestionApiClient;
    private final ResourceStateStorage stateStorage;
  
    @Autowired
    public BlocksPublisher (IngestionApiClient ingestionApiClient, ResourceStateStorage stateStorage) {
        this.ingestionApiClient = ingestionApiClient;
        this.stateStorage       = stateStorage;
    }
}
```

Assuming there is already some state information stored in the internal storage (as soon as some entities have been successfully published to the Inbound API for example)
you can get the latest state by calling the `getStateToken` method, which returns the state's token previously created with `ResourceState.from(...)`;

```java 
stateStorage.getStateToken("Block", () -> "<token_default_value>"); 
```

> Note, that if no token is found for the key (`Block` in the example) being requested, 
the supplier (passed as a second parameter) will be invoked and its value will become the method's return value (`<token_default_value>`).

# Implementation example

Several examples of Inbound Modules using the `ingestion-core` library are available:
  - [Stellar Inbound Module](https://github.com/amberdata/stellar-ingestion-api-module) for collecting [Stellar](https://www.stellar.org) data  

# Licensing

This project is licensed under the [Apache Licence 2.0](./LICENSE).

See also [Contributing](./CONTRIBUTING.md)
