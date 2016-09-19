---
layout: page
title: "Walkthrough"
description: "Debussy Walkthrough"
group: navigation
---
{% include JB/setup %}

* TOC
{:toc}


-------------------------

This walkthrough will demonstrate the Debussy CLI commands.  Throughout the tutorial, we will use the following prompt syntax:

* `$` to denote Linux commands typed into the shell prompt
* `debussy>` to denote commands typed into the Debussy CLI
* `app_name>` to denote commands typed into the sub-shell of application _app_name_


-------------------------

### Part 1: Startup Options

To display the help message describing Debussy's startup options:

    $ sudo ./debussy.py --help


#### Mininet

By default, Debussy will start Mininet and load the specified Mininet topology into a PostgreSQL database.  To specify a Mininet-style topology, use the parameter `--topo=TOPO`.  The topology parameter is required.  Debussy also accepts custom topologies in a Mininet format using the parameter `--custom=CUSTOM`.  For detailed documentation on creating custom topologies, refer to the [Mininet walkthrough](http://mininet.org/walkthrough/#custom-topologies).

For example, to start Mininet in the background with single switch and three hosts:

    $ sudo ./debussy.py --topo=single,3


#### PostgreSQL
To start Debussy without Mininet (e.g., to run database scalability tests), use the `--onlydb` flag:

    $ sudo ./debussy.py --topo=single,3 --onlydb

To specify the database name and username, use the options `--db=DB` and `--user=USER`.  To force a password prompt for the database, use `--password`.  This is equivalent to connecting to Postgres with `psql --dbname=DB --username=USER --password`.  The default database name and username can be set in _debussy.cfg_.

To reconnect to an existing database state, use the `--reconnect` flag.  _Note_: this assumes the specified topology is the same as the one already loaded in the database.


#### Verbosity

The default verbosity level is `info`.  Use the `--verbosity=LEVEL` parameter to specify one of: `error`, `warning`, `info`, `debug`.

    $ sudo ./debussy.py --topo=single,3 --verbosity=debug



-------------------------

### Part 2: Debussy Commands
To list CLI commands:

    debussy> help

To exit the CLI:

    debussy> exit

To display the current configuration (i.e., database name, username, applications path):

    debussy> stat

Debussy searches for available applications placed in the _apps_ directory.  To view discovered applications and their status:

    debussy> apps


#### SQL

Debussy provides a connection the database specified in the command-line options (or _debussy.cfg_).  To execute a SQL statement on the database, prefix the statement with the command `p`:

    debussy> p SELECT * FROM hosts;

To display real-time changes to a database table in a separate xterm window, use the `watch` command.  Specify one or table names separated by spaces:

    debussy> watch cf rm

To refresh the database by truncating all tables except the topology (_note_: this will not clear switch flow tables):

    debussy> reinit


#### Mininet

Debussy also provides a connection to the Mininet instance started in the background (unless the `--onlydb` flag is used).  Prefix the Mininet command with `m`:

    debussy> m h1 ping h2

To drop into a Mininet sub-shell, type `m` with no additional parameters:

    debussy> m
    mn> h1 ping h2


#### Performance

To report execution time for a command:

    debussy> time [command]

To report detailed execution time (if the command profiles individual operations):

    debussy> profile [command]

(_Note_: the profile command only monitors operations when running Debussy with Mininet, i.e., starting Debussy without the `--only-db` parameter.)


-------------------------

### Part 3: Orchestration

In Debussy, application are loaded using the `orch` (orchestration) command.  Orchestration translates high-level application operations onto Debussy's base tables and coordinates the resulting updates for multiple applications running simultaneously.

To load one or more applications, specify a list of applications in ascending order of priority using the `load` command:

    debussy> orch load routing fw

Here, orchestration will assign `fw` the highest priority.  Priorities are used to manage conflicts in updates proposed by different applications. Updates from higher-priority applications will override updates from lower-priority ones.  _Note:_ the `load` command requires a total ordering of applications.  When running the command a second time, any applications that are not listed in the second `load` call will be unloaded automatically.

Under orchestration, each application can propose a change.  To commit a change and check for conflicts with other running applications, use `orch run`.  To automatically commit changes, use `orch auto on`.  To disable, use `orch auto off`.  For example, to propose adding a flow:

    debussy> orch load routing
    debussy> rt addflow h1 h2
    debussy> orch run

This is the same as:

    debussy> orch load routing
    debussy> orch auto on
    debussy> rt addflow h1 h2

To report execution or profiled times for flow modification commands, use `orch auto on`:

    debussy> orch load routing
    debussy> orch auto on
    debussy> profile rt addflow h1 h2
    debussy> time rt addflow h1 h3

-------------------------

### Part 4: Applications and Sub-shells

#### Application Shells

Along with a SQL file containing application tables, views, and rules, an application can provide a Python file containing a sub-shell for monitoring and controlling the application.  For example, the sample application implements an `echo` command:

    debussy> orch load sample
    debussy> sample echo Hello World

To drop into an application's sub-shell, type the application's name (or shortcut) with no additional parameters:

    debussy> sample
    sample> echo Hello World


#### Watching Application Components

To watch real-time changes to all of an application's components (i.e., tables, views) use:

    debussy> sample watch


#### Help Commands
To print help for an application's commands, open it's shell for normal use of the help command:

    debussy> sample
    sample> help echo

Or, type `help [app name] [app command]` from the Debussy CLI.  (_Note_: an application must be loaded for its help to be accessible from the main CLI.)

    debussy> orch load sample
    debussy> help sample echo

To see a description of an application and its available commands, use:

    debussy> orch load sample
    debussy> help sample


#### Application: Routing

To add a flow between hosts, load the `routing` application and use the `addflow` command with Mininet names as the hosts' names:

    debussy> orch load routing
    debussy> orch auto on
    debussy> rt addflow h1 h2
    debussy> m h1 ping h2

To set the firewall attribute for a flow, specifying that it should be routed through a firewall, if possible, append a 1 to the `addflow` command:

    debussy> rt addflow h1 h2 1


To delete a flow, use `delflow`, specifying either the hosts' names or flow ID (i.e., fid from the table rm).

    debussy> p SELECT * FROM rm;
      fid    host1    host2
    -----  -------  -------
        1        1        2
    debussy> rt delflow 1
    

#### Application: Firewall

The firewall application implements a stateful firewall.  To add a (src,dst) pair to the whitelist, using Mininet hostnames:

    debussy> orch load fw
    debussy> fw addflow h1 h2

To add a host to the whitelist, to allow a host to initiate an outbound connection:

    debussy> fw addhost h1

To remove a host or flow from the whitelist:

    debussy> fw delflow h1 h2
    debussy> fw delhost h1



-------------------------

### Part 5: Orchestration Demo

Now, let's see orchestration in action by combining the routing and firewall applications.  First, start the Debussy CLI with a 4-switch, 4-host topology:

    $ sudo ./debussy.py --linear,4

Load the routing and firewall applications, assigning higher priority to the firewall application:

    debussy> orch load routing fw

Next, add a flow and a host to the whitelist:

    debussy> fw addhost h4
    debussy> fw addhost h2
    debussy> fw addflow h4 h3

Launch a watch window to observe insertions to the configuration table, firewall policy table, and firewall violation table:

    debussy> watch rm cf fw_violation fw_policy_acl

Add a flow that is in the whitelist of approved flows, and specify that the flow should be routed through a firewall by appending a 1 to the `addflow` command:

    debussy> rt addflow h4 h3
    debussy> orch run

Observe that the flow is installed in the configuration table (`cf`) and the the hosts can ping each other:

    debussy> m h4 ping h3

Now, try adding a flow that is not in the approved flow or host whitelist, by proposing a flow with an external host as the source:

    debussy> rt addflow h1 h2

Observe a new row is inserted into the firewall violation table.  Next, commit the change:

    debussy> orch run

Observe that the firewall application repairs the violation by removing the proposed flow from the reachability table `rm`.



-------------------------

### Part 6: Network State Changes: Rerouting

Debussy can react to changes in network state (e.g., link or switch failures).  This part of the walkthrough will demonstrate the routing application's ability to reroute installed flows automatically when links fail.


First, start Debussy with the custom diamond topology:

    $ sudo ./debussy.py --topo=diamond --custom=~/topo/diamond.py


This will start Debussy and Mininet with the following topology (hostnames are listed above node IDs):

{:refdef: style="text-align: center;"}
![diamond topology diagram]({{site.url}}/images/diamond_topo.png)
{: refdef}
 
Load the routing application under orchestration and install a flow between the two hosts in the topology:

    > orch load routing
    > rt addflow h1 h2
    > orch run

Examine the installed path by querying the configuration table `cf`:

    > p select * from cf;
        fid      pid     sid      nid
      -----  -------  -------  -------
         1        5        1       3
         1        1        3       4
         1        3        4       6

Notice that the install path is: 5 -> 1 -> 3 -> 4 -> 6, or using hostnames: h1 -> s1 -> s3 -> s4 -> h2.

Now, let's take the links s1-s3 and s3-s4 by taking switch s3 offline:

    > m switch s3 stop

Examine the configuration table again:

    > p select * from cf;
        fid      pid     sid      nid
      -----  -------  -------  -------
         1        5        1       2
         1        1        2       4
         1        2        4       6

Notice that the path is rerouted to now use switch s2.  The new path is now: 5 -> 1 -> 2 -> 4 -> 6, or: h1 -> s1 -> s2 -> s4 -> h2.
