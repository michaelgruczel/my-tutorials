# mesos

## short explanantion and terminology

Mesosphere lets you run distributed apps, on resources pooled across an entire datacenter or cloud.
That means you can deploy containers and schedule batch job and mesoshere takes sure that the instance are distributed effectively.

Elements:

* Marathon := container orchestrator that starts and monitors applications and services.
* Chronos := Cron for your entire datacenter
* Apache Mesos := Core of the Mesosphere DCOS. It enables an operating system that spans the entire datacenter.
* Mesosphere DCOS Services := Additional Services and libraries like Spark, Kafka, Hadoop and Cassandra 

## let's play


### setup

you need:

* git
* vagrant
* virtual box

setup boxes

    git clone https://github.com/mesosphere/playa-mesos.git
    cd playa-mesos
    bin/test
    vagrant up
    vagrant ssh
    ps -eaf | grep mesos
    curl http://10.141.141.10:8080/v2/apps

### Marathon: create application


Let's create 2 instances of nginx

    $ vi nginx.json
     
    {
        "id": "nginx",
        "container": {
          "docker": {
            "image": "nginx",
            "network": "BRIDGE",
            "portMappings": [
              { "containerPort": 80, "hostPort": 0, "protocol": "tcp"}
            ],
            "parameters": []
          }
        },
        "cpus": 0.2,
        "mem": 32.0,
        "instances": 2
    }      
    
hostPort 0 means mesos should select a free port

     $ curl -vX POST http://10.141.141.10:8080/v2/apps -d @nginx.json --header "Content-Type: application/json"
     $ curl http://10.141.141.10:8080/v2/apps
     $ curl http://10.141.141.10:8080/v2/apps/nginx

details of the api can be found here http://10.141.141.10:8080/api-console/index.html or here https://mesosphere.github.io/marathon/docs/rest-api.html

connect to the Mesos Web UI on 10.141.141.10:5050 and the Marathon Web UI on 10.141.141.10:8080 to see the result as well,
maybe test whether you can reach one instance from outside e.g http://10.141.141.10:31551 (check your port and ip, it might differ)

## WARNING: not tested from this point

### cli (WARNING: only usable with account)

let's install a cli in order to get rid of curl commands see https://docs.mesosphere.com/administration/cli/
we are still in the box

     $ git clone https://github.com/mesosphere/dcos-cli.git
     $ cd dcos-cli/
     $ sudo apt-get install python-pip
     $ sudo pip install virtualenv
     $ make env
     $ make packages
     $ cd cli
     $ make env
     $ source bin/env-setup-dev
     $ dcos config set core.dcos_url http://10.141.141.10:5050
     $ dcos marathon app list
     $ dcos marathon app update nginx instances=4

more see https://docs.mesosphere.com/getting-started/tutorials/deploy-containerized-app/    

### remove everything

    $ vagrant destroy   
    
### cluster setup

    $ git clone https://github.com/everpeace/vagrant-mesos.git
    $ vi multinodes/cluster.yml
    $ cd multinodes
    $ vagrant up
    
access:
    
* Mesos web UI on: http://172.31.1.11:5050
* Marathon web UI on: http://172.31.3.11:8080
* Chronos web UI on: http://172.31.3.11:8081         