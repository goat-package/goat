# Programming with the GoAt Go library
Using the library, you can write Go programs that take advantage of the attribute-based communication paradigm. 

## How to define a component
The skeleton of a component definition follows:

    package main

    import (
        "github.com/goat-pakage/goat/goat"
    )

    func main(){
        environment := map[string]interface{}{...}
        agent := ...
        comp := goat.NewComponentWithAttributes(agent, environment)
        goat.NewProcess(comp).Run(func(p *goat.Process) {
            ...
	    })
    }

`environment` is a map that contains the set of attributes (and corresponding values) that the component has at startup. Attribute names must be strings, while the associated values can be of any type. Note that associating an attribute to `nil` is not the same as not having that attribute associated to anything. `agent` acts as an adapter between the infrastructure and the component. It can be seen as the access point to the infrastructure from the component. It's constructor differs according to the infrastructure that the component participates to:

Infrastructure | Agent constructor |
---|---|
Single Server | `goat.NewSingleServerAgent(serverAddressAndPort)` |
Cluster | `goat.NewClusterAgent(messageQueueAddressAndPort, registrationAddressAndPort)` |
Ring | `goat.NewRingAgent(registrationAddressAndPort)` |
Tree | `goat.NewTreeAgent(registrationAddressAndPort)` |

`comp` is the component. It contains both the dynamic behaviour (the process) and its state (the set of attributes, called environment). You can create a component by calling:
* `goat.NewComponentWithAttributes(agent, environment)`, to create a new component linked to the infrastructure via the agent `agent`; its environment is set according to the map `environment`;
* `goat.NewComponent`, to create a new component linked to the infrastructure via the agent `agent`; its environment empty.

Note that an agent must be associated with at most one component: that is, for each component you have to instantiate one different agent. 

` goat.NewProcess(comp).Run(func(p *goat.Process) {...})` sets up the behaviour of the component and starts it. The ellipsis must be replaced by the behaviour you want that component to have. `p` represents the `goat.Process` object, that is the access point to the functions that allows the specification of the behaviour.
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
        "github.com/goat-pakage/goat/goat"
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
        "github.com/goat-pakage/goat/goat"
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

## How to instantiate an infrastructure
Since the infrastructures presented here are distributed, you need to create one program for each node type. Before running the components, you need to make sure that the infrastructure is up and running.

### Single Server
It can be instantiated with:

    package main

    import (
        "github.com/goat-pakage/goat/goat"
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
* a node that handles the registration procedure;
* a node that handles the message queue;
* a node that provides fresh message ids upon request;
* a set of serving nodes.

The following code is used to instantiate the registration node

	package main

	import (
	    "github.com/goat-pakage/goat/goat"
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
	    "github.com/goat-pakage/goat/goat"
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
	    "github.com/goat-pakage/goat/goat"
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
	    "github.com/goat-pakage/goat/goat"
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
This infrastructure has:
* a node that handles the registration procedure;
* a node that provides fresh message ids upon request;
* a set of serving nodes.

The following code is used to instantiate the registration node

	package main

	import (
	    "github.com/goat-pakage/goat/goat"
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
	    "github.com/goat-pakage/goat/goat"
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
	    "github.com/goat-pakage/goat/goat"
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

The following code is used to instantiate the registration node

	package main

	import (
	    "github.com/goat-pakage/goat/goat"
	)

	func main(){
	    port := 17000
	    nodesAddresses := []string{} // list of all the serving nodes in the cluster
	    chnTimeout := make(chan struct{})
	    go goat.NewTreeAgentRegistration(port, nodesAddresses).Work(0, chnTimeout)
	    <-chnTimeout
	}

The following code instantiates the root node

	package main

	import (
	    "github.com/goat-pakage/goat/goat"
	)

	func main(){
	    port := 17001
	    chnTimeout := make(chan struct{})
	    childNodesAddresses := []string{...}
	    go goat.NewTreeNode(port, "", childNodesAddresses).Work(0, chnTimeout)
	    <-chnTimeout
	}
	

The following code instantiates the a non-root serving node

	package main

	import (
	    "github.com/goat-pakage/goat/goat"
	)

	func main(){
	    port := 17002
	    chnTimeout := make(chan struct{})
	    childNodesAddresses := []string{}
	    parentAddress := ...
	    go goat.NewTreeNode(port, parentAddress, childNodesAddresses).Work(0, chnTimeout)
	    <-chnTimeout
	}

Components can connect to the infrastructure by calling `goat.NewTreeAgent("<registrationAddress>:<port>")` with the address of the registration node and the listening port provided (here 17000).
