 Master node: These nodes are the control ships which manage,plan,schedule and Monitor the worker nodes
 
 Worker node: These are the ships which load containers
 
 etcD Cluster: it is the key value store which stores information about the nodes
ETCD is distributed reliable key value store which simple and fast.
ETCDCTL is the CLI tool used to interact with ETCD
==./etcdctl put key1 value1
./etcdctl get key1==

 kubecontoller manager:
 -NodeController:takes care of nodes.It checks status of the nodes every 5 secs
 -Replication-Controler: takes care desired no. of container.It also monitors the status of replica set
to setup it you need to download it extract and run its as service.


 kube-scheduler: Identifies the right pod for the node. It only decides
 kubelet setups the pod.

kube-apiserver: orchestrates all operations within the server.It exposes the kubernetes api.when we run a kubectl command the request is authenticated by the kube-apiserver and information is then fetchde from ETCD cluster and then forwarded to the user .only component that directly interacts with etcd datastore. 

kubelet: It is agent that runs on each node in the cluster , it listens for instruction from kube-apiserver and create or destroy node as required
server fetches status report from the kubelets to monitor the nodes
You always have to manually download and setup the kubelet

kube-proxy:It ensures necessery rules on the worker nodes to allow comm between them.Its a process that runs every node. it creates an IP table rule onn each node of the cluster to forward traffic heading to the IP of the service to the IP of the POD

master                                               worker node
etcdcluster                                         kubelet
kubeserverapi                                    kubeproxy
kube controller manager                   container run time
kube-scheduler