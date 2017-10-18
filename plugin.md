# Programming with the GoAt Eclipse plugin
We developed the Eclipse plugin to give the user the power to describe CAS interaction schemes in a language agnostic way. Another point we stress is that even the interaction scheme is independent from the message exchange infrastructure.

When launching the plugin, you get this screen

![GoAt plugin at startup](goat_startup.png)

On the left, you have the set of projects that are in your workspace. In GoAt, a project is a description of your system in term of components, interaction and infrastructure. You can create a new project by right-clicking on the Project Explorer, New > Project... . Among the wizards, choose Goat > Goat project, then click Next and give it a name (e.g. "a\_project").

Now, in the Project Explorer, expand the project tree.

![Project tree](project_tree.png)

The `src` directory contains two types of files: `.ginf` and `.goat`. `.ginf` files describe the communication infrastructure between the component (that is, how the messages are delivered), while `.goat` files describe the components of the system and how they interact. We require the infrastructure details to be kept away from the components because:
* we want to stress that the component behaviour is independent from how the messages are delivered;
* they might be developed by different users: the system administrator configures the infrastructure, and the programmer develops the components.

The only linking point is the `infrastructure` statement in `.goat` files which states that the components described inside use a specific named infrastructure.

The `src-gen` directory contains the go code generated from the code generation procedure that corresponds to the infrastructure(s) and component(s) described in the project. When a system is run, the plugin calls Go and runs the files inside this directory. Note that component and infrastructure files are kept separated.

## Running a project
This section describes how to run a GoAt project. To do this, we provide a simple system just to see some interaction. Next sections describe the specification language.

Open `infrastructure.ginf` and paste this code, then save the file (CTRL+S):
```
singleserver infr {
  server : "127.0.0.1:17654"
}
```
Open `system.goat` and paste this code, then save the file:
```
infrastructure infr

process P {
  send{"hello world"} @ (true);
}

process Q {
  receive(true){x} print("Got $x$");
}

component {} : P
component {} : Q
```
This code describes:
* a centralised infrastructure whose address is `127.0.0.1:17654`;
* a system made of two component, where one sends `hello world` to the other, using the infrastructure.

The plugin offers an editor with syntax higlighting and error checking. Try, for example, to change `send` to `sed` and you will get an error.

Right-click on the project, then click Run As > Run System. Now you get two consoles in the Console view, and you can switch between them with this drop-down menu ![](console.png). The consoles contain the standard output (in black) and standard error (in red) of the system and of the infrastructure. Open the `system.goat` console. It contains `Got hello world`. The component whose process is P has sent the `hello world` message to every component (`true` predicate). The component whose process is Q received the message and printed it out with the `print` directive. Since both the components terminated, also the system terminated. Now open the `infrastructure.ginf` console. The only output line is `Started`, to singal that the infrastructure started. The infrastructure is still running, waiting for new messages. You can stop it by pressing the Terminate button (![](stop.png)). Before closing the plugin or running other systems, remember to stop all running infrastructure and components as they might interefere with the new runs.

## Programming components
This section describes how to program the components in `.goat` files. Components are the parts of the system that interact in order to attain a global behaviour. The first part describes the main constructs that can be used; the remainder describes some  patterns that recur during the development.

### GoAt constructs
A `.goat` file begins with `infrastructure infr` where `infr` is the name of an infrastructure declared in the project. This statement says that the components described in this file are to be attached to the infrastructure `infr`.

The remainder of the file is a set of process, environment, component and function declarations.
* The processes represent the dynamic behaviour of components. They can send and/or receive mesages, access the component environment and use local attributes. Local attributes are not shared between processes, even if they are on the same component. 
* The environment is a map between names (called component attributes) and values. The environment represent the state of a component. The environment is not shared between components, but all processes running on a component can read and write over it.
* The component is the pair of an environment and a process. Components interact between each other by message passing according to their state.
* Functions are used to compute values, that can be sent with messages, saved in attributes or used in predicates.

### Process
A process is defined as 
```
process P {
  ...statement 1...
  ...statement 2...
  
  ...statement n...
}
```
`P` is the name of the process. The statements are executed one after another, and when the last statement terminates the process terminates. The available statements are: `send`, `receive`, `waitfor`, `set`, `if`, `call`, `spawn` and `loop`.

#### `send`
The `send` statement is used to send a message. Its syntax is 

```send{expr1, expr2, ..., exprn} @ (pred) [attr1 := expr_a1, attr2 := expr_a2, ..., attrm := expr_am] print("output");```

This statement sends the message (the tuple `{expr1, expr2, ..., exprn}`) to any component satisfying the predicate `pred`. In the expressions `expr_i` it is possible to use component attributes and local attributes. They can be accessed by prepending `comp.` or `proc.` to the name of the attribute. 

The 
> **Note 1:** Make sure that the component or local attributes you are referring exist. Otherwise, the component will crash at runtime.
> **Note 2:** Instead, you may refer to receiver attributes that do not exist on other component. 
