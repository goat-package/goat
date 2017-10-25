# Attribute-based Interaction in Google Go

## A Virtual Machine with Goat
We have already setup a machine to experiment with the goat API and the goat plugin. The plugin is a standalone application installed in the virtual machine. You only need to run and start working with directly. The virtual machine can be found here: [virtual machine](https://drive.google.com/open?id=0B9zaHQRMT9M3LVRUcXJnNG1EdXM).

## What are attributes?
Attributes are the key concept of Attribute-based Communication (AbC, see LINK). AbC is a calculus where systems communicate between each other by message passing. Instead of defining a set of point-to-point channels, systems receive message from other systems according to their behaviour (i.e. the attributes): this is a desiderata of Collective Adaptive Systems (CASs).

As an example, think of a distributed system that depicts a room where two classes (chemistry and history) are held at the same time. There are two systems (the teachers, in AbC terms called components) that send messages (the lesson) to other systems (their students). Students can join or leave the classroom at any time, and can choose (dynamically) to listen at chemistry, history, both or none. In this scenario, the chemistry teacher sends its messages to students that have the attribute "listening to chemistry" set to true. The behaviour of the history teacher is similar. The student systems can just start at any time, set their attributes according to whether they attend chemistry/history/both/nothing (hence their behaviour) or terminate.

This simple example exposes some strengths of AbC:
1. Anonimity: teachers do not know their students, and viceversa. They only communicate because they are teaching/attending a speific lesson.
2. Scalability: teachers do not send a copy of the lesson to each student. Moreover, a third lesson (e.g. biology) can be held a the same time without changing anything in the protocol.


## How do components exchange messages?
AbC does not state how messages are actually exchanged; the only requirement is that if a component is expected to receive a message, eventually it will be delivered and messages are delivered in the same order among the components. In GoAt, the set of software and logics behind the actual message exchange is called infrastructure. The infrastructure delivers a message to any component attached to it, by using TCP connections. The infrastructure, in order to enhance scalability, is implemented as a distributed system. 

GoAt implements many types of infrastructures: a centralized server, a cluster based infrastructure, a ring based infrastructure and a tree based infrastructure. For more information, see LINK. Since each type of infrastructure interacts in a different way with its components, a piece of software (called agent) is put between the infrastructure and the component. Each agent is specialised in interacting with a precise infrastructure type. The following figure depicts how the parts interact between each other.

![Part interaction](abc_component_infrastructure.svg)

## How do I implement my own systems?
CAS that take advantage of AbC can be implemented either using the GoAt's API and the Go language or using an Eclipse plug-in that provides a language to describe the processes and a simple expression language. We stress that in both case there is a one-to-one mapping between GoAt semantics and AbC semantics, plus some macros to enhance the programming experience.

Below we discuss Goat, its plugin, and the supported infrastructures:
* The Eclipse plugin; [tutorial](plugin.md)
* The GoAt API in Google Go; [tutorial](library.md)
* The supported infrastructures; [link](infrastructure.md)

## Stable Allocation
[Stable Allocation](stable_allocation.pdf)
