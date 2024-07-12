.. _howto.getting.started:

Getting Started
***************

This page will help you take your first steps with OpenSVC services setup.

It will guide you through the sequence of tasks to achieve a simple but working dual-node failover cluster.

Prerequisites
=============

The demonstration environment is composed of:

* A pair of Ubuntu 24.04 servers named ``demo1`` and ``demo2``, respectively acting as first and second cluster node
* Both nodes are connected to the same network segment on 5.196.34.128/27
* root access is mandatory

OpenSVC Installation
====================

We will follow the steps described in `Nodeware installation <agent.install.html>`_

Install the OpenSVC Agent on both cluster nodes.

**On both nodes**::

        $ wget -O /tmp/opensvc.latest.deb https://repo.opensvc.com/deb/current

        $ sudo apt install -y /tmp/opensvc.latest.deb

        $ sudo systemctl start opensvc-agent.service

        $ sudo systemctl is-active opensvc-agent.service
        active

We can also check for proper daemon behaviour:

.. raw:: html

    <pre class=output>
    $ sudo om mon

    Threads                           <span style="font-weight: bold">demo1</span>
     <span style="font-weight: bold">daemon</span>    <span style="color: #00aa00">running</span>               |
     <span style="font-weight: bold">listener</span>  <span style="color: #00aa00">running</span> :1214
     <span style="font-weight: bold">monitor</span>   <span style="color: #00aa00">running</span>
     <span style="font-weight: bold">scheduler</span> <span style="color: #00aa00">running</span>

    Nodes                             <span style="font-weight: bold">demo1</span>
     <span style="font-weight: bold"> score</span>                          | 70
     <span style="font-weight: bold">  load 15m</span>                      | 0.0
     <span style="font-weight: bold">  mem</span>                           | 10/98%:3.82g
     <span style="font-weight: bold">  swap</span>                          | -
     <span style="font-weight: bold"> state</span>                          |

    */svc/*                          <span style="font-weight: bold">  demo1</span>
    </pre>

The ``Threads`` section is explained here :ref:`agent.daemon`

The OpenSVC agent is now operational.

SSH Keys Setup
==============

Cluster members needs ssh mutual authentication to exchange some OpenSVC configuration files. Each node must trust its peer through key-based authentication to allow these communications.

* demo1 will be able to connect to demo2 as root.
* demo2 will be able to connect to demo1 as root.

.. note::

        It is also possible for the agent to login on a peer cluster node using an unprivileged user, using the user node.conf parameter. In this case, the remote user needs sudo priviles to run the following commands as root: ``nodemgr``, ``svcmgr`` and ``rsync``.

**On demo1**::

	demo1:/ # om node update ssh keys --node='*'


Set Host Environment
====================

As we are in a lab environment, we do not need to specify the host environment : "TST" is the default value, and is adequate.

For other purposes than testing, we would have defined on both nodes the relevant mode with the method described here :ref:`set-node-environment`

Cluster Build
=============

As our first setup consist in a dual node cluster, we have to follow the steps described here :ref:`agent.configure.cluster`

**On node1**

.. raw:: html

    <pre class=output>
    $ om cluster set --kw hb#1.type=unicast

    $ sudo om mon

    Threads                           <span style="font-weight: bold">demo1</span>
     <span style="font-weight: bold">daemon</span>    <span style="color: #00aa00">running</span>               |
     <span style="font-weight: bold">listener</span>  <span style="color: #00aa00">running</span> :1214
     <span style="font-weight: bold">monitor</span>   <span style="color: #00aa00">running</span>
     <span style="font-weight: bold">scheduler</span> <span style="color: #00aa00">running</span>

    Nodes                             <span style="font-weight: bold">demo1</span>
     <span style="font-weight: bold"> score</span>                          | 70
     <span style="font-weight: bold">  load 15m</span>                      | 0.0
     <span style="font-weight: bold">  mem</span>                           | 10/98%:3.82g
     <span style="font-weight: bold">  swap</span>                          | -
     <span style="font-weight: bold"> state</span>                          |

    */svc/*                          <span style="font-weight: bold">  demo1</span>
    </pre>

A pair of "Threads" section lines will appear later, when at least a second node joins this cluster seed : one for the sender and the other for the receiver.

Service Creation
================

The OpenSVC service can be created using one of the following two methods:

* provisioning
* manual : build config file from templates (located in ``<OSVCDOC>``)

We will describe the second, manual option, for a better understanding of what happens. 

Step 1 : Service creation
+++++++++++++++++++++++++

A simple command is needed to create an empty service named ``svc1``::

    $ om svc1 create

The expected file name is ``svc1.conf`` located in ``<OSVCETC>``
At this time, the configuration file should be empty, except its unique id. You have to edit it in order to define your service.

We can list our services with the following command::

    $ om '*/svc/*' ls
    svc1

We are going to define a service running on the primary node ``demo1``, failing-over to node ``demo2``, using one IP address named ``demosvc.opensvc.com`` (name to ip resolution is done by the OpenSVC agent), one LVM volume group ``vgsvc1`` and two filesystems hosted in logical volumes ``/dev/vgsvc1/app`` and ``/dev/vgsvc1/data``. We expect those resources to already exists on the system. #ajouter ote disque partagé

**On demo1 node**::

        $ om svc1 edit config

        [DEFAULT]
        app = MyApp
        nodes = demo1 demo2
        id = d74cc963-8f18-4c73-9691-9a833dba2a22

        [ip#0]
        ipname = demosvc.opensvc.com
        ipdev = br-prd

        [disk#0]
        type = vg
        name = vgsc1

        [disk#1]
        type = lv
        name = app
        vg = {disk#0.name}

        [disk#2]
        type = lv
        name = data
        vg = {disk#0.name}

        [fs#app]
        type = ext4
        dev = {disk#1.exposed_devs[0]}
        mnt = /{name}/{disk#1.name}

        [fs#data]
        type = ext4
        dev = {disk#1.exposed_devs[0]}
        mnt = /{name}/{disk#2.name}

The DEFAULT section in the service file describes the service itself: human readable name, nodes where the service is expected to run on...

Every other section defines a resource managed by the service.

Step 4 : Service configuration check
++++++++++++++++++++++++++++++++++++

As a final check, we can list all entries that match our ``svc1`` service ::

        node1:/etc/opensvc # ls -lart | grep svc1
        -rw-r--r--.  1 root root  331 Jul  12 12:13 svc1.conf

You should be able to see the service configuration file (svc1.conf)

At this point, we have configured a single service with no application launcher on node demo1.

Service Testing
===============

Query service status
++++++++++++++++++++

Our first service is now ready to use. We can query its status.

**On demo1**

.. raw:: html

    <style>
        .down {
            color: red;
        }
        .up {
            color: green;
        }
        .warn {
            color: brown;
        }
        .frozen {
            color: navy;
        }
        .not-provisioned {
            color: red;
        }
        .idle {
            color: gray;
        }
    </style>

    <pre>
    svc1                           <span class="down">down</span>
        `- instances
           `- demo1                      <span class="warn">warn</span>       <span class="warn">warn</span>, <span class="frozen">frozen</span>, <span class="not-provisioned">not provisioned</span>, <span class="idle">idle</span>
              |- ip#0           .....P.. <span class="down">down</span>       5.196.34.135/255.255.255.224 br-prd demosvc.opensvc.co demo1.opensvc.com
              |- disk#0         .....P.. <span class="down">down</span>       vg vgsvc1
              |- disk#1         .....P.. <span class="down">down</span>       lv vgsvc1/app
              |- disk#2         .....P.. <span class="down">down</span>       lv vgsvc1/data
              |- fs#app         .....P.. <span class="down">down</span>       xfs /dev/vgsvc1/app@/svc1/app
              |- fs#data        .....P.. <span class="down">down</span>       xfs /dev/vgsvc1/data@/svc1/data
              `- sync#i0        ...O./.. n/a        rsync svc config to nodes
                                                                                          info: paused, service not up
    </pre>

.. note::

        by default the agent believes that all resources are not created that's why there is P letter on each resource line, which means unprovisioned. As those resources have already been created by the system administrator we have to inform the opensSVC agent about it using the command ``om svc1 set provisioned``.

**On demo1**

.. raw:: html

    <style>
        .down {
            color: red;
        }
        .up {
            color: green;
        }
        .warn {
            color: brown;
        }
        .frozen {
            color: navy;
        }
        .not-provisioned {
            color: red;
        }
        .idle {
            color: gray;
        }
    </style>

    <pre>
    $ om svc1 set provisioned

    $ om svc1 print status

    svc1                           <span class="down">down</span>
        `- instances
           `- demo1                      <span class="warn">warn</span>       <span class="warn">warn</span>, <span class="frozen">frozen</span>, <span class="idle">idle</span>
              |- ip#0           ........ <span class="down">down</span>       5.196.34.135/255.255.255.224 br-prd demosvc.opensvc.co demo1.opensvc.com
              |- disk#0         ........ <span class="down">down</span>       vg vgsvc1
              |- disk#1         ........ <span class="down">down</span>       lv vgsvc1/app
              |- disk#2         ........ <span class="down">down</span>       lv vgsvc1/data
              |- fs#app         ........ <span class="down">down</span>       xfs /dev/vgsvc1/app@/svc1/app
              |- fs#data        ........ <span class="down">down</span>       xfs /dev/vgsvc1/data@/svc1/data
              `- sync#i0        ...O./.. n/a        rsync svc config to nodes
                                                                                          info: paused, service not up
    </pre>

This command collects and displays status for each service ressource :

- overall status is ``warn`` due to the fact that all ressources are not in ``up`` status
- all other ressources are ``down`` or non available ``n/a``

Start service
+++++++++++++

The use of OpenSVC for your services management saves a lot of time and effort.
Once the service is described on a node, you just need one command to start the overall application.

Let's start the service on the local node::

        opensvc@demo1:~$ om svc1 start --local
        @ n:demo1 o:svc1 r:ip#0 sc:n
          checking 5.196.34.135 availability (5s)
          /sbin/ip addr add 5.196.34.135/27 dev br-prd
          send gratuitous arp to announce 5.196.34.135 is at br-prd
        @ n:demo1 o:svc1 r:disk#0 sc:n
          vgchange --addtag @demo1 vgsvc1
          | Volume group "vgsvc1" successfully changed
          vgchange -a y vgsvc1
          | 2 logical volume(s) in volume group "vgsvc1" now active
        @ n:demo1 o:svc1 r:disk#1 sc:n
          lv vgsvc1/app is already up
        @ n:demo1 o:svc1 r:disk#2 sc:n
          lv vgsvc1/data is already up
        @ n:demo1 o:svc1 r:fs#app sc:n
          e2fsck -p /dev/vgsvc1/app
          | /dev/vgsvc1/app: clean, 15/65536 files, 15363/262144 blocks
          mount -t ext4 /dev/vgsvc1/app /svc1/app
        @ n:demo1 o:svc1 r:fs#data sc:n
          e2fsck -p /dev/vgsvc1/data
          | /dev/vgsvc1/data: clean, 20/131072 files, 26167/524288 blocks
          mount -t ext4 /dev/vgsvc1/data /svc1/data

The startup sequence reads as:

- check if service IP address is not already used somewhere
- bring up service ip address 
- volume group activation (if not already in the correct state)
- fsck + mount of each filesystem


Manual filesystem mount check::

        demo1:~$ mount | grep svc1
        /dev/mapper/vgsvc1-app on /svc1/app type ext4 (rw,relatime,stripe=2048)
        /dev/mapper/vgsvc1-data on /svc1/data type ext4 (rw,relatime,stripe=2048)


Manual ip address plumbing check on br-prd (svc1.opensvc.com is 5.196.34.135)::

        demo1:~$ ip addr list br-prd
        3: br-prd: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
            link/ether c2:98:d3:28:7e:7b brd ff:ff:ff:ff:ff:ff
            inet 5.196.34.153/27 brd 5.196.34.159 scope global br-prd
               valid_lft forever preferred_lft forever
            inet 5.196.34.135/27 scope global secondary br-prd
               valid_lft forever preferred_lft forever
            inet6 fe80::c098:d3ff:fe28:7e7b/64 scope link
               valid_lft forever preferred_lft forever


We can confirm everything is OK with the service's ``print status`` command

.. raw:: html

    <style>
        .up {
            color: green;
        }
        .frozen {
            color: navy;
        }
        .started {
            color: gray;
        }
        .idle {
            color: gray;
        }
    </style>

    <pre>
    $ om svc1 print status

    svc1                           <span class="up">up</span>
        `- instances
           `- demo1                      <span class="up">up</span>       <span class="frozen">frozen</span>, <span class="idle">idle</span>, <span class="started">started</span>
              |- ip#0           ........ <span class="up">up</span>       5.196.34.135/255.255.255.224 br-prd demosvc.opensvc.co demo1.opensvc.com
              |- disk#0         ........ <span class="up">up</span>       vg vgsvc1
              |- disk#1         ........ <span class="up">up</span>       lv vgsvc1/app
              |- disk#2         ........ <span class="up">up</span>       lv vgsvc1/data
              |- fs#app         ........ <span class="up">up</span>       xfs /dev/vgsvc1/app@/svc1/app
              |- fs#data        ........ <span class="up">up</span>       xfs /dev/vgsvc1/data@/svc1/data
              `- sync#i0        ...O./.. n/a        rsync svc config to nodes
                                                                                          info: paused, service not up
    </pre>

At this point, we have a running service, configured to run on demo1 node.

Application Integration
=======================

We have gone through the setup of a single service, but it does not start applications yet. Let's add an application to our service now.

We will use a very simple example : a tiny webserver with a single index.html file to serve

Application Binary
++++++++++++++++++

In the service directory structure, we put a standalone binary of the Mongoose web server (https://code.google.com/p/mongoose/) ::

        demo1:$ cd /svc1/app

        demo1:svc1/app$ sudo wget -O /svc1/app/webserver https://github.com/mufeedvh/binserve/releases/download/v0.2.0/binserve-v0.2.0-x86_64-unknown-linux-gnu.tar.gz
        --2024-07-12 09:34:40--  https://github.com/mufeedvh/binserve/releases/download/v0.2.0/binserve-v0.2.0-x86_64-unknown-linux-gnu.tar.gz
        Resolving github.com (github.com)... 140.82.121.4
        Connecting to github.com (github.com)|140.82.121.4|:443... connected.
        HTTP request sent, awaiting response... 302 Found
        Location: https://objects.githubusercontent.com/github-production-release-asset-2e65be/300180221/4b52d8b8-d2c3-4f02-aad2-ac86b825da52?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=releaseassetproduction%2F20240712%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20240712T073440Z&X-Amz-Expires=300&X-Amz-Signature=bde3ca8606945cad7e30db87443d80bc58d0556f8500e29155e6db11126cbc2c&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=300180221&response-content-disposition=attachment%3B%20filename%3Dbinserve-v0.2.0-x86_64-unknown-linux-gnu.tar.gz&response-content-type=application%2Foctet-stream [following]
        --2024-07-12 09:34:40--  https://objects.githubusercontent.com/github-production-release-asset-2e65be/300180221/4b52d8b8-d2c3-4f02-aad2-ac86b825da52?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=releaseassetproduction%2F20240712%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20240712T073440Z&X-Amz-Expires=300&X-Amz-Signature=bde3ca8606945cad7e30db87443d80bc58d0556f8500e29155e6db11126cbc2c&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=300180221&response-content-disposition=attachment%3B%20filename%3Dbinserve-v0.2.0-x86_64-unknown-linux-gnu.tar.gz&response-content-type=application%2Foctet-stream
        Resolving objects.githubusercontent.com (objects.githubusercontent.com)... 185.199.109.133, 185.199.110.133, 185.199.108.133, ...
        Connecting to objects.githubusercontent.com (objects.githubusercontent.com)|185.199.109.133|:443... connected.
        HTTP request sent, awaiting response... 200 OK
        Length: 3639943 (3.5M) [application/octet-stream]
        Saving to: ‘/svc1/app/webserver’

        /svc1/app/webserver        100%[=====================================>]   3.47M  --.-KB/s    in 0.1s

        2024-07-12 09:34:41 (33.2 MB/s) - ‘/svc1/app/webserver’ saved [3639943/3639943]


        demo1:/svc1/app # ls -l /svc1/app/binserve
        -rw-r--r-- 1 root root 2527016 June 13  2022 /svc1/app/binserve

Then, you can extract the content of binserve archive::

        demo1:/svc1/app # tar -xvf binserve

Now we can launch the server and bind our IP address with ``--host`` option::

        demo1:/svc1/data # sudo ./binserve --host 5.196.34.135:8080
         _   _
        | |_|_|___ ___ ___ ___ _ _ ___
        | . | |   |_ -| -_|  _| | | -_|
        |___|_|_|_|___|___|_|  \_/|___| 0.2.0

        [INFO] Build finished in 291 μs ⚡
        [SUCCESS] Your server is up and running at 5.196.34.135:8080 🚀


Applications launcher script
++++++++++++++++++++++++++++

We have to create a management script for our web application. At minimum, this script must support the ``start`` argument.

As a best practice, the script should also support the additional arguments:

- stop
- status
- info

Of course, we will store our script named ``binserve_launcher`` in the directory previsouly created for this purpose::

        demo1:/ # cd /svc1/app

        demo1:/svc1/app# cat weblauncher
        #!/bin/bash
        set -x
        SVCROOT=/svc1
        APPROOT=${SVCROOT}/app
        DAEMON=${APPROOT}/binserve
        DAEMON_BASE=$(basename $DAEMON)
        DAEMONOPTS="--host 5.196.34.135:8080"

        function status {
                pgrep -f "$DAEMON $DAEMONOPTS" >/dev/null 2>&1
        }

        case $1 in
        start)
                status && {
        	        echo "binserve is already started"
    	            exit 0
                }
                echo "Starting Binserve..."
                nohup $DAEMON $DAEMONOPTS > /dev/null 2>&1 &
                echo "Binserve started !"
	            ;;
        stop)
	            killall $DAEMON_BASE
	            ;;
        info)
	            echo "Name: binserve"
	            ;;
        status)
	            status
	            exit $?
	            ;;
        *)
	            echo "unsupported action: $1" >&2
	            exit 1
	            ;;

        esac
        exit 0

Make sure the script is working fine outside of the OpenSVC context (and don't forget to add execute rights to /svc1/app/webserver) ::

        demo1:/svc1/app/ # ./binserve_launcher status
        demo1:/svc1/app/ # echo $?
        1
        demo1:/svc1/app/ # ./binserve_launcher start
        demo1:/svc1/app/ # ./binserve_launcher status
        demo1:/svc1/app/ # echo $?
        0
        demo1:/svc1/app/ # ./binserve_launcher stop
        demo1:/svc1/app/ # ./binserve_launcher status
        demo1:/svc1/app/ # echo $?
        1

Now we need to instruct OpenSVC to handle this script for service application management ::

        # om svc1 edit config

        (...)
        [app#web]
        type = forking
        start = {fs#app.mnt}/binserve_launcher start
        stop = {fs#app.mnt}/binserve_launcher stop
        check = {fs#app.mnt}/binserve_launcher status
        info = {fs#app.mnt}/binserve_launcher info

This configuration tells OpenSVC to call the ``binserver_launcher`` script with :

- ``start`` argument when OpenSVC service starts
- ``stop`` argument when OpenSVC service stops
- ``status`` argument when OpenSVC service needs status on application

Now we can give a try to our launcher script, using OpenSVC commands::

        demo1:/svc1/app/ # om svc1 start --rid app#web
        @ n:demo1 o:svc1 r:app#web sc:n
          exec /svc1/app/binserve_launcher start as user root
          | Starting Binserve...
          | Binserve started !
          start done in 0:00:00.213427 - ret 0

We can see that OpenSVC is successfully launching our server. In addition we can see here that OpenSVC can start resources individually.

Querying the service status, the ``app`` ressource is now reporting ``up``

**On node1**

.. raw:: html

    <style>
        .up {
            color: green;
        }
        .frozen {
            color: navy;
        }
        .started {
            color: gray;
        }
        .idle {
            color: gray;
        }
    </style>

    <pre>
    $ om svc1 print status

    svc1                           <span class="up">up</span>
        `- instances
           `- demo1                      <span class="up">up</span>       <span class="frozen">frozen</span>, <span class="idle">idle</span>, <span class="started">started</span>
              |- ip#0           ........ <span class="up">up</span>       5.196.34.135/255.255.255.224 br-prd demosvc.opensvc.co demo1.opensvc.com
              |- disk#0         ........ <span class="up">up</span>       vg vgsvc1
              |- disk#1         ........ <span class="up">up</span>       lv vgsvc1/app
              |- disk#2         ........ <span class="up">up</span>       lv vgsvc1/data
              |- fs#app         ........ <span class="up">up</span>       xfs /dev/vgsvc1/app@/svc1/app
              |- fs#data        ........ <span class="up">up</span>       xfs /dev/vgsvc1/data@/svc1/data
              |- app#web        ...../.. <span class="up">up</span>       forking: weblauncher
              `- sync#i0        ...O./.. n/a        rsync svc config to nodes
                                                                                          info: paused, service not up
    </pre>

Let's check if that is really the case::

        node1:/ # ps auxww | grep binserver
        root      473398  0.0  0.2 554288  8960 ?        Sl   14:39   0:01 /svc1/app/binserve --host 5.196.34.135:8080

        node1:~ # curl demosvc.opensvc.com:8080
        <!--
            +----------------------------+
            | Auto-generated by binserve |
            +----------------------------+
        -->
        <!DOCTYPE html>
        <html lang="en">
        <head>
            <meta charset="UTF-8">
            <meta http-equiv="X-UA-Compatible" content="IE=edge">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>Binserve v0.2.0</title>
            <link rel="stylesheet" href="assets/css/styles.css">
        </head>
        <body>
            <div class="contents">
                <img class="logo" src="assets/images/binserve.webp" alt="binserve logo">
                <h1>Hello, Universe! 🚀</h1>
                <p>Your new web server is up and ready!</p>
                <br>
                <details>
                    <summary><strong>Get Started</strong></summary>
                    <p>Looks like everything's working perfectly! Hurray! 🎉</p>
                    <p>By default, binserve generates these static content as a boileplate for you to get started quickly.</p>
                    <p>The directory structure, configuration file, and these sample files utilize all the features.</p>
                    <p>This can help you learn the usage just by skimming through the generated static files.</p>
                    <h3>✨ Enjoy the breeze experience! ✨</h3>
                    <a href="/usage"><button>Usage</button></a>
                </details>
                <br>
                <p><em><strong>binserve v0.2.0</strong></em></p>
            </div>
        </body>

Now we can stop our service::

        node1:/ # om svc1 stop --local app#web
        @ n:demo1 o:svc1 r:app#web sc:n
          exec /svc1/app/binserve_launcher stop as user root
          stop done in 0:00:00.403740 - ret 0
        @ n:demo1 o:svc1 r:fs#data sc:n
          umount /svc1/data
        @ n:demo1 o:svc1 r:fs#app sc:n
          umount /svc1/app
        @ n:demo1 o:svc1 r:disk#2 sc:n
          lvchange -a n vgsvc1/data
        @ n:demo1 o:svc1 r:disk#1 sc:n
          lvchange -a n vgsvc1/app
        @ n:demo1 o:svc1 r:disk#0 sc:n
          vgchange --deltag @demo1 vgsvc1
          | Volume group "vgsvc1" successfully changed
        @ n:demo1 o:svc1 r:ip#0 sc:n
          /sbin/ip addr del 5.196.34.135/27 dev br-prd
          checking 5.196.34.135 availability (1s)

Once again, a single command:

- brings down the application
- unmounts filesystems
- deactivates the volume group
- disables the service ip address

The overall status is now reported as being down

**On demo1**

.. raw:: html

    <style>
        .down {
            color: red;
        }
        .up {
            color: green;
        }
        .warn {
            color: brown;
        }
        .frozen {
            color: navy;
        }
        .not-provisioned {
            color: red;
        }
        .idle {
            color: gray;
        }
    </style>

    <pre>
    $ om svc1 print status

    svc1                           <span class="down">down</span>
        `- instances
           `- demo1                      <span class="warn">warn</span>       <span class="warn">warn</span>, <span class="frozen">frozen</span>, <span class="idle">idle</span>
              |- ip#0           ........ <span class="down">down</span>       5.196.34.135/255.255.255.224 br-prd demosvc.opensvc.co demo1.opensvc.com
              |- disk#0         ........ <span class="down">down</span>       vg vgsvc1
              |- disk#1         ........ <span class="down">down</span>       lv vgsvc1/app
              |- disk#2         ........ <span class="down">down</span>       lv vgsvc1/data
              |- fs#app         ........ <span class="down">down</span>       xfs /dev/vgsvc1/app@/svc1/app
              |- fs#data        ........ <span class="down">down</span>       xfs /dev/vgsvc1/data@/svc1/data
              |- app#web        ...../.. <span class="down">down</span>       forking: weblauncher
              `- sync#i0        ...O./.. n/a        rsync svc config to nodes
                                                                                          info: paused, service not up
    </pre>

Let's restart the service to continue this tutorial::

        node1:/ # om svc1 start --local

At this point, we have a running service on node node1, with a webserver application embedded.

Service Failover
================

Our service is running fine, but what happens if the ``demo1`` node fails ? Our ``svc1`` service will also fail.
That's why we want to extend the service configuration to declare ``demo2`` as a failover node for this service.
After this change, the service configuration needs replication to the ``demo2`` node. 

First, we are going to add ``demo2`` in the same cluster than ``demo1``

**On demo1**

**On demo2**

.. raw:: html

    <style>
        .running {
            color: green;
        }
        .O-caret {
            color: green;
        }
        .X-caret {
            color: gray;
        }
    </style>

    <pre>
        $ om cluster get --kw cluster.secret
        2bfca0d1393611efa4dc00163e000000

        $ om daemon join --secret 2bfca0d1393611efa4dc00163e000000 --node node1
        @ n:demo2
          freeze local node
          add heartbeat hb#1
          join node demo1
          thaw local node

        $ om mon

        Threads                            demo1 demo2
         hb#1.rx   <span class="running">running</span> [::]:10000    | <span class="O">O</span>     /
         hb#1.tx   <span class="running">running</span>               | <span class="O">O</span>     /
         listener  <span class="running">running</span>      :1214
         monitor   <span class="running">running</span>
         scheduler <span class="running">running</span>

        Nodes                              demo1 demo2
         15m                             | 0.1   0.1
         state                           |       *

        Services                           demo1 demo2
         svc1      <span class="running">up</span>              1/1   | <span class="caret">O^</span>    <span class="X-caret">X^</span>
    </pre>

OpenSVC will synchronize configuration files for your service since this one should be able to run on node1 or node2.
In order to force it now, run on ``demo1`` ::

        # om svc1 sync nodes --rid sync#i0

The configuration replication will be possible if the following conditions are met:

- the new node is declared in the service configuration file ``<OSVCETC>/svc1.conf`` (parameter "nodes" in the .conf file)
- the node sending config files (node1) is trusted on the new node (node2) (as described in a previous chapter of this tutorial)
- the node sending config files (node1) must be running the service (the service availability status, apps excluded, is up).
- the previous synchronisation is older than the configured minimum delay, or the --force option is set to bypass the delay check.

**On node1**

We can now try to switch the svc1 service from ``demo1`` to ``demo2``

**On demo2**

.. raw:: html

    <style>
        .down {
            color: red;
        }
        .placement {
            color: red;
        }
        .up {
            color: green;
        }
        .warn {
            color: brown;
        }
        .frozen {
            color: navy;
        }
        .not-provisioned {
            color: red;
        }
        .idle {
            color: gray;
        }
    </style>

    <pre>
        $ om svc1 switch
        $ om mon
        $ om svc1 print status -r

        svc1                             <span class="up">up</span>         <span class="placement">non-optimal placement</span>
        `- instances
           |- demo1                      <span class="down">down</span>       <span class="idle">idle</span>
           `- demo2                      <span class="up">up</span>         <span class="idle">idle</span>, started
              |- ip#0           ........ <span class="up">up</span>         5.196.34.135/255.255.255.224 br-prd demosvc.opensvc.com
              |- disk#0         ........ <span class="up">up</span>         vg vgsvc1
              |- disk#1         ........ <span class="up">up</span>         lv vgsvc1/app
              |- disk#2         ........ <span class="up">up</span>         lv vgsvc1/data
              |- fs#app         ........ <span class="up">up</span>         ext4 /dev/vgsvc1/app@/svc1/app
              |- fs#data        ........ <span class="up">up</span>         ext4 /dev/vgsvc1/data@/svc1/data
              |- app#web        ...../.. <span class="up">up</span>         forking: weblauncher
              `- sync#i0        ...O./.. <span class="up">up</span>         rsync svc config to nodes
    </pre>

Service svc1 is now running on node ``demo2``. Service relocation operational is easy as that.

Now, what happens if we try to start our service on ``demo1`` while already running on ``demo2`` ? ::

        node1:/ # om svc1 start --local
        node1.svc1.ip#0        checking 192.168.121.42 availability
        node1.svc1           E start aborted due to resource ip#0 conflict
        node1.svc1             skip rollback start: no resource activated

Fortunately, OpenSVC IP address check prevent the service from starting on ``demo1``.

.. note::

        At this point, we have a 2-node failover cluster. Although this setup meets most needs, the failover is _manual_, so does not qualify as a high availability cluster.


High Availability
+++++++++++++++++

Now, we have to configure your service to be able to failover without any intervention.
You only have to change the orchestration mode to ``ha``. For more information about orchestration : `Orchestration <agent.service.orchestration.html>`_

On the node currently running your service, add ``orchestrate = ha`` in the ``DEFAULT`` section::

        # om svc1 edit config

        [DEFAULT]
        app = MyApp
        nodes = node1 node2
        orchestrate = ha
        (...)

Once this setup is in place, OpenSVC will be able to failover your service.

The last needed step is to define some resources that will trigger relocation. Those resources have to be marked as ``monitor=True`` in the service configuration file.

For example::

        # om svc1 edit config
        (...)
        [app#web]
        monitor = True
        type = forking
        start = {fs#app.mnt}/weblauncher start
        stop = {fs#app.mnt}/weblauncher stop
        check = {fs#app.mnt}/weblauncher status
        info = {fs#app.mnt}/weblauncher info


Unfreeze your service to allow the daemon to orchestrate your service::

        # om svc1 thaw

Now, if your webserver resource failed, OpenSVC will relocate the service on the other node without any human intervention.



