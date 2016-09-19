---
layout: page
title: "Download"
description: "Download and Install"
group: navigation
---
{% include JB/setup %}

<!-- ------------------------- -->

To try Debussy, download a virtual machine with all software pre-installed or download the code from GitHub and install from source.

* TOC
{:toc}

-------------------------

### Install from Source

1. Check out a copy of Debussy's source code:

    `git clone http://github.com/debussy-net/debussy`   

2. Run the Debussy install script:

    `cd debussy`   
    `git checkout v0.1`   
    `util/install.sh [options]`

3. Update _debussy.cfg_ with the __*absolute*__ path to Pox.  If installing Pox with _install.sh_, this will be Debussy's parent directory.

Options for `install.sh` are:

* `-a`: install all required packages, including Mininet, Pox, and PostgreSQL
* `-m`: install only Mininet and Pox (i.e., Mininet install with options `-kmnvp`)
* `-p`: install only PostgreSQL
* `-r`: install Python libraries required by Debussy, configure PostgreSQL account and extensions


#### Configure PostgreSQL

By default, PostgreSQL uses _peer_ authentication, in which the client's username authenticates a connection to the database.  Debussy requires either _trust_ (any user can connect) or _md5_ (password-based) authentication.  If using _md5_, you will need to start Debussy with the `--password` flag to force a prompt for the password (e.g., `sudo ./debussy.py --topo single,3 --password`).

To set the authentication method, edit _/etc/postgresql/9.3/main/pg_hba.conf_ and set _postgres_ and _all_ users to _trust_ or _md5_.  Alternatively, when running `install.sh` with `-a` or `-r`, you will be prompted to make this change automatically.

_install.sh_ will create a PostgreSQL user `debussy` and database `debussy`.  To connect directly to this PostgreSQL database, use:

{% highlight bash %}
psql -Udebussy
\c debussy
{% endhighlight %}

#### Optional: Configure _ovs-ofctl_

Debussy supports multiple protocols for database triggers to interact with the OpenFlow switches.  This allows changes in the database to be propagated to the network.  Supported protocols, configured in _debussy.cfg_, are: message queues (the default protocol), RPC, and _ovs-ofctl_.  If using the _ovs-ofctl_ command, you must set passwordless sudo so database triggers can run commands using sudo:

1. Add the postgres user to sudoers

    sudo adduser postgres sudo

2. Allow passwordless sudo by editing /etc/sudoers (e.g., using `sudo visudo`) and set the user specification to:

    postgres ALL=(ALL) NOPASSWD:ALL

