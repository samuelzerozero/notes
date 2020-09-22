# Kubernetes Notes
This is a list of kubernetes notes collected while try to figure out stuff about the subject

### Table of Contents
**[Setup](#setup)**<br>
**[Pod and Containers Lifecycle](#pod-and-containers-lifecycle)**<br>
**[ReplicationController ReplicaSet DeamonSet](#replicationcontroller-replicaset-deamonset)**<br>
**[Jobs](#jobs)**<br>
**[Services](#services)**<br>



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

