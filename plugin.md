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
