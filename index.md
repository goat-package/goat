# GoAt: Go powered with Attributes


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


## How do I implement my own systems?
CASs that take advantage of AbC can be implemented either using the GoAt's API and the Go language or using an Eclipse plug-in that provides a language to describe the processes and a simple expression language. We stress that in both case there is a one-to-one mapping between GoAt semantics and AbC semantics, plus some macros to enhance the programming experience.

### Using GoAt's API and Go
First of all, you need to install the `goat-plugin/goat/goat` package. 
#### How to define a component
The skeleton of a component definition follows:

    package main

    import (
        "goat-plugin/goat/goat"
    )

    func main(){
        environment := map[string]interface{}{...}
        agent := goat.New...Agent(...)
        comp := goat.NewComponentWithAttributes(agent, environment)
        goat.NewProcess(comp).Run(func(p *goat.Process) {
            ...
	    })
    }

`environment` is a map that contains the set of attributes (and corresponding values) that the component has at startup. Attribute names must be strings, while the associated values can be of any type. Note that associating an attribute to `nil` is not the same as not having that attribute associated to anything. `agent` is the agent that acts as an adapter between the infrastructure and the component. It's constructor is different according to the infrastructure that the component participates to:

Infrastructure | Agent constructor |
---|---|
Single Server | `goat.NewSingleServerAgent(serverAddress)` |
Cluster | `goat.NewClusterAgent(messageQueueAddress, registrationAddress)` |
Ring | `goat.NewRingAgent(registrationAddress)` |
Tree | `goat.NewTreeAgent(registrationAddress)` |

`comp` is the component. It contains both the dynamic behaviour (the process) and the set of attributes (called environment). Its constructor `goat.NewComponentWithAttributes` requires to state both the agent you want to use (hence, indirectly, which infrastructure you are connecting to) and the initial setup of the environment. If you do not want to set any attributes at the beginning, you can use the `goat.NewComponent` constructor whose only parameter is the agent. Note that an agent must be associated with at most one component: that is, for each component you have to instantiate one different agent. ` goat.NewProcess(comp).Run(func(p *goat.Process) {...})` sets up the behaviour of the component and starts it. The ellipsis must be replaced by the behaviour you want that component to have. `p` represents the `goat.Process` object, that is the access point to the functions that allows the specification of the behaviour.
#### How to define a process with the `goat.Process` API?
The `goat.Process` API was designed to match as much as possible the AbC constructs. In the following, we assume that `proc` is an object of type `goat.Process`.

`proc.Spawn(procFnc func(p *goat.Process))` creates a new process that belongs to the `proc`'s component that will run concurrently with `proc`. This means that the two processes share the same environment, but they cannot exchange message between each other (according to AbC semantics). `procFnc` defines the new process' behaviour.

`proc.WaitUntilTrue(todo func(*Attributes) bool)` halts `proc` until the `todo` function returns `true`. `todo` is allowed to alter the environment (via its argument), hence it can be used to implement an update and/or awareness construct.

`proc.SendOrReceive(chooseFnc func(attr *Attributes, receiving bool) SendReceive)` is the call used to send or receive a message. `chooseFnc` is a function that, given the environment `attr`:
* if `receiving` is `true`, it can return 
  - `goat.ThenFail()` if it does not want to receive a message because of the environment; this process won't receive any other message until the environment changes.
  - `goat.ThenReceive(accept func(attr *Attributes, msg Tuple) bool)` if, according to the environment, a message might be accepted; the `accept` function returns `true` iif the message `msg` is to be accepted and updates accordingly the environment.
* if `receiving` is `false`, it can return:
  - `goat.ThenFail()` if it does not want to send a message because of the environment; this process won't retry to send a message until the environment changes.
  - `goat.ThenSend(msg Tuple, msgPred Predicate)` if it wants to send a message `msg` (that can be created with `goat.NewTuple(part1, part2, ...)`) with the predicate `pred`. Predicates are defined with the constructors `goat.And{p1, p2}`, `goat.Or{p1, p2}`, `goat.Not{p1}`, `goat.Comp{Par1, IsAttr1, Op, Par2, IsAttr2}`. In `goat.Comp{Par1, IsAttr1, Op, Par2, IsAttr2}`, `Par1`(`Par2`) is the first (second) item in the comparison, `IsAttr1` (`IsAttr2`) indicates if the corresponding item to be compared is the actual value (`IsAttrx = false`) or it is provided as the attribute name to be found in the receiving component (`IsAttrx = true`); `Op` indicates which type of comparison must be performed (`<`,`<=`,`==`,`!=`,`>=`,`>`).

The environment can be set using the API of the `goat.Attributes` struct. The `Get(x)` method checks if the attribute `x` is set in the environment: if yes, it will return the value of attribute `x` and the `true` value, otherwise it will return `nil` and `false`. The `Set(x, v)` sets the attribute `x` to value `v`. Note that if the call in which the environment is altered fails, the updates are not saved.

As an example, let's see how we can model a teacher of chemistry that is holding a lesson and at the same time can answer to individual questions posed by the students.

    package main

    import (
        "goat-plugin/goat/goat"
    )

    type Question struct {
        question string
        sender int
    }

    func main(){
        attrInit := map[string]interface{}{"teaching": "chemistry"}
        agent := goat.New...Agent(...)
        comp := goat.NewComponentWithAttributes(agent, attrInit)
        goat.NewProcess(comp).Run(func(p *goat.Process) {
            questions := []Question{}
            for{
                p.SendOrReceive(func(attr *goat.Attributes, receiving bool) goat.SendReceive{
                    if receiving {
                        return goat.ThenReceive(func(attr *goat.Attributes, msg goat.Tuple) bool{
                            if msg.IsLong(3) && msg.Get(0) == "question" {
                                question := msg.Get(1).(string)
                                sender := msg.Get(2).(int)
                                questions = append(questions, Question{question, sender})
                                return true
                            }
                            return false
                        })
                    } else {
                        if len(questions) > 0 {
                            ans := answer(questions[0].question)
                            receiver := questions[0].sender
                            questions = questions[1:]
                            return goat.ThenSend(goat.NewTuple("answer",ans), goat.Comp{"id", true, "==", receiver, false})
                        } else {
                            lesson := nextLessonPart()
                            return goat.ThenSend(goat.NewTuple("lesson",lesson), goat.Comp{"attendingChemistry", true, "==", true, false})
                        }
                    }
                })
            }
	    })
    }

Now, the following program models a chemistry students that attends the lesson and might ask questions to the teacher:

    package main

    import (
        "goat-plugin/goat/goat"
    )

    func main(){
        attrInit := map[string]interface{}{"attendingChemistry": true, "id": 1}
        agent := goat.New...Agent(...)
        comp := goat.NewComponentWithAttributes(agent, attrInit)
        goat.NewProcess(comp).Run(func(p *goat.Process) {
            p.Spawn(func(p *goat.Process) {
                for{
                    // .......
                    question := //....
                    var answer string
                    p.SendOrReceive(func(attr *goat.Attributes, receiving bool) goat.SendReceive{
                        if receiving {
                            return goat.ThenFail()
                        } else {
                            id, _ := attr.Get("id")
                            return goat.ThenSend(goat.NewTuple("question", question, id), goat.Comp{"teaching", true, "==", "chemistry", false})
                        }
                    })
                    p.SendOrReceive(func(attr *goat.Attributes, receiving bool) goat.SendReceive{
                        if receiving {
                            return goat.ThenReceive(func(attr *goat.Attributes, msg goat.Tuple) bool{
                                if msg.IsLong(2) && msg.Get(0) == "answer" {
                                    answer = msg.Get(1).(string)
                                    return true
                                }
                                return false
                            })
                            
                        } else {
                            return goat.ThenFail()
                        }
                    })
                    // .......
                }
            })
            for{
                // .......
                var lessonPart string
                p.SendOrReceive(func(attr *goat.Attributes, receiving bool) goat.SendReceive{
                    if receiving {
                        return goat.ThenReceive(func(attr *goat.Attributes, msg goat.Tuple) bool{
                            if msg.IsLong(2) && msg.Get(0) == "lesson" {
                                lessonPart = msg.Get(1).(string)
                                return true
                            }
                            return false
                        })
                        
                    } else {
                        return goat.ThenFail()
                    }
                })
                // .......
            }
	    })
    }
    
### Using the Eclipse plugin
TODO

## How to instantiate an infrastructure
Each infrastructure has its own distinctive features, hence we will describe them separately. Since the infrastructures presented here are distributed, you need to create one program for each node type. Before running the components, you need to make sure that the infrastructure is up and running.

### Single Server
This is the simplest infrastructure: only one node that receives and dispatches messages to other components.

![Single server](single_server.svg)

This type of infrastructure does not scale, since it has only one central node. It was implemented only for debugging purpouses. It can be instantiated with:

    package main

    import (
        "goat-plugin/goat/goat"
    )

    func main(){
        port := 17000
        var timeoutMsec int64 = 15000
        term := make(chan struct{})
        goat.RunCentralServer(port, term , timeoutMsec)
        <-term
    }
    
Components can connect to the infrastructure by calling `goat.NewSingleServerAgent("<serverAddress>:<port>")` with the address of the central node and the listening port provided (here 17000).

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

The following code is used to instantiate the registration node

	package main

	import (
	    "goat-plugin/goat/goat"
	)

	func main(){
	    port := 17000
	    nodesAddresses := []string{} // list of all the serving nodes in the cluster
	    chnTimeout := make(chan struct{})
	    go goat.NewClusterAgentRegistration(port, "<messageQueueAddress>:<mqPort>", nodesAddresses).Work(0, chnTimeout)
	    <-chnTimeout
	}

The following code instantiates the message queue

	package main

	import (
	    "goat-plugin/goat/goat"
	)

	func main(){
	    port := 17001
	    chnTimeout := make(chan struct{})
	    go goat.NewClusterMessageQueue(port).Work(0, chnTimeout)
	    <-chnTimeout
	}
	
The following code instantiates the provider of fresh message ids

	package main

	import (
	    "goat-plugin/goat/goat"
	)

	func main(){
	    port := 17002
	    chnTimeout := make(chan struct{})
	    go goat.NewClusterCounter(port).Work(0, chnTimeout)
	    <-chnTimeout
	}

The following code instantiates a serving node

	package main

	import (
	    "goat-plugin/goat/goat"
	)

	func main(){
	    port := 17003
	    chnTimeout := make(chan struct{})
	    messageQueueAddress := "..."
	    freshMidAddress := "..."
	    registrationAddress := "..."
	    go goat.NewClusterNode(port, messageQueueAddress, freshMidAddress, registrationAddress).Work(0, chnTimeout)
	    <-chnTimeout
	}
	
Components can connect to the infrastructure by calling `goat.NewClusterAgent("<messageQueueAddress>:<port>", "<registrationAddress>:<port>")` with the address of the message queue node and the registration node with the ports provided (here 17001 and 17000).
	
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

The following code is used to instantiate the registration node

	package main

	import (
	    "goat-plugin/goat/goat"
	)

	func main(){
	    port := 17000
	    nodesAddresses := []string{} // list of all the serving nodes in the cluster
	    chnTimeout := make(chan struct{})
	    go goat.NewRingAgentRegistration(port, nodesAddresses).Work(0, chnTimeout)
	    <-chnTimeout
	}

The following code instantiates the provider of fresh message ids

	package main

	import (
	    "goat-plugin/goat/goat"
	)

	func main(){
	    port := 17001
	    chnTimeout := make(chan struct{})
	    go goat.NewClusterCounter(port).Work(0, chnTimeout)
	    <-chnTimeout
	}

The following code instantiates a serving node

	package main

	import (
	    "goat-plugin/goat/goat"
	)

	func main(){
	    port := 17002
	    chnTimeout := make(chan struct{})
	    freshMidAddress := "..."
	    nextNodeAddress := "..."
	    go goat.NewRingNode(port, freshMidAddress, nextNodeAddress).Work(0, chnTimeout)
	    <-chnTimeout
	}

Components can connect to the infrastructure by calling `goat.NewRingAgent("<registrationAddress>:<port>")` with the address of the registration node and the listening port provided (here 17000).

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
## Installing GoAt
TODO

## Installing the Eclipse plugin
TODO
