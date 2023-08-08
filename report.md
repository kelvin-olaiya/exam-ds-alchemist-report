# Alchemist simulation batches distribution

## Abstract
Alchemist is an open-source and general-purpose simulator developed in
the University of Bologna. Simulation can be executed by writing their configurations in YAML. Sometimes it may be useful to execute the same configuration with different parameters, called variables. The set of simulation
differing by their variables constitute a batch. Alchemist provides a way to
launch a simulation batch sequentially. The aim of this project is to implement in Alchemist a batch distribution mechanism in order to parallelize
batch simulations computation and reduce the time spent in executing the
batch.

## Analysis

In this section the meta-model the Alchemist simulator will be reported. Afterwards, once the functional and non-functional requirements have been formalised, considerations will be made on the distribution aspects.

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

### Requirements

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


## Technologies

Apache Ignite has been choosen to take care of cluster formation, load balancing, distributed file system

I think etcd will be usefull to store configuration data

## Design

For the result collection it could be useful to create a Global Exporter class


### Architecture

