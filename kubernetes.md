kubernetes

Manage an infrastructure cluster as a single system to simplify container operations.
Kubernetes progressively rolls out changes to your application or its configuration, 
while monitoring application health to ensure it doesn’t kill all your instances at the same time. 
If something goes wrong, Kubernetes will rollback the change.

some terminology before we play:

* cluster := all nodes/vm/hosts aggregated to one virtual system (cluster)
* pod:= group of containers (docker) that group local unit. Each container knows the other containers within a pod
* replication controllers:= you can define a replication of pods, means you can define how many instances of a pod you want to have and kubernetes will start another pod if one fails
* service:= logical set of Pods running somewhere in your cluster, that all provide the same functionality.Pods can be configured to talk to the Service, and know that communication to the Service will be automatically load-balanced out to some pod that is a member of the Service.
* label:= key-value principle to organize groups e.g. role:loadbalancer

## install to play

install Kubernetes locally by Vagrant

    export KUBERNETES_PROVIDER=vagrant
    curl -sS https://get.k8s.io | bash
    
lets login in a shell to check whether all services are running in the vm:

    $ vagrant ssh master
    $ sudo su
    $ systemctl status kubelet    
    $ journalctl -ru kubelet
    $ systemctl status docker
    $ journalctl -ru docker
    $ tail -f /var/log/kube-apiserver.log
    $ tail -f /var/log/kube-controller-manager.log
    $ tail -f /var/log/kube-scheduler.log
    $ kubectl get nodes
    $ kubectl get pods
    $ kubectl get services
    $ kubectl get replicationcontrollers
    
open https://<kubernetes-master>/ui to see the dashboard (user vagrant and password vagrant)   
    
## start containers    
    
ok now lets play with the kubernetes services itself. We will do this outside from one vm, login e.g. to master

    $ kubectl get nodes
    $ kubectl get pods
    $ kubectl get services
    $ kubectl get replicationcontrollers
    
so let's start a docker container with a nginx in it with 3 replicas, means 3 nginx instances will run   

    $ kubectl run my-nginx --image=nginx --replicas=3 --port=80
    $ kubectl get pods
    $ kubectl get pods -L run
    # -L means show label
    $ kubectl get rc
    $ kubectl describe pod my-nginx  
    $ kubectl get pods -l app=nginx -o json | grep podIP  

just for fun login into another machine (not master) to see running container

    $ vagrant ssh node-1
    $ sudo docker images    
    
we should see that 3 container instances are running and for each instance a pod was created, so let us scale them down to 2 instances
   
    $ kubectl scale rc my-nginx --replicas=2
   
lets delete them now

    $ kubectl delete pod,rc,svc -l run=my-nginx 
        
doing it by hand is not the way we want it, let's define it in a yaml which could be checked in, run-my-nginx.yaml:
            
## start services       
         
we have learned how to deploy replica sets in kubernetes, let's go to the next step, services.
We will start 2 instances as nginx services and put group them in a service

lets create 2 nginx replications again

    $ vi nginx_pod.yaml
    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: nginx
    spec:
      replicas: 2
      selector:
        app: nginx
      template:
        metadata:
          name: nginx
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80

now lets create the service
    
    $ vi nginx_service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        name: nginxservice
      name: nginxservice
    spec:
      ports:
        # The port that this service should serve on.
        - port: 80
      # Label keys and values that must match in order to receive traffic for this service.
      selector:
        app: nginx
      type: LoadBalancer
          
let's run and check it

    $ kubectl create -f nginx_pod.yaml
    $ kubectl create -f nginx_service.yaml
    $ kubectl get svc
    $ kubectl describe svc nginxservice
    $ kubectl get pods -o json | grep -i podip    
    $ curl -k http:/...
  
remark: In order to call the service by dns you have to create a pod for your tests, see DNS for this  
        
clean up

    $ kubectl delete service nginxservice
    $ kubectl delete service nginx        
      
         
a more vivid example can be found under https://github.com/kubernetes/kubernetes/tree/release-1.2/examples/guestbook/         

## Updates

a rolling update on base of versioned containers is easy to implement. lets say you have deployed nginx version 1.9.10, then 

    $ kubectl rolling-update my-nginx --image=nginx:1.9.11

would do the trick in our case

## DNS

first check whether the kubernetes (skydns) is activated (see https://github.com/kubernetes/kubernetes/blob/release-1.2/cluster/addons/dns/README.md#how-do-i-configure-it),
if not we have to do this first

    $ kubectl get services kube-dns --namespace=kube-system  
    
Our created service from the last example nginxservice should be available for other containers under the name
In order to test this we have to create a new pod and make a call from that pod

pod description

    $ cat curlpod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: curlpod
    spec:
      containers:
      - image: radial/busyboxplus:curl
        command:
          - sleep
          - "3600"
        imagePullPolicy: IfNotPresent
        name: curlcontainer
      restartPolicy: Always
  
resolve dns from within that pod  
  
    $ kubectl create -f ./curlpod.yaml
    $ kubectl exec curlpod -- nslookup nginxservice
    
    
more details see http://kubernetes.io/docs/user-guide/connecting-applications/  

## mounting volumes

see http://kubernetes.io/docs/user-guide/production-pods/ for details, but to make it simple
you can define a volume which live longes than a container by adding some entries to the replication controller
check the example

    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: redis
    spec:
      template:
        metadata:
          labels:
            app: redis
            tier: backend
        spec:
          # Provision a fresh volume for the pod
          volumes:
            - name: data
              emptyDir: {}
          containers:
          - name: redis
            image: kubernetes/redis:v1
            ports:
            - containerPort: 6379
            # Mount the volume into the pod
            volumeMounts:
            - mountPath: /redis-master-data
              name: data   # must match the name of the volume, above
              
              
## secrets

kubernetes offer an option to distribute credentials to all pods/containers
for details check http://kubernetes.io/docs/user-guide/secrets/
a simple example could look like this

secret.yaml:

    apiVersion: v1
    kind: Secret
    metadata:
      name: mysecret
    type: Opaque
    data:
      password: MWYyZDFlMmU2N2RmCg==
      username: YWRtaW4K
  
create the secrets  
  
    $ kubectl create -f ./secret.yaml  
    $ kubectl get secrets

let's use the secret as environment variable

    apiVersion: v1
    kind: Pod
    metadata:
      name: secret-env-pod
    spec:
      containers:
        - name: mycontainer
          image: redis
          env:
            - name: SECRET_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mysecret
                  key: username
            - name: SECRET_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysecret
                  key: password
      restartPolicy: Never

## container hooks

you can execute command when a new container is started or stooped, e.g. to add them to a monitoring service or to log something.
Hooks are intended to be executed “at least once”.

You have this option for execution:

* PostStart 
* PreStop

and this commands:

* HTTP - Executes an HTTP request
* Exec - Executes a specific command

so here is an example:

    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: my-nginx
    spec:
      replicas: 2
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80
            lifecycle:
              preStop:
                exec:
                  # SIGTERM triggers a quick exit; gracefully terminate instead
                  command: ["/usr/sbin/nginx","-s","quit"]


## health checks

You can define health checks in order to take sure that traffic is not send to some instances or the instances are restarted

You have this probes:

* ReadinessProbe: indicates whether the container is ready to service requests. If the ReadinessProbe fails, the endpoints controller will remove the pod’s IP address from the endpoints of all services that match the pod. 
* LivenessProbe: indicates whether the container is live, i.e. still running. The LivenessProbe hints to the kubelet when a container is unhealthy. If the LivenessProbe fails, the kubelet will kill the container and the container will be subjected to it’s RestartPolicy. 

and this actions / handlers:

* ExecAction: executes a specified command inside the container expecting on success that the command exits with status code 0
* TCPSocketAction: performs a tcp check against the container’s IP address on a specified port expecting on success that the port is open.
* HTTPGetAction: performs an HTTP Get against the container’s IP address on a specified port and path expecting on success that the response has a status code greater than or equal to 200 and less than 400.

Let's look at an example:

    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: my-nginx
    spec:
      replicas: 2
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80
            livenessProbe:
              httpGet:
                # Path to probe; should be cheap, but representative of typical behavior
                path: /index.html
                port: 80
              initialDelaySeconds: 30
              timeoutSeconds: 1

