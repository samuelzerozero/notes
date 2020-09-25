# Kubernetes Notes
This is a list of kubernetes notes collected while try to figure out stuff about the subject

### Table of Contents
**[Setup](#setup)**<br>
**[Pod and Containers Lifecycle](#pod-and-containers-lifecycle)**<br>
**[ReplicationController ReplicaSet DeamonSet](#replicationcontroller-replicaset-deamonset)**<br>
**[Jobs](#jobs)**<br>
**[Services](#services)**<br>
**[Volumes](#volumes)**<br>
**[ConfigMaps and Secrets](#configmaps-and-secrets)**<br>
**[Metadata](#metadata)**<br>
**[Deployments](#deployments)**<br>
**[StatefulSets](#statefulsets)**<br>
**[Architecture](#architecture)**<br>
**[Securing the Api Server](#securing-the-api-server)**<br>
**[Computational Resources Management](#computational-resources-management)**<br>

## Setup

#### What's the easiest way to install k8s on a local machine?
Use docker for desktop. You can find "enable kubernetes" on the options. That will launch a linux VM and install Control Plane, Kubelet, Docker daemon

#### Command line to have a like-root access to the VM on docker desktop?
```
$ docker run --net=host --ipc=host --uts=host --pid=host --privileged \
    --security-opt=seccomp=unconfined -it --rm -v /:/host alpine chroot /host
```
#### What is the program to iteract with a cluster?
*kubectl*. The client iteract only to *Api Server* and nothing else.

#### Where is the configuration file?
```
~/.kube/config
```
Otherwise is possible to set the env variable KUBECONFIG

#### How to get the info for the cluster?
```
$ kubectl cluster-info <- Server Api e DNS
$ kubectl get nodes <- All nodes in the cluster
$ kubectl describe node <nameOfNode> extra info on object
```

#### How to install a web dasboard on docker for desktop?
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc5/aio/deploy/recommended.yaml 
$ kubectl proxy
```
Now go to http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/ 
and get the token with
```
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | awk '/^deployment-controller-token-/{print $1}') | awk '$1=="token:"{print $2}'
```

## Kubectl commands and Containers

#### How to see system pods?
```
$ kubectl get pod --namespace kube-system
```

#### How to quickly create a deployment object via command line?
You now have a running application that is represented by a Deployment and exposed to the world by a Service object. Then magically scaled-out:
```
$ kubectl create deployment kubia --image=luksa/kubia:1.0 <- create deployment object
$ kubectl get deployments
$ kubectl get pods
$ kubectl describe pod
$ kubectl expose deployment kubia --type=LoadBalancer --port 8080 <- create service object
$ kubectl get svc
$ kubectl scale deployment kubia --replicas=3 <- modify the deployment object
$ kubectl get pods -o wide <- extra info (worker nodes details)
$ curl localhost:8080 <- will give extra 2 answers with the pods deployed
```

#### How are the load balancers (service) deployed?
It depends. Every provider has its own way. On minikube there's no external load balancer. In GKE for example, there is a load balancer per node (That will dispatch the service withing the pods in the node) and one per cluster, that will dispatch on the different nodes.

#### How to get all the type of objects and abbreaviations via command line?
```
$ kubectl api-resources
```

#### How is possible to explore the manifest of a node?
```
$ kubectl get node <node-name> -o yaml
```

#### Can two containers in the same pod expose the same port?
No, because containers in pods share the same Linux namespace. They can communicate through the loopback device (127.0.0.1). All the containers in a pod see also the same system hostname. 

#### What are sidecar containers?
Sidecar containers are "support" containers, that add extra functionalities to the main container and run in the same pod. An example is a reverse proxy, since that just enhance the functionality withouth modifying the original container. Other examples are log rotators and collectors, data processors, communication adapters, ...

#### Why specify container ports in pod definitions?
It's purely informative. If the container accepts connections through a port bound to its IP address, anyone can connect to it, even if the port isn’t explicitly specified in the pod spec or if you specify an incorrect port number.

#### How do you test a connection within a pod?
It's possible to create a one off pod that curl that specific api on a port, for example
```
$ kubectl get pods -o wide <- get the ip of the pod
$ kubectl run --image=tutum/curl -it --restart=Never --rm client-pod curl <ipOfThePod>:8080
```
Otherwise is possible to use a **port-forward proxy** 
```
$ kubectl port-forward kubia 8080 <- Forwarding from 127.0.0.1:8080 -> 8080
$ curl localhost:8080
```

#### How to read the logs?
If the output is on stdout and stderr the logs are saved in /var/log/containers
```
$kubectl logs <nameofPod>
--follow/f <- to follow the log
 –-timestamps=true <- to allow timestamps to be printed
```

#### Is possible to copy files from local to a pod and vice versa?
```
$ kubectl cp kubia:/etc/hosts /tmp/kubia-hosts
$ kubectl cp /path/to/local/file kubia:path/in/container
```

#### Is possible to run a command on a running pod container?
```
$ kubectl exec kubia -- ps aux
$ kubectl exec kubia -- curl -s localhost:8080
$ kubectl exec -it kubia -- bash
```
with -- to delimt the arguments to be run inside the container

#### Is possible to run a command on a specific container in a pod?
```
$ kubectl logs kubia-ssl -c kubia
$ kubectl logs kubia-ssl -c envoy
$ kubectl logs kubia-ssl --all-containers
```
If not specified, the command defaults on the first container in the manifest

#### What is an init container?
A pod manifest can specify a list of containers to run when the pod starts and before the pod’s normal containers are started.
Init containers are typically added to pods to achieve the following:
• Initialize files in the volumes used by the pod’s main containers. This includes retrieving certificates and private keys used by the main container from secure certificate stores, generating config files, downloading data, and so on.
• Initialize the pod’s networking system. Because all containers of the pod share the same network namespaces, and thus the network interfaces and configuration, any changes made to it by an init container also affect the main container.
• Delay the start of the pod’s main containers until a precondition is met. For example, if the main container relies on another service being available before the container is started, an init container can block until this service is ready.
• Notify an external service that the pod is about to start running. In special cases where an external system must be notified when a new instance of the application is started, an init container can be used to deliver this notification.

#### Where are the init containers defined?
In a pod manifest, init containers are defined in the initContainers. They will run in sequence and then all the other will starts, if successful

#### How is possible to delete object?
Deleting a pod will terminate its containers and remove them from the node. Deleting a Deployment object causes the deletion of its pods, whereas deleting a LoadBalancer-typed Service deprovisions the load balancer if one was provisioned.
```
$ kubectl delete po kubia <- by name
$ kubectl delete -f kubia-ssl.yaml <- by file
$ kubectl delete -f kubia.yaml,kubia-ssl.yaml <- multiple files
$ kubectl delete -f Chapter05/ <- all dir
```
By deleting a pod, you state that you no longer want the pod or its containers to exist. The Kubelet shuts down the pod’s containers, removes all associated resources, such as log files, and notifies the API server after this process is complete. The Pod object is then removed.
By default, the kubectl delete command waits until the object no longer exists. To skip the wait, run the command with the --wait=false option
If put autoscaling, those pods will reappear (if defined deployments)
```
$ kubectl delete svc --all (or name)
$ kubectl delete deploy --all (or name)
$ kubectl delete all --all
```

## Pod and Containers Lifecycle

#### What are the pod phases?
Phase | Description
------------ | -------------
Pending | After you create the Pod object, this is its initial phase. Until the pod is scheduled to a node and the images of its containers are pulled and started, it remains in this phase.
Running | At least one of the pod’s containers is running.
Succeeded | Pods that aren’t intended to run indefinitely are marked as Succeeded when all their containers complete successfully.
Failed |  When a pod is not configured to run indefinitely and at least one of its containers terminates unsuccessfully, the pod is marked as Failed.
Unknown | The state of the pod is unknown because the Kubelet has stopped reporting communicating with the API server. Possibly the worker node has failed or has disconnected from the network.
```
$ kubectl describe po kubia | grep Conditions: -A5
```
#### What are the container states?

Container State | Description
------------ | -------------
Waiting | The container is waiting to be started. The reason and message fields indicate why the container is in this state.
Running | The container has been created and processes are running in it. The startedAt field indicates the time at which this container was started.
Terminated | The processes that had been running in the container have terminated. The startedAt and finishedAt fields indicate when the container was started and when it terminated. The exit code with which the main process terminated is in the exitCode field.
Unknown | The state of the container couldn’t be determined.

#### What are the possible restart strategies for a container?
Restart Policy | Description
------------ | -------------
Always | Container is restarted regardless of the exit code the process in the container terminates with. This is the default restart policy.
OnFailure | The container is restarted only if the process terminates with a non-zero exit code, which by convention indicates failure.
Never | The container is never restarted - not even when it fails.

Surprisingly, the restart policy is configured at the pod level and applies to all its containers. It can’t be configured for each container individually.

The default value is Always.

#### Is it possible to define the status of liveness for a container? And for readiness?
Yes for both, that's possible defining a) An HTTP GET probe b) A TCP Socket probe c) An Exec probe
It's even possible do define a startup probe, that will run until gets healthy for the first time.

#### Is it possible to hook actions after the start and before the shutdown?
Yes, with *lifecycle: postStart or preStop

#### In the initialization phase of a pod, what are the image pull policies?
Image pull policy | Description
------------ | -------------
Not specified | If the imagePullPolicy is not explicitly specified, it defaults to Always if the :latest tag is used in the image. For other image tags, it defaults to IfNotPresent.
Always | The image is pulled every time the container is (re)started. If the locally cached image matches the one in the registry, it is not downloaded again, but the registry still needs to be contacted.
Never | The container image is never pulled from the registry. It must exist on the worker node beforehand. Either it was stored locally when another container with the same image was deployed, or it was built on the node itself, or simply downloaded by someone or something else.
IfNotPresent | Image is pulled if it is not already present on the worker node. This ensures that the image is only pulled the first time it’s required.

#### What is a grece period when in the termination phase of a container?
That's the time where the container will upgrade from SIGTERM to SIGKILL

#### What means if a liveness or readiness probe fails?
For a liveness probe, giving up means the pod will be restarted. For a readiness probe, giving up means not routing traffic to the pod, but the pod is not restarted. Liveness and readiness probes can be used in conjunction. A probe has a number of configuration parameters to control its behaviour, like how often to execute the probe; how long to wait after starting the container to initiate the probe; the number of seconds after which the probe is considered failed; and how many times the probe can fail before giving up.

## ReplicationController, ReplicaSet, DeamonSet

#### What does happen to a pod manually created if a node where is deployed fails? 
It will disappear. In order to be managed you need to create Replication Controllers or deployments.

#### What does happen when a pod label is changed if managed by a replication controller? DEPRECATED
It will end up being manually managed, hence the controller will spin automatically another one with the right label.
```
$ kubectl get pods --show-labels
$ kubectl label pod <nameOfPOd> type=special <- extra label, don't do anything
$ kubectl label pod <nameOfPOd> app=foo --overwrite <- overwrite selector, pod disconnected from controller and created a new pod
```
#### What does happen when a replication controller change the selector? DEPRECATED
if there's no pod with labels that matches the new selector, it will recreate pods from scratch

#### What does happen when a replication controller change the template? DEPRECATED
That will only be used for new span pods, won't modify existing.

#### How can scale a replication controller? DEPRECATED
via command line 
```
$ kubectl scale rc kubia --replicas=3
```
or changing and applying the *replicas* in *replicas* and apply.

#### What happens when deleted a replication controller? DEPRECATED
By default all the pods will get deleted also. To avoid that it's possible to add the flag
```
$ kubectl delete rc kubia --cascade=false
```

#### What's the difference between a ReplicaSet to a ReplicationController?
ReplicaSet can match more complicated labels (matchExpressions), and those are used by deployments. ReplicationControllers are simpler and deprecated.

#### What is a DeamonSet?
It's a special ReplicaSet but it runs one pod matching the label for each node (skips k8s scheduler).


#### Does a DeamonSet run on each node?
By default, yes. However you can specify a node selector.

## Jobs

#### Is there a way to run a finite job?
Yes, with Job. That can be defined like any other template.

#### Does a job in a pod gets deleted if that finishes?
The pod gets marked as Completed and the ready field is 0/1. The pod will be deleted when you delete it or the Job that created it.

#### What are the restartPolicy available to Jobs?
restart policy to either OnFailure or Never. Always doesn't make sense in this context and it's forbidden.

#### Is it possible to run multiple instances of the job?
Yes, adding completions: 5 (numbero of runs)

#### Is it possible to run the same job in parallel?
Yes, adding parallelism: 2 (number of threads)

#### Can increase a job parallelism on fly?
Yes, just change parallelism to a higher level. That works similar to ReplicaSet

#### Is possible to limit the duration of a job?
Yes, with activeDeadlineSeconds

#### How to schedule a job in the future, or periodically?
Using CronJob specifying syntax like linux cron.

#### For time sensitive jobs, can I force start?
Not really, but using startingDeadlineSeconds the job will be marked as fail after the seconds passed.

#### Is it guaranteed that a job is run once at time?
In normal circumstances, a CronJob always creates only a single Job for each execution configured in the schedule, but it may happen that two Jobs are created at the same time, or none at all. To combat the first problem, your jobs should be idempo- tent (running them multiple times instead of once shouldn’t lead to unwanted results). For the second problem, make sure that the next job run performs any work that should have been done by the previous (missed) run.

## Services

#### Why Services are needed?
Pods are ephemeral, Kubernetes assigns an IP address to a pod after the pod has been scheduled to a node and before it’s started and Horizontal scaling means multiple pods may provide the same service. Not knowing the configuration, services aim to provide one single IP address to serve the service as a single point of entry.
The primary purpose of services is exposing groups of pods to other pods in the cluster. But can be exposed externally also.

#### Can I connect to a load balancer with forward-proxy? 
It doesn't seems to work with docker for desktop. Removing that automatically will deploy an external one load balancing correctly. If used will just get stuck to a single pod.

#### How can I test that the service works?
a)Create a pod that will ping and then inspect the log or b) ssh into a kubernettes node or c) exec from a running node, for example  
Get the IP of the service with
```
$ kubectl get svc
```
Once recorded exec on a pod (will pick the first container if not specified)
```
$ kubectl exec pod/kubia-jwd7f -- curl -s http://<ipOfService>:8001
```

#### How to stick the request per client?
With session-affinity just adding sessionAffinity: ClientIP

#### Why there's no session-affinity for cookies session?
Kubernetes supports only two types of service session affinity: None and ClientIP. You may be surprised it doesn’t have a cookie-based session affinity option, but you need to understand that Kubernetes services don’t operate at the HTTP level. Services deal with TCP and UDP packets and don’t care about the payload they carry.

#### How is possible to map a port from service to container without knowing the port number?
Can use *named ports*. If in the Pods the contaniner port is declared (you don't really need to, it's just informative) then the service can refer 
```
spec:
    ports:
    - name: http
      port: 80
      targetPort: http
```
instead of 
```
spec: 
    ports:
    - name: http 
      port: 80
      targetPort: 8080
```

#### How do you get the ip address and port of a service from a pod?
There are 2 ways:
Every service gets injected environment variables with the naming underscore form. Executing
```
$ kubectl exec pod/kubia-9247m env
```
you can find for example KUBIA_SERVICE_PORT and KUBIA_SERVICE_HOST
Otherwise there's an internal dns-server where everything gets an ip to lookup. To get a service  
```
$ kubectl exec pod/kubia-9247m -- curl -s  http://kubia.default.svc.cluster.local:8001
$ kubectl exec pod/kubia-9247m -- curl -s  http://kubia:8001
```
where *nameOfService.namespace*.svc.cluster.local
in reality can lose the last part *svc.cluster.local* and even the *namespace* if it's assuming it's in the same. So the final is *kubia*

#### Why Ping doesn't work on a service IP?
Because that's a vitual IP.

#### What is an Endpoint?
Endpoint is something inplicit that gets called when creating a service: It's a list of IP:ports associated with a name. If no selector on service, that won't be created. It's possible to create an endpoint manually/yaml type Endpoint. 
```
$ kubectl get endpoints
```
#### How would you connect a service with an external IP?
Create a service without selectors and create an enpoint with the ips that corresponds to that name.

#### How would you connect a service with an external DNS?
Another easier way, is to create a service with spec type: ExternalName 
```
kind: Service
metadata:
    name: external-service 
spec:
    type: ExternalName
    externalName: someapi.somecompany.com
```
with local fqdn you can reach curling external-service

#### How to expose a service externally?
Setting the service type to *NodePort*, *LoadBalancer* or creating an *Ingress* resource

#### What is the differece between ClusterIP and NodePort?
ClusterIP exposes a service and a port on a constant internal IP. 
Nodeport instead opens and reserve that specific port for each node IP.

#### If connecting on a node that exposes a service Nodeport, does the pod servicing is going to be on the node itself?
It's not guaranteed. It works like a redirect on a ClusterIP really.
```
spec:
type: NodePort 
ports:
- port: 80
    targetPort: 8080
    nodePort: 30123
NAME CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kubia-nodeport 10.111.254.223 <nodes> 80:30123/TCP 2m
```
node1IP:30123 -> 10.111.254.223:80 -> pod
node2IP:30123 -> 10.111.254.223:80 -> pod

#### What happens if nodeport is not specified? 
It will assign one randomly.

#### How to get the ip address of a node?
```
$ kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'
```

#### What is the difference between Nodeport and LoadBalancer?
LoadBalancer will provision a load balancer Level4, so from 1 external IP can reach the service. If the provider doesn't have LoadBalancer
 implemented that will behave like NodePort
No firewall rules need to be changed if using a Loadbalancer (nodeport needs rules) 
```
NAME CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kubia-loadbalancer 10.111.241.153 130.211.53.173 80:32143/TCP 1m
```

#### Is possible to reduce a hop when LoadBalancer?
Yes, with externalTrafficPolicy: Local but you need to make sure that each node has a pod to connect, otherwise the connection will hang a the nodeport

#### Is the client IP preserved into the hops?
No, since SNAT is used to translate internal IPs. 
It can be preserved if with externalTrafficPolicy: Local, since there's no translation once in the node. 

#### What's the difference between LoadBalancer and Ingress?
Load balancer is connecting to nodeports that connects to 1 service, meanwhile Ingress connects directly to multiple services (or maybe nodeports).
*paths* are defined (more fine grained) and a domain needs to be declared (that binds to the CNAME)
Ingress controllers on cloud providers (in GKE, for example) require the Ingress to point to a NodePort service. But that’s not a requirement of Kubernetes itself.

#### Are ingress resources working out of the box? 
No, it depends on the cloud provider.

#### How does a request get resolved with ingress?
1) Client ping DNS server to get the controller IP
2) Ingress Controller match the host:xxx so can get the service
3) Ther service will get the endpoints
4) endpoints will give the pods that return the response
```
rules:
-host: foo.example.com http:
 paths:
   - path: / 
```
Ingress can have multiple hosts and multiple paths

#### Is possible to add TLS support on the ingress?
Yes, using tls: instead of http: and adding the certificate as a secret.

#### What is an headless service?
It's a service that returns all the pod IPs instead of returning only the one for the lookup. It's possible setting *clusterIP: None*
```
$ kubectl exec dnsutils nslookup kubia-headless
...
Name: kubia-headless.default.svc.cluster.local 
Address: 10.108.1.4
Name: kubia-headless.default.svc.cluster.local 
Address: 10.108.2.5
```
Returns all the A addresses, instead of only one. 
#### How to run a pod on-fly?
```
$ kubectl run dnsutils --image=tutum/dnsutils --generator=run-pod/v1 --command -- sleep infinity
```

## Volumes

#### How does a directory can be mounted in the context of a pod?
In the pod 2 things needs to be defined: volumeMount for the container, and a volume for the pod.
```
- image: nginx:alpine
  name: web-server 
  volumeMounts:
    - name: html <- ref of the volume delcalred
      mountPath: /usr/share/nginx/html
...
volumes: <- declaring volumes
- name: html
  emptyDir: {}
```

#### What are the available volume types?
Name | Description
------------ | -------------
emptyDir | A simple empty directory used for storing transient data.
hostPath | Used for mounting directories from the worker node’s filesystem into the pod.
gitRepo | A volume initialized by checking out the contents of a Git repository.
nfs | An NFS share mounted into the pod.
gcePersistentDisk | (Google Compute Engine Persistent Disk)
awsElastic-BlockStore | (Amazon Web Services Elastic Block Store Volume)
azureDisk | (Microsoft Azure Disk Volume)
cinder, cephfs, iscsi, flocker, glusterfs, quobyte, rbd, flexVolume, vsphereVolume, photonPersistentDisk, scaleIO | Used for mounting other types of network storage.
configMap, secret, downwardAPI | Special types of volumes used to expose certain Kubernetes resources and cluster information to the pod.
persistentVolumeClaim | A way to use a pre-or dynamically provisioned persistent storage.

#### What are the available medium of emptyDir?
Memory or Disk
```
volumes:
- name: html
emptyDir:
      medium: Memory
```

#### How long the emptyDir last? 
It's all about the pod lifecycle. when that gets deleted, that goes away.

#### What is doing gitRepo volume?
It's cloning a git repo a start time
```
volumes:
- name: html
  gitRepo:
    repository: https://github.com/luksa/kubia-website-example.git revision: master
    directory: .
```
#### Is gitRepo able to clone a private repo?
No. Use the git sync sidecar.


#### Is gitRepo in sync?
No. The only way to resync is deleting the pod and starting a new one or adding a sidecar that sync the cloned dir.

#### What is an hostPath volume? How long does it last? For what is used?
It's a mount of the node that is hosting the pod. That stays even after the pod lifecycle end. That is used for example for fluentd (logs for the node)

#### Would you use the volume hostPath for the database?
No. Since if the pod gets deleted and rescheduled on another node, the db would serve empty.

#### How can you abstract the logical part of the mounts with the specifc underlying technology?
Use PersistentVolumes and PersistentVolumeClaims, so the implemenation is decoupled:
```
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity: 
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/mongodb
```
declare the PersistentVolumeClaim so something can match the requirements:
```
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc 
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
```

And then mount the persistentVolumeClaim to the pod
```
kind: Pod
spec:
 containers:
   ...
   volumeMounts:
   - name: mongodb-data
     mountPath: /data/db
...
 volumes:
 - name: mongodb-data
   persistentVolumeClaim:
   claimName: mongodb-pvc
```

#### How to get the informations for persistent volumes?
Creating a PersistentVolume mongodb-pv
```
$ kubectl get pv
NAME         CAPACITY   RECLAIMPOLICY   ACCESSMODES   STATUS      CLAIM
mongodb-pv 1Gi Retain RWO,ROX Available
```
After the creation of PersistentVolumeClaim mongodb-pvc that matches the requirements 
```
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
mongodb-pv   1Gi        RWO,ROX        Retain           Bound    default/mongodb-pvc                           5m57s
```

#### What RWO,ROX,RWX mean when declaring a PersistentVolume?
*RWO, ROX, and RWX pertain to the number of worker nodes that can use the volume at the same time, not to the number of pods*
RWO—ReadWriteOnce—Only a single node can mount the volume for reading and writing.
ROX—ReadOnlyMany—Multiple nodes can mount the volume for reading.
RWX—ReadWriteMany—Multiple nodes can mount the volume for both reading
and writing.

#### What happens to the PersistentVolume when the the PersistenVolumeClaim is removed?
It depends on the *persistentVolumeReclaimPolicy* set on the PersistentVolume
Retain (default for manually created PersistentVolumes),
Recycle (deprecated),
Delete (default for dynamically provisioned PersistentVolumes)

#### What is a StorageClass?
Instead of manually create PersistentVolumes, an admin can define a StorageClass that acts as a template to create a PersistenVolume,
 whenever a PersistentVolumeClaim is requested.
```
apiVersion: storage.k8s.io/v1 kind: StorageClass
kind: StorageClass
metadata:
name: fast
provisioner: kubernetes.io/gce-pd 
parameters:
 type: pd-ssd
 zone: europe-west1-b 
```
and the PersistentVolumeClaim can just refer storageClassName: fast
```
$ kubectl get sc
NAME TYPE
fast kubernetes.io/gce-pd
standard (default) kubernetes.io/gce-pd
```

#### Are PersistentVolumes and StorageClass namespaced?
No.

#### Is there a default StorageClass?
Yes, defined by the provisioner
```
$ kubectl get sc standard -o yaml
```
The default storage class is what’s used to dynamically provision a PersistentVolume if the PersistentVolumeClaim doesn’t explicitly say which storage class to use.

#### What does it mean if a PersistentVolumeClaim specify storageClassName: ""? WERID
If you hadn’t set the storageClassName attribute to an empty string, the dynamic volume provisioner would have provisioned a new PersistentVolume, despite there being an appropriate pre-provisioned PersistentVolume.

## ConfigMaps and Secrets

#### What are the known methods to pass configurations to the contianer?
a) Passing command-line arguments to containers
b) Setting custom environment variables for each container
b) Mounting configuration files into containers through a special type of volume

#### Is it possible to override ENTRYPOINT and CMD in a pod? Can be changed after?
Yes, with command: and args:. Can't be updated after the pod is created

#### How to define evnironment variables?
```
containers:
- image: luksa/fortune:env
  env:
    - name: INTERVAL
      value: "30"
```

#### Can I refer to previously declared env in the env?
Yes      
```     
  env:
  - name: FIRST_VAR
  value: "foo"
  - name: SECOND_VAR
  value: "$(FIRST_VAR)bar"
```

#### What is configMap?
Is a set of key values that are stored in k8s, and can be referenced as environment variable. 
Can be different in different environment making the pods agnostic.

#### Howo to create one via command line?
```
$ kubectl create configmap fortune-config --from-literal=sleep-interval=25 --from-literal=bar=baz
$ kubectl get configmap fortune-config -o yaml
...
data:
  sleep-interval: "25" 
kind: ConfigMap
...
```

#### Can I create a ConfigMap from file?
With all the variants:
```
$ kubectl create configmap my-config 
    --from-file=customkey=config-file.conf <- assign key customkey=content of file
    --from-file=foo.json  <- foo.json=content of
    --from-file=config-opts/ <- each file name is the key and the content of the file is the value 
    --from-literal=some=thing
```

#### How to refer to a ConfigMap value in a pod?
In env using *valueFrom* instead of *value*
```
- name: INTERVAL
  valueFrom:
    configMapKeyRef:
       name: fortune-config 
       key: sleep-interval
```

#### What happens if the pod start referring to a ConfigMap that doesn't exist?
Unless configMapKeyRef.optional: true by default the pod will error. It will start automatically when the config is created


#### Can I pass all the envorinment variables at once?
Yes, with envFrom. Can even use a prefix to prepend to the key if needed.

#### What happens if the env variable is not valid like CONFIG_FOO-BAR?
It's skipped, and an event is recorded to inform the action.

#### How to mount a configMap to a directory? Can restrict a file only?
Yes
```
  volumeMounts:
     - name: config
       mountPath: /usr/share/nginx/html
       readOnly: true
 
  volumes:
  - name: config
    configMap:
      name: fortune-config
```
Yes, can be restricted with volumes with configMap set with
```
    items:
    - key: my-nginx-config.conf
      path: gzip.conf
```
Or without overriding directories use mountPath:myconfig.conf and subPath: myconfig.conf

#### Can set the permissions?
yes defaultMode: "6600"

#### What happens if a subpath file is mounted and the configmap changes?
One big caveat relates to updating ConfigMap-backed volumes. If you’ve mounted a single file in the container instead of the whole volume, the file will not be updated! 
At least, this is true at the time of writing this chapter.
You can use otherwise mount all into another directory and create a symlink instead.

#### If update a configmap, when it's possible to see the new values?
If the app does support reloading, modifying the ConfigMap usually isn’t such a big deal, but you do need to be aware that because files in the ConfigMap volumes aren’t updated synchronously across all running instances, the files in individual pods may be out of sync for up to a whole minute. 

#### What are Secrets?
Secrets are secured ConfigMaps. Kubernetes helps keep your Secrets safe by making sure each Secret is only distributed to the nodes that run the pods that need access to the Secret.
 Also, on the nodes themselves, Secrets are always stored in memory and never written to physical storage, which would require wiping the disks after deleting the Secrets from them.
```
$ kubectl get secrets 
NAME                  TYPE                                  DATA      AGE
default-token-cfee9 kubernetes.io/service-account-token 3 39d
```

#### What's the difference between stringData and data in Secrets?
stringData is plain text, meanwhile data is Base64

#### How to mount Secrets in Pods?
Like configMaps: 
```volumes:
    - name: certs
    secret:
    secretName: fortune-https
```
or via env variables like in configmap
```
env:
    - name: FOO_SECRET
      valueFrom:
        secretKeyRef:
           name: fortune-https
           key: foo
```

#### Where are Secret mounted?
Secrets are mounted in memory (tmpfs)

#### What's the best way to pass the secrets, mount or env var?
Mounting. Since env vars sometimes can be dumped in logs, or child processes can inherit them exposing to extra security vulnerabilities.

#### How to create commandline? There are different types?
Yes, to create generic, or credentials for docker hub/registry.
```
$ kubectl create secret generic fortune-https --from-file=https.key --from-file=https.cert --from-file=foo
```

#### How to set the credential for a docker registry via Secrets? (INCOMPLETE - there's an easier way rather than specify every single pod)
Having a file secret mounted on .dockercfg with:
```
$ kubectl create secret docker-registry mydockerhubsecret --docker-username=myusername --docker-password=mypassword --docker-email=my.email@provider.com
```
and using it in the pod.
```
spec:
    imagePullSecrets:
    - name: mydockerhubsecret
```

#### Metadata

## What is the Downward Api?
It's the api that contains all the metadata for the objects. It's in the Api Server. To get the api DNS:

## How can I get those infos in a pod? Are updated on a change?
a) Via environment variable using valueFrom: fieldRef: / resourceFieldRef: are not updated
b) Mounting a volume downwardAPI are updated:
```
volumes:
    - name: downward
        downwardAPI:
            items:
                - path: "podName" fieldRef:
                 fieldPath: metadata.name
```
c) Via the api using the *proxy* via the local machine
```
$ kubectl cluster-info
$ curl https://192.168.99.100:8443 -k
Unauthorized
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

d) From a pod in the cluster manually with:
* Find the location of the API server.
```
$ kubectl get svc kubernete <- via kubectl
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   42h
root@curl:/# env | grep KUBERNETES_SERVICE <- in the pod (automatically variable injected by default)
curl https://kubernetes <- DNS is already set for that
```
* Make sure you’re talking to the API server and not something impersonating it and Authenticate with the server; otherwise it won’t let you see or do anything.
```
$ ls /var/run/secrets/kubernetes.io/serviceaccount
$ export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt <-- or set --cacert so we know it's the right server
$ export TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
$ curl -H "Authorization: Bearer $TOKEN" https://kubernetes
$ export NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace) <- get the own namespace
$ curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/$NS/pods
```
e) From the pod using an *ambassador container* (a proxy that wraps TLS and authentication)

f) Use client libraries (java, python, etc)

## Deployments

#### If an update of a version for a pod is needed, what are the strategies using simple elements? What are the drawbacks?
------|-------|----
Downtime | remove the old replicaSet, update the template, apply the new replicaSet | There's an out of service
Blue Green | Spin a replicaSet With version v2 and then change the selector for the service to switch over once happy | needs to double the infrastructure
Rolling Update| Scaling down the first replicaSet and scaling up the second | error prone if manually and versions need to be compatible

#### Can put more than one resource in one single yaml file?
Yes, use the delimiter ---

#### Can do a rolling update manually?
Yes
```
$ kubectl rolling-update kubia-v1 kubia-v2 --image=luksa/kubia:v2 <- deprecated, use rollout
```
That will add another replicationController and change the labels of the pods with a deployment id (for the selection) before scaling up/down

#### Why rolling-update is obsolete?
Because it's client side (it's kubectl performing api calls) and moreover is changing labels to existing objects.
If something goes wrong, connectivity falls, that will be interrupted. Plus, it's imperative rather than declarative.

#### What is a Deployment?
A Deployment is a higher-level resource meant for deploying applications that uses ReplicaSet to generate pods. 
This is the hierarchy: Deployment -> ReplicaSet -> Pods
Basically the Deployment is the object that coordinate the dance of the ReplicaSets. It has a almost identical format as the ReplicationController/ReplicaSet

#### How do you create a deployment?
```
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  template:
```
and issue the command
```
$ kubectl apply -f kubia-deployment-v1.yaml --record 
```
#### Why to set --record when creating a deployment?
Exra information (CHANGE-CAUSE) is recorded in the history when updating a deployment

#### How to check the status of a deployment?
```
$ kubectl rollout status deployment kubia
deployment kubia successfully rolled out
```
#### What's the value in the pods and replicaset generated by a deployment?
pod kubia-1506449474-otnnh 
replicaset kubia-1506449474
1506449474 is the hash of the template, the rest is randomly generated string

#### What are the strategies available to Deployment?
RollingUpdate, Recreate

#### How to run a rollingupdate on command line? And a Rollback?
```
$ kubectl set image deployment kubia nodejs=luksa/kubia:v2
$ kubectl rollout status deployment kubia <- check the status
$ kubectl rollout undo deployment kubia <- rollback to the previous version
```

#### Why in a rolling update there's the old ReplicaSet hanging?
To allow a possible rollback

#### How can I see the history of the updates?
```
$ kubectl rollout history deployment kubia
```

#### What is kubectl patch?
Modify a property without edit the file 
```
$ kubectl patch deployment kubia -p '{"spec": {"minReadySeconds": 10}}'
```

#### How to rollback to a specific version?
```
$ kubectl rollout history deployment kubia <- get the revison number
$ kubectl rollout undo deployment kubia --to-revision=1
```

#### How to limit clutter on the history of a deployemnt?
set revisionHistoryLimit (default is 10)

#### What is maxSurge and maxUnavailable?
maxUnavailable is the number of pods that are under the replicas that can be tollerated (default 25% of replicas)
maxSurge is the cumulative max number of pods that can be span during deployment over the replicas (default 25% of replicas)
```
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
```

#### How to test a deployment on some instance? (Canary Release)
It's possible to roll out a new version and pause straight after
```
$ kubectl set image deployment kubia nodejs=luksa/kubia:v4
$ kubectl rollout pause deployment kubia
```
If everything is fine you can continue the rollout
```
$ kubectl rollout resume deployment kubia
```
or rollback.
Or better, use another Deployment object

#### what is minReadySeconds?
(NEEDS TO CHECK, COULD NOT UNDERSTAND PROPERLY)
To be considered available, it needs to be ready for at least 10 seconds (minReadySeconds) otherwise that will never become available and block the deployment

#### Does a rollout have a deadline?
By default, after the rollout can’t make any progress in 10 minutes, it’s considered as failed.
If you use the kubectl describe deployment command, you’ll see it display a ProgressDeadlineExceeded condition, as shown in the following listing.
Can be configured with *progressDeadlineSeconds*

## StatefulSets

#### What problem StatefulSets try to avoid?
Permanent IP addresses (for static clusters-like mongo) and individual PersistentVolumes for ReplicaSets (by default all pods in replicaSet have the same spec template, so pointing to the same)

#### What happens when StatefulSets dies?
When a stateful pod instance dies (or the node it’s running on fails), the pod instance needs to be resurrected on another node, but the new instance needs to get the same name, network identity, and state as the one it’s replacing. This is what happens when the pods are managed through a StatefulSet.

#### Is possible to scale up and scale down StatefulSets?
Yes, but scale up once at time, and scale down the same, and blocked if any of the instance is unhealthy.

#### How to create PersistentVolumes/PersistentVolumeClaims for a memeber of StatefulSet?
Using a PersistenVolume template. That will assign a consitent naming like the pods.
 If scaling down the claim won't be deleted automatically and if rescaled up, the Pod will attach the same volume claim (same name)
Kubernetes guarantees *at-most-one* semantic for statefulSets

#### How to create a governing Service?
Just set to healdless setting the clusterIP field to None
```
kind: Service
metadata:
  name: kubia
spec:
  clusterIP: None
 ```

#### How to declare a StatefulSet?
```
kind: StatefulSet
metadata:
  name: kubia
spec:
   ...
    volumeClaimTemplates:
    - metadata: name: data
        spec:
          resources:
            requests:
              storage: 1Mi
          accessModes:
            - ReadWriteOnce
```
it's like a replicaSet but have volumeClaimTemplate instead of volumeClaim


#### How to hit an endpoint via proxy and api? (?????Magic)
```
$ kubectl proxy
$ curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
$ curl -X POST -d "Hey there! This greeting was submitted to kubia-0." localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
```

#### Even if there's a service headless, can I use as well a non headless service?
Yes, that will dispatch round


#### can group items in the yaml?
Yes also with kind: List or ---

## Architecture

#### Which are the components of k8s?
The Kubernetes Control Plane and The (worker) nodes.

#### What are the components of the control plane?
a) The etcd distributed persistent storage
b) The API server
c) The Scheduler
d) The Controller Manager

#### What are the components of the worker nodes?
a) The Kubelet
b) The Kubernetes Service Proxy (kube-proxy)
c) The Container Runtime (Docker, rkt, or others)

#### What are the add-on components?
a) The Kubernetes DNS server
b) The Dashboard
c) An Ingress controller
d) Heapster
e) The Container Network Interface network plugin 

#### How the various components communicate?
**Everything goes through the API server**. Basically all the components watch that as a single source of truth and react accordingly incase of state change.

#### How to get the healcheck of the components via command line?
```
$ kubectl get componentstatuses
$ kubectl get po -o custom-columns=POD:metadata.name,NODE:spec.nodeName --sort-by spec.nodeName -n kube-system
```

#### How the control pane elements can be replicated?
Api servers can be deployed and active in multiple locations. Schedulers and Control manager can be replicated, but only one active at time.

#### How works ETCD replication?
In case of network split, the majority accept updates and the other is blocked. RAFT consensus algorithm.
in Kubernetes, the only etcd client is the API server, but there may be multiple instances

#### What is admission control in Api Server?
If the request is trying to create, modify, or delete a resource, the request is sent through Admission Control. 
It doesn't go through if just tries to read.

#### What a Kubelet does?
In a nutshell, the Kubelet is the component responsible for everything running on a worker node. Its initial job is to register the node it’s running on by creating a Node resource in the API server. 
Then it needs to continuously monitor the API server for Pods that have been scheduled to the node, and start the pod’s containers.
The Kubelet is also the component that runs the container liveness probes, restart- ing containers when the probes fail. 
Lastly, it terminates containers when their Pod is deleted from the API server and notifies the server that the pod has terminated

#### Is kube-proxy really a proxy?
No, it was initially but now it only takes care to modify ipTables to have a direct link.

#### What is a pause container?
It's the infrastructure container that spins when spinning a pod.

#### Is there NAT happening between pods and nodes?
No, It's handled via Container Network Interface (CNI)  plugin. Only for services outside, since they need the public ip translation.

#### How services gets configured by which component?
It's the kube-proxy that takes care of that. Are virtual addresses that just return other ips. 

#### Can I have high availability with scaling=1?
Yes, can use some techniques of using sidecars for leader-election (having a dormant replica, ready to go if the other pods fails)
https://github.com/kubernetes-retired/contrib/tree/master/election

#### What are the limits of making Control Plane components highly available?
Component | Description
------|--------
etcd | best partition - use 3 (1 failure tollerance),5,7 nodes. More than 7 is impacting performance
server Api | stateless, can horizontally scale
Controller manager | 1 active (elected leader), the other dormants
Scheduler | 1 active (elected leader), the other dormants

## Securing the Api Server

#### What happens when trying to connect to api server?
The connection run through the first matching Authentication plugin, then authorization. 
That can be obtained from the client certificate,an authentication token passed in an HTTP header,Basic HTTP authentication, or others.

#### What type of users are in kubernetes?
1) Actual humans (users)

2) Pods (more specifically, applications running inside them)

#### What is a serivce account?
Pods have information stored in resources ServiceAccount, as a mechanism for authentication for non human.

#### What are groups, how that works in k8s?
It's several users grouped. The group is a string returned by the controller or system groups

Name | Description
------|--------
system:unauthenticated |  used for requests where none of the authentication plugins could authenticate the client.
system:authenticated |   automatically assigned to a user who was authenticated successfully.
system:serviceaccounts | encompasses all ServiceAccounts in the system.
system:serviceaccounts:<namespace> | includes all ServiceAccounts in a specific namespace.

#### How does it work an authentication for a pod
authenticate with a secret mounted on
```
/var/run/secrets/kubernetes.io/serviceaccount/token
```
and return a username like, that is then passed to the authorisation controller:
```
system:serviceaccount:<namespace>:<service account name>
$ kubectl get sa
NAME      SECRETS   AGE
default   1         2d22h
```

#### Can a pod  use a ServiceAccount from the another namespace?
No

### Why to create more service accounts apart from the default one?
Bacause of security, restricting pods that don't need metadata, modify objects, read etc.

#### How to create a service account?
```
$ kubectl create serviceaccount foo
$ kubectl describe sa foo
$ kubectl describe secret foo-token-qzq7j <- the secret contains ca,namespace and token 
```
or on yaml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account 
    imagePullSecrets: 
      - name: my-dockerhub-secret
```

#### How to create a RBAC autorisation?
Creating a resource Roles and ClusterRoles,RoleBindings and ClusterRoleBindings (the difference bewtween roles and clusterRoles is that the first is namespaced)
RoleBinding can bind a clusterRole.

#### How to create/get a pod for a different namespace?
Same commands as usual, but specifying -n <namespace>. Otherwise is "default"
```
$ kubectl create ns foo
$ kubectl get po -n foo
$ kubectl exec -it test-7b4bb6b9ff-47scj -n foo sh
```
https://github.com/docker/for-mac/issues/3694
#### How to create a Role
```
kind: Role
metadata:
  namespace: foo
name: service-reader 
 rules:
  - apiGroups: [""]
    verbs: ["get", "list"] 
    resources: ["services"]

$ kubectl create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo

kind: RoleBinding
metadata:
name: test namespace: foo
roleRef:
 apiGroup: rbac.authorization.k8s.io 
 kind: Role
 name: service-reader
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
```

#### How to bind a role to a user instead or a group?
To bind a Role to a user instead of a ServiceAccount, use the --user argument to specify the username. To bind it to a group, use --group

#### How many service accounts should be created?
It’s a good idea to create a specific ServiceAccount for each pod (or a set of pod replicas) and then associate it with a tailormade Role (or a ClusterRole) through a RoleBinding (not a ClusterRoleBinding,
 because that would give the pod access to resources in other namespaces, which is probably not what you want)
Don’t add all the necessary permissions required by both pods to the default ServiceAccount in the namespace.

#### How to mitigate if app is compromised?
Your aim is to reduce the possibility of an intruder getting hold of your cluster. 
Today’s complex apps contain many vulnerabilities. You should expect unwanted persons to eventually get their hands on the ServiceAccount’s authentication token,
 so you should always constrain the ServiceAccount to prevent them from doing any real damage.
 
#### Can you use the pod with a real host network of the node? Can I bind a real port?
Yes, hostNetwork: true. If you just need the port, can specify hostPort to just bind the real port rather than the whole network namespace.

#### What is the container security context?
Besides allowing the pod to use the host’s Linux namespaces, other security-related features can also be configured on the pod and its container through the security- Context properties, which can be specified under the pod spec directly and inside the spec of individual containers.
Specify the user (the user’s ID) under which the process in the container will run.
 Prevent the container from running as root (the default user a container runs as is usually defined in the container image itself, so you may want to prevent
containers from running as root).
 Run the container in privileged mode, giving it full access to the node’s kernel.
 Configure fine-grained privileges, by adding or dropping capabilities—in con-
trast to giving the container all possible permissions by running it in privi-
leged mode.
 Set SELinux (Security Enhanced Linux) options to strongly lock down a
container.
 Prevent the process from writing to the container’s filesystem.

#### How Running a container as a specific user? 
```
spec:
containers:
   securityContext:
      runAsUser: 405
```

#### How avoid a container as a non root user? 
```
spec:
containers:
   securityContext:
      runAsNonRoot: true
```

#### How run a container as a privileged?
securityContext.privileged: true 

#### How ro fine grain add/remove capabilities?
Get the capabilities bur remove the linux previx *CAP_* when specified:
```
capabilities:
   add:
     - SYS_TIME
    drop:
     - CHOWN
```
#### How to make the fs readonly?
readOnlyRootFilesystem: true. Still can mount a volume to allow rw

#### Can I set the securityContext to a pod level as well?
Yes, at pod.spec.securityContext

#### How to share a mounted volume between 2 containers running with different userIDs?
spec.securityContext.fsGroup and spec.securityContext.supplementalGroups. The fs group is the one that will be used for the FS, additional the others

#### How is possible to centralise the restrictions?
creating PodSecurityPolicy:
 Whether a pod can use the host’s IPC, PID, or Network namespaces  Which host ports a pod can bind to
 What user IDs a container can run as
 Whether a pod with privileged containers can be created
 Which kernel capabilities are allowed, which are added by default and which are always dropped
 What SELinux labels a container can use
 Whether a container can use a writable root filesystem or not
 Which filesystem groups the container can run as
 Which volume types a pod can use
```
kind: PodSecurityPolicy
metadata:
  name: default
spec:
  hostIPC: false
  hostPID: false
  hostNetwork: false
  hostPorts:
  - min: 10000
    max: 11000
  - min: 13000
    max: 14000
  privileged: false
  readOnlyRootFilesystem: true
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  volumes:
  - '*'
```

#### Can restrict ranges of ids, guids etc on PodSecurityPolicy?
Yes,   rule: MustRunAs ranges:min/max

#### Can constraint the type of volume that must be used on PodSecurityPolicy?
Yes

#### Can assign PodSecurityPolicy to ClusteRole to users?
Yes, 

#### Can Isolate the pod network?
Yes, with NetworkPolicy
```
kind: NetworkPolicy
metadata:
name: postgres-netpolicy spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
- from:
- podSelector:
        matchLabels:
          app: webserver
ports:
- port: 5432
```

#### Can Isolate between namespaces?
Yes
```
spec:
  podSelector:
    matchLabels:
app: shopping-cart ingress:
- from:
- namespaceSelector:
        matchLabels:
          tenant: manning
ports:
- port: 80
```

#### Can limit outbound egress?
Yes

#### Can limit ip ranges?
yes, for example with ingress.ipblock.cidr: 192.168.1.0/24

## Computational Resources Management

#### Why if there's available 2CPUs in a node I won't be able to place 2 containers with 1CPU requested?
Because there are other system pods.


#### How to define what to kill first if used the total of shared resources?
assign Quality of Service (QoS) classes
BestEffort (the lowest priority)  Burstable
 Guaranteed (the highest)

#### Can create limits per pod namespace?
LimitRange, that extends to volumeclaims and not only memory and cpu
```
kind: LimitRange
metadata:
  name: example
spec:
limits:
- type: Pod
    min:
      cpu: 50m
      memory: 5Mi
max: cpu: 1
memory: 1Gi
- type: Container defaultRequest:
type: PersistentVolumeClaim min:
    storage: 1Gi
  max:
    storage: 10Gi
A LimitRange can also set the minimum and maximum amount of storage a PVC can request.
 
```

#### What are ResourceQuota?
A ResourceQuota limits the amount of computational resources the pods and the amount of storage PersistentVolumeClaims in a namespace can consume. It can also limit the number of pods, claims, and other API objects users are allowed to create inside the namespace. Because you’ve mostly dealt with CPU and memory so far, let’s start by looking at how to specify quotas for them.

#### How to set the right quotas?
Monitoring, collecting stats using Heapster

#### How heapster works?
Is an additional component that is getting info from the cAdvisor (a component of Kubelet)

#### How to install Heapster?
TO DO, not installed on docker for desktop mac. Some clooud providers by default.
```
$ kubectl top node
```
#### What to use to view the data collected?
InfluxDb (time series db) and Grafana (analitics and visualization)
