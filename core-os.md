# coreOs

OS desingned for running containers:

* contains leet := distributes and starts container (schedules) on several hosts
* contains etcd: = service discovery, means configuration of containers by distributed key value based db. Data stored in etcd is distributed across all of your machines running CoreOS
* automatic update mechanism to increase security
* atomic package installation
* contains only the kernel and container tools
* minimal foodprint

## Demo

Demo with vagrant:

* clone https://github.com/coreos/coreos-vagrant.git
* rename config.rb.sample to config.rb
* set instances in config.rb to 3
* rename user-data.sample to user-data
* replace <token> with value from https://discovery.etcd.io/new in user-data
* vagrant up
* vagrant status

So lets play:

* open 3 shells and login in the 3 different vagrant machines
* connect to one machine by vagrant in shell 1: vagrant ssh core-01 -- -A
* connect to one machine by vagrant in shell 2: vagrant ssh core-02 -- -A
* connect to one machine by vagrant in shell 2: vagrant ssh core-03 -- -A

etcd:

* set value in etcd db in core-01 by: etcdctl set /message "Hello world"
* read the value in core-02 and core-03 by: etcdctl get /message

do the same with curl

    curl -L http://127.0.0.1:2379/v2/keys/message -XPUT -d value="Hello world"
    curl -L http://127.0.0.1:2379/v2/keys/message

docker:

run a container in one shell and stop it: docker run busybox /bin/echo hello world

fleet:

Using the fleetctl tool, you can query the status of a unit, remotely access its logs and more.
Standard units are long-running processes that are scheduled onto a single machine. If that machine goes offline, the unit will be migrated onto a new machine and started.
Global units will be run on all machines in the cluster.

    $ fleetctl list-units
    $ fleetctl list-machines

hello.service

    [Unit]
    Description=My Service
    After=docker.service

    [Service]
    TimeoutStartSec=0
    ExecStartPre=-/usr/bin/docker kill hello
    ExecStartPre=-/usr/bin/docker rm hello
    ExecStartPre=/usr/bin/docker pull busybox
    ExecStart=/usr/bin/docker run --name hello busybox /bin/sh -c "trap 'exit 0' INT TERM; while true; do echo Hello World; sleep 1; done"
    ExecStop=/usr/bin/docker stop hello

    # uncomment this lines to let the container run on all machines (global)
    #[X-Fleet]
    #Global=true

back to commandline:

    $ fleetctl load hello.service
    $ fleetctl start hello.service
    $ fleetctl status hello.service
    $ fleetctl list-units
    $ fleetctl list-machines
    $ fleetctl destroy hello.service

aditional tasks:

* now make service global running and start it
* simulate failover (hint you can use vagrant suspend core-01 to stop one vm)

more see https://coreos.com/os/docs/latest/booting-on-vagrant.html
