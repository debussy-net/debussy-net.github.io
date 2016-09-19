---
layout: page
title: "Manual"
description: "Developer Guide"
group: navigation
---
{% include JB/setup %}

* TOC
{:toc}

-------------------------

### Debussy Overview
Debussy allows multiple applications to execute simultaneously and collectively drive network control.  This is accomplished through _orchestration_.  A Debussy user sets priorities on the orchestrated applications using the ordering passed to the `orch load` command.  (For more details, see the orchestration demo in the [Walkthrough]({{site.url}}walkthrough#part-4-orchestration).)

Applications can _propose_ updates (through an insertion, deletion, or update to one of its views), which are then checked against the policies of all other orchestrated applications.  If the proposed update violates the constraints of another application, a higher priority application can overwrite the update.

#### Debussy Base Tables

Debussy uses a flat representation of the network and exposes the topology and forwarding tables as SQL tables.  Tables that may be of interest to applications are:

{% highlight sql%}
/* Topology table - pairs of connected nodes
 * sid: switch id
 * nid: next-hop id
 * ishost: if nid is a host
 * isactive: if the sid-nid link is online
 * bw: link bandwidth
 */
tp(sid integer, nid integer, ishost integer, isactive integer, bw integer)

/* Configuration table - per-switch flow configuration
 * fid: flow id
 * pid: id of previous-hop node
 * sid: switch id
 * nid: id of next-hop node
 */
cf (fid integer, pid integer, sid integer, nid integer)

/* Reachability matrix - end-to-end reachability
 * fid: flow id
 * src: the IP address of the source node
 * dst: the IP address of the destination node
 * vol: volume allocated for the flow
 * FW: if flow should pass through a firewall
 * LB: if flow should be load balanced
 */
tm (fid integer, src integer, dst integer, vol integer, FW integer, LB integer)
{% endhighlight %}

Additional node tables include:

{% highlight sql%}
/* Host table
 * hid: host id
 * ip: host's IP address
 * mac: host's MAC address
 * name: hostname (in Mininet)
 */
hosts (hid integer, ip varchar, mac varchar, name varchar)

/* Switch table
 * sid: switch id (primary key, NOT datapath id)
 * dpid: datapath id
 * ip: switch's IP address
 * mac: switch's MAC address
 * name: switch's name (in Mininet) 
 */
switches (sid integer, dpid varchar, ip varchar, mac varchar, name varchar)

/* Node view - all nodes and switches in the network
 * id: the node's id from its respective table (hosts.hid or switches.sid)
 * name: the node's name
 */
nodes (id integer, name varchar)
{% endhighlight %}
    

### Developing Applications
Applications in Debussy can consist of two components: an implementation in SQL and a sub-shell implemented in Python.  Application sub-shells provide commands to monitor and control the application from the Debussy CLI.  Name the SQL file _[appname].sql_ and the Python file _[appname].py_, and place both files in the _apps/_ directory.  The CLI will search for SQL and Python files in this directory.

For an overview of the CLI and application superclasses, see the [API](api/annotated.html).


#### SQL Component
An application's SQL component can create tables, views on Debussy base tables, 
or add triggers on Debussy base tables.  To interact with orchestration, an application must define its constraints and a protocol for reconciling conflicts.  To add constraints, create a violation table in the form `appname_violation`.  To create a repair protocol, add a rule in the form `appname_repair`.

For example, suppose we want to implement a bandwidth monitor that limits flows to a particular rate (see _apps/merlin.sql_ for the full implementation).  We can create a table to store the rate for each flow:

{% highlight sql %}
CREATE TABLE bw_policy (
    fid      integer,
    rate     integer,
    PRIMARY KEY(fid)
);
{% endhighlight %}

And then a violation table that finds flows that violate the constraint `fid < rate`:

{% highlight sql %}
CREATE VIEW bw_violation AS (
    SELECT tm.fid, rate AS req, vol AS asgn
	FROM tm, bw_policy
	WHERE tm.fid = bw_policy.fid AND rate > vol
);
{% endhighlight %}

If the view contains more than zero rows, a conflict has occurred.  To repair a conflict, we can reset the flow rate:

{% highlight sql %}
CREATE RULE bw_repair AS
    ON DELETE TO bw_violation
    DO INSTEAD
        UPDATE tm SET vol=OLD.req WHERE fid=OLD.fid;

{% endhighlight %}

#### Python Sub-Shell
To interact with an application, the Debussy CLI can load a shell with commands for monitoring and controlling the application's behavior.  An application _sub-shell_ can be launched from the Debussy CLI by typing the application's name or shortcut after it has been loaded.  To create a sub-shell, create a Python file with a class extending [AppConsole](api/classdebussy_1_1app_1_1AppConsole.html) and add the following variables:

* `shortcut`: defines a shortcut for the application from the Debussy CLI
* `description`: a short description of the application
* `console`: defines the class inheriting [AppConsole](api/classdebussy_1_1app_1_1AppConsole.html)

For example:

{% highlight python %}
from debussy.app import AppConsole

class MyConsole(AppConsole):
    do_echo(self, line):
        "Echo input"
        print "MyConsole says:", line

shortcut = "my"
description = "my demo console"
console = MyConsole

{% endhighlight %}

The [AppConsole](api/classdebussy_1_1app_1_1AppConsole.html) class contains the properties:

* `self.db`: a reference to [debussy.db.DebussyDb](api/classdebussy_1_1db_1_1DebussyDb.html)
* `self.env`: a reference to [debussy.env.Environment](api/classdebussy_1_1env_1_1Environment.html), the CLI's executing environment
* `self.components`: a list of [debussy.app.AppComponent](api/classdebussy_1_1app_1_1AppComponent.html), the application's SQL components (i.e., tables, views, rules)

For more information about the application superclasses and their properties, see the [API](api/annotated.html).



### Measuring Performance
The Debussy CLI provides the command `time` to measure the execution time of a command, similar to the Linux time command: `time [command] [command args]`.

For more granular inspection of performance, the Debussy CLI provides a `profile` command.  This command reports the execution time of manually-defined blocks of code.  When extending the Debussy runtime or developing applications, a new block of code can be defined using the class [profiling.PerfCounter](api/classdebussy_1_1profiling_1_1PerfCounter.html).  The constructor accepts a string to identify the block of code.  The functions [start()](api/classdebussy_1_1profiling_1_1PerfCounter.html#a5668f7bc8650c3e6e308f888f7d6dbec) and [stop()](api/classdebussy_1_1profiling_1_1PerfCounter.html#ad268c2cff4282db907b7c8570afe6a98) define the start and end of the code to be profile:

{% highlight python %}
from debussy.profiling import PerfCounter

def my_profiled_function():
    pc = PerfCounter("name of code block")
    pc.start()

    # will measure the execution time of the code here

    pc.stop()

{% endhighlight %}
