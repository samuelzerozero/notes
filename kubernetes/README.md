# Kubernetes Notes
This is a list of kubernetes notes collected while try to figure out stuff about the subject

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
