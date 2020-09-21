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
