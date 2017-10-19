---
title: GoAt: A Data-Centric Interaction Skeleton for CAS in the Go Language
---

## What are attributes?
Attributes are the key concept of Attribute-based Communication (AbC, see LINK). AbC is a calculus where systems communicate between each other by message passing. Instead of defining a set of point-to-point channels, systems receive message from other systems according to their behaviour (i.e. the attributes): this is a desiderata of Collective Adaptive Systems (CASs).

As an example, think of a distributed system that depicts a room where two classes (chemistry and history) are held at the same time. There are two systems (the teachers, in AbC terms called components) that send messages (the lesson) to other systems (their students). Students can join or leave the classroom at any time, and can choose (dynamically) to listen at chemistry, history, both or none. In this scenario, the chemistry teacher sends its messages to students that have the attribute "listening to chemistry" set to true. The behaviour of the history teacher is similar. The student systems can just start at any time, set their attributes according to whether they attend chemistry/history/both/nothing (hence their behaviour) or terminate.

This simple example exposes some strengths of AbC:
1. Anonimity: teachers do not know their students, and viceversa. They only communicate because they are teaching/attending a speific lesson.
2. Scalability: teachers do not send a copy of the lesson to each student. Moreover, a third lesson (e.g. biology) can be held a the same time without changing anything in the protocol.


## What is Go?
TODO


## How do components exchange messages?
AbC does not state how messages are actually exchanged; the only requirement is that if a component is expected to receive a message, eventually it will be delivered and messages are delivered in the same order among the components. In GoAt, the set of software and logics behind the actual message exchange is called infrastructure. The infrastructure delivers a message to any component attached to it, by using TCP connections. The infrastructure, in order to enhance scalability, is implemented as a distributed system. 

GoAt implements many types of infrastructures: a centralized server, a cluster based infrastructure, a ring based infrastructure and a tree based infrastructure. For more information, see LINK. Since each type of infrastructure interacts in a different way with its components, a piece of software (called agent) is put between the infrastructure and the component. Each agent is specialised in interacting with a precise infrastructure type. The following figure depicts how the parts interact between each other.

![Part interaction](abc_component_infrastructure.svg)

## The infrastructures
Each infrastructure has its own distinctive features, hence we will describe them separately. Since the infrastructures presented here are distributed, you need to create one program for each node type. Before running the components, you need to make sure that the infrastructure is up and running.

### Single Server
This is the simplest infrastructure: only one node that receives and dispatches messages to other components.

![Single server](single_server.svg)

### Cluster infrastructure
This infrastructure has:
* a node that handles the registration procedure; it informs each serving node that a new component is available;
* a node that handles the message queue; the serving nodes share that message queue;
* a node that provides fresh message ids upon request;
* a set of serving nodes; they pick a message from the shared queue and distribute it to each registered components.
The following image depicts the registration procedure:

![Cluster infrastructure: registration procedure](cluster_reg.svg)

The following image depicts how messages are exchanged:

![Cluster infrastructure: message exchange](cluster_run.svg)

### Ring infrastructure
In this infrastructure, the serving nodes are connected between each other in a ring fashion. Each serving node has a next node, and it is the next node for some serving node. When a new component joins the infrastructure, its agent contacts the registration node. The registration node assigns the agent to a serving node. When the agent forwards a message, it sends the message to the associated serving node. The serving node forwards the message to the other agents assigned to it and to the next node. Each node forwards the message it receives to its agent and to its next node. When the message reaches the first node that forwarded it, it is discarded. This procedure removes the requirement of a centralised message queue. However, the issuance of the message ids is still performed by a single node.

The following image depicts how the registration procedure works:

![Ring infrastructure: registration procedure](ring_reg.svg)

The following image depicts how messages flow:

![Ring infrastructure: message flows](ring_connection.svg)

The following image depicts how a message is spread along the infrastructure when sent from an agent:

![Ring infrastructure: message speading](ring_msg.svg)

Summarising, to create a ring you need:
* a node that handles the registration procedure;
* a node that provides fresh message ids upon request;
* a set of serving nodes connected in a ring fashion.

### Tree infrastructure
In this infrastructure, the serving nodes are connected in a tree fashion. Each node (apart from one, called _root_) is connected to another serving node called _parent_. Each agent, to join the infrastructure, asks to the registration node to be associated with a serving node. When an agent wants to send a message, it asks to the associated node for a message id. Each node forwards the request for a new message id to its parent, unless it is the root. Then, the root assigns a fresh message id and forwards it to the child where the request came from. The message id is forwarded along the same path of the request (but in reversed order) so that the agent eventually receives it. After that, the agent emits the message to be sent (with the id it got). The message is sent to the associated serving node. Each node forwards the message to each node or agent associated with it but the node/agent where the message comes from. This infrastructure lifts the requrement of a special node that assigns message ids, as this task is performed by the root. It is easy to see that each message is delivered exactly once to each agent connected to the infrastructure (but the sender).

The following image depicts how the registration procedure works:

![Tree infrastructure: registration procedure](tree_reg.svg)

The following image depicts how nodes interact:

![Tree infrastructure: nodes interaction](ring_connection.svg)

The following image depicts how a message id request is carried out:

![Tree infrastructure: message id request](tree_mid.svg)

The following image depicts how a message is spread along the infrastructure when sent from an agent:

![Tree infrastructure: message speading](tree_msg.svg)

## How do I implement my own systems?
CASs that take advantage of AbC can be implemented either using the GoAt's API and the Go language or using an Eclipse plug-in that provides a language to describe the processes and a simple expression language. We stress that in both case there is a one-to-one mapping between GoAt semantics and AbC semantics, plus some macros to enhance the programming experience.

We provide two possible ways to implement system:
* using an Eclipse plugin; [tutorial](plugin.md)
* using the GoAt library directly and the Go language; [tutorial](library.md)

## How to use GoAt
To use GoAt, there is a fully configured virtual machine with all the tools needed properly set up.
LINK VM
