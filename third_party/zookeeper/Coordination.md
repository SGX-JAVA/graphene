# ZooKeeper: Distributed Process Coordination
## Administering ZooKeeper
### Running ZooKeeper
#### Monitoring with JMX
ZooKeeper also uses a standard Java management protocol called JMX (Java Management Extensions) to provide more powerful monitoring and management capabilities. There are many books about how to set up and use JMX and many tools for managing servers with JMX; in this section we will use a simple management tool called *`jconsole`* to explore the ZooKeeper management functionality that is available via JMX.

The *`jconsole`* utility is distributed with Java. In practice, a JMX tool such as *`jconsole`* is used to monitor remote ZooKeeper servers, but for this part of the exercise we will run it on the same machine as our ZooKeeper servers.

First, let's start up the second ZooKeeper server (the one with the ID of 2). Then we'll start *`jconsole`* by simply running the *`jconsole`* command from the command line. You should see a window similar to Figure 10-7 when *`jconsole`* starts.

Notice the process that has "zookeeper" in the name. This is the local process that *`jconsole`* has discovered that it can connect to.

Now let's connect to the ZooKeeper process by double-clicking that process in the list. We will be asked about connecting insecurely because we do not have SSL set up. Clicking the insecure connection button should bring up the screen shown in Figure 10-8.

As we can see from this screen, we can get various interesting statistics about the ZooKeeper server process with this tool.
