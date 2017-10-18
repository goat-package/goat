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
