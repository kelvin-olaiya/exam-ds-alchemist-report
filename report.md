# Alchemist simulation batches distribution

## Abstract
Alchemist is an open-source and general-purpose simulator developed in
the University of Bologna. Simulation can be executed by writing their configurations in YAML. Sometimes it may be useful to execute the same configuration with different parameters, called variables. The set of simulation
differing by their variables constitute a batch. Alchemist provides a way to
launch a simulation batch sequentially. The aim of this project is to implement in Alchemist a batch distribution mechanism in order to parallelize
batch simulations computation and reduce the time spent in executing the
batch.

## Analysis

### Alchemist's meta-model

![Alchemist's meta-model](assets/img/alchemist-meta-model.svg)

The main element's that compose the Alchemist meta-model are:

* **Molecule**: the name that is associated with a data item. 
* **Concentration**: the value associated to a particular *molecule*.
* **Node**: a container of *molecules* and *reaction* that lives inside an *environment*.
* **Environment**: a space abstraction. It contains nodes and it is able to tell where nodes are (thier position), 
distance between two nodes and optionally gives support for moving nodes within the environment.
* **Linking rule**: a function based on the current state of the environement that associate to each node a *neighborhood*.
* **Neighbourhood**: entity composed of a node (center) and a set of nodes (neighbors).
* **Reaction**: any event that can change the status of the *environment*. It is defined by a possibly empty set of *conditions*, one or more actions and a *Time distribution*. Toghether with a *static rate* parameter, the value of each condition, a *rate equation* giving back an *instantaneous rate* by combining the static rate and the value of conditions, the *time distribution* determines the frequency at which the reaction will happen.
* **Condition**: a function that outputs a boolean and a number based on the current state of the *environment*
* **Action**: models a change in the environment.

#### Incarnations

The meta-model is instantiated by one of the available incarnations that are:

* **Protelis**
* **SAPERE**
* **Biochemistry**
* **Scafi**

An incarnation of Alchemist includes a type definition of concentration, and possibly a set of specific conditions, actions and (rarely) environments and reactions that operate on such types. More information could be found on the Alchemist official website.

### Requirements analysis

The Alchemist simulator already provide means for writing simulation configurations meant to be run locally. It also provide modes for executing batch of simulation. A simulation batch is a set of simulations, each differing by one ore more variables. This enables to test or execute the same configuration with different parameters. Running a simulation can be time consuming let alone running a simulation batch. Taking advange of multiple computing resource can be a way to reduce the time necessary to complete a simulation batch. The following functional requirement have been identified:

* It should be possible to create a cluster of nodes, each executing a service exposing
Alchemist.
* Alchemist should provide a way to distribute a batch of simulations to be executed
by one or more nodes on a cluster.
* Each node of a cluster must be up and ready to receive and execute configurations
of simulations.
* None of the distributed simulations should get lost, meaning that in case of a node
failure, a recovery mechanism should redistribute the simulations assigned to the
failing node.
* Once a simulation is computed by a node, results should be made available to the
user who launched the distribution.

### Q/A

### Scenarios

There are two main kinds of scenarios:

* A user launches Alchemist in *server mode*, adding it self to an existing cluster, or creating a new one.

* A user lunches Alchemist to distribute a batch of simulation across nodes in a cluster.

![](assets/img/Use_cases.png)


### Self assessment policy

Assessment will be made by means of automatic unit and integration testing. Testing should cover at least the requirements reported in the analysis.


## Design

### Architecture

![](assets/img/Architecture.png)

The project architecture follow a client-server style. Communication between parties is made trough message exchange. Thus the need of Message broker, responsible of message delivery.
All information necessary for the correct functioning of the system is held by a registry.  

### Structure

In the following the main entities componing the domain model will be described.

![Model design](assets/img/Design-structure.png)

* **Cluster** is an entity representing the collection of nodes that are currently connected forming a cluster. Through the cluster it is possible to obtain a `Dispatcher`, specifing the complexity that the nodes in the dispatcher should be able to handle. 

* **ClusterNode** represent a server node to which jobs can be distributed.

* **Dispatcher** contains a subset of the nodes in the cluster. It is responsible of accepting `SimulationBatches` and distribute them across subset of nodes. Distribution is made according to a `DispatchStrategy`

* **DispatchStrategy** it models the strategy with which the work load gets distributed to a collection of nodes (e.g *round-robin*).

* **Complexity** describes the complexity in terms of ram usage and memory occupation for the simulations in a batch.

* **SimulationBatch** represents a simulation batch with its complexity. It is compose of a simulation configuration and a collection on simulation initializers.

* **SimulationConfig** contains the general batch information such as the end step and end time of the simulations and a loader from which simulation instances will be created. `Dependencies` are files that must me made available to all servers in order to execute the simulation correctly.

* **SimulationInitializer** containts a combination of variables values what will be use to create a simulation instance. To every simulation initializer in a simulation batch will correspond a job for the cluster nodes.

* **BatchResult** it models the result of a simulation batch that have been submitted via a `Dispatcher`. It gives information on the total number of result that have occured while executing the batch and an utility method to save all the distributed export files locally. 

* **SimulationResult** it the result of a single job. 


### Behavior

#### Alchemist - server side
![Alchemist server behavior](assets/img/AlchemistServer-state.png)

When Alchemist is launched in server mode, the server initially is in IDLE state. During this time the server is detecting any cluster node fault. It wait for job orders to execute and immidiately executes then when it receives them. When all jobs have been executed the server return to IDLE state and so on.

![Alchemist client behavior](assets/img/AlchemistClient-state.png)

Client side, when Alchemist is launched in distributed batch mode, 
the first thing happening is the building of the various simulation each corresponding to a job. When all simulation have been built, they are distributed to the cluster nodes. Client is then in WAITING state.While waiting the client will run a fault detection routine. In case a fault is detected, jobs dispatched to the faulty server are redistributed across the remaining cluster nodes. If no more node are available the user is notified of the error.

### Interactions

#### Fault detection

![Cluster fault detection interaction](assets/img/ClusterFaultDetector.png)

In order to be able to respond to failure a simple fault detection routine has been designed.
Every node in the cluster checks for the liveness of other nodes. 
When a server AS01 want to see if another server AS02 is still running, it will send a healthCheckRequest to that server. If AS02 response withing the timeout period, then AS01 will consider it still alive. Otherwise the server is considered faulty and will be removed from the registry (Actually AS01 may send multiple health check requests and decide that after a number of missing replies AS02 is dead).

#### Batch distribution

![Distribution interaction](assets/img/Simulation%20distribution%20-%20interactions.png)

As for the distribution of the simulation batch, when a user launches the client firstly the client builds all the simulation. When this is done, the various simulation are submitted to the registry. Then the client sends a job order to the servers that were previously selected. 
When a server receives the job order, it will retrieve the simulation from the registry, run it and submit the results back to the registry. After that, it will notify to the client that the job had been executed. At this point the client can get the results from the registry and make them available to the user. 


## Implementation details

### Technologies

#### Etcd

Etcd is a distributed, reliable and strongly consistent key-value store. It has been used to store the most important data for the functioning of the system (The registry). In theory, if all nodes in the cluster chrashes, just by using the information stored in the registry it could be possible to resume any job that was executing before failure. Thus it is the most important technology that had been used in the project.

![](assets/img/Registry-KVStore.png)

#### RabbitMQ

RabbitMQ is an open-source message broker based on the Advanced
Message Queuing Protocol (AMQP) for reliable communication. RabbitMQ funziona come un intermediario. It supports point-to-point and publish/subscribe message patterns.

![Communication queues](assets/img/Communication%20Queues.png)

For this project a point-to-point has been chosen. In particular Every server has two message queues: one for receiving job orders and another to receive health check requests.

#### Protocol Buffers

Protocol Buffers are a language-neutral, platform-neutral extensible mechanism for serializing structured data. Once the structure of the data has been defined it is possible to use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages.

It has been used in this project for the serialization and deserialization of data for persistence in the registry and for message exchange.


## Self-assessment

![Test results](assets/img/Test_results.png)

The main project requirements have been tested using the Kotest testing framework. For the distribution tests, the non-deterministic testing features have been used. They include primitives such as:

```kotlin
eventually(duration) {
    // code that gets executed for the specified duration
    // until no exception is thrown
}

until(duration) {
    // Boolean expression that should be true before the time passes the duration.
}
```

These made the testing of the interaction between client and server much easier and idiomatic.

## Deployment

### Usage example

## Conlusion

### Future works

### What I've learned


