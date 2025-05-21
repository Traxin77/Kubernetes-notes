Before performing networking operations on kubernetes we will first try out some networking in docker 
when running a docker container there are different networking options to choose from 
==docker run --network none nginx==
with none network the container cannot reach the outside world

==docker run --network host nginx==
the container is attached to the host network if we have defined container to listen on port 80 we can access the directly by the ip of host and port 80 . another container listening on same port wont be allowed

==docker run nginx==
third networking option is bridge where internal private network is created which the docker host and containers attach to. Each device connected will get their own internal private network address.

on the bridge we can connect the  interface named docker0 will be created . Whenever  a container is created docker creates a network namespace for it. it creates another interface in the container like eth0@if10 and creates and creates a interface pair between the container and the bridge.
Now we have created a private network where every container can access each other. 
for the external users to access the application docker provides port mapping option 
==docker run -p 8080: 80 nginx==
docker does thisby adding the rule in the iptables
iptables \
	-t nat \
	-A DOCKER \
	-j DNAT \
	--dport 8080 \
	--to-destination 172.17.0.3:80
to see the rules created by the docker 
==iptables -nvL -t nat==

Container network interface
It is a set of standards that defines how programs should be developed to solve networking challenges in a container runtime environment. Program are referred to as plugins  in this case bridge program is plugin to CNI.
CNI defines set of responsibilities for container runtime and plugins
rules for CR
- it must create network namespace
- Identify network the container must attach to
- invoke Network plugin(bridge) when container is added or deleted
- JSON format for network configuration
rules for plugin
- must support command line argument for ADD/DEL/CHECK
- must support parameters like container id , network etc
- must manage IP address  assignment to PODs
- return result in particular format
some CNI interfaces eg bridge VLAN, windows, IPVLAN etc
some third party plugin like weave ,cilium ,flannel also works well because they follow CNI standards

Docker has its own set of standards called  CNM(container network model) . due to the differences the plugins don't integrate with docker 
to deal with this issue when kubernetes use docker container
it creates them in none network
==docker run --network=none nginx ==
it the invokes configured CNI plugins to take care of configuration
==bridge add 2e34dcf34 /var/run/netns/2e34dcf34==

Cluster Networking
Networking is a central part of Kubernetes, but it can be challenging to understand exactly how it is expected to work. There are 4 distinct networking problems to address:

1. Highly-coupled container-to-container communications: this is solved by Pods and localhost communications.
2. Pod-to-Pod communications: this is the primary focus of this document.
3. Pod-to-Service communications: this is covered by Services.
4. External-to-Service communications: this is also covered by Services.
Kubernetes clusters require to allocate non-overlapping IP addresses for Pods, Services and Nodes, from a range of available addresses configured in the following components:

- The network plugin is configured to assign IP addresses to Pods.
- The kube-apiserver is configured to assign IP addresses to Services.
- The kubelet or the cloud-controller-manager is configured to assign IP addresses to Nodes.
Kubernetes clusters, attending to the IP families configured, can be categorized into:

- IPv4 only: The network plugin, kube-apiserver and kubelet/cloud-controller-manager are configured to assign only IPv4 addresses.
- IPv6 only: The network plugin, kube-apiserver and kubelet/cloud-controller-manager are configured to assign only IPv6 addresses.
- IPv4/IPv6 or IPv6/IPv4 dual-stack:
    - The network plugin is configured to assign IPv4 and IPv6 addresses.
    - The kube-apiserver is configured to assign IPv4 and IPv6 addresses.
    - The kubelet or cloud-controller-manager is configured to assign IPv4 and IPv6 address.
    - All components must agree on the configured primary IP family.

Kubernetes clusters only consider the IP families present on the Pods, Services and Nodes objects, independently of the existing IPs of the represented objects. Per example, a server or a pod can have multiple IP addresses on its interfaces, but only the IP addresses in `node.status.addresses` or `pod.status.ips` are considered for implementing the Kubernetes network model and defining the type of the cluster
to get ip of a node 
==k get nodes -o wide==

POD networking
when we have configured the network that connects all the nodes we all need to configure the network at the pod layer,kubernetes does not comes with a built in solution for this
kubernetes expects:
- every pod to have IP address
- every pod should be able to communicate with every other pod in same node
- every pod should be able to  communicate with every other pod in different nodes without NAT
To establish connection between internal pods we can create bridge 1.network in each node and bring them up
==ip link add v-net type bridge | ip link set dev v-net up==
2.next we set ip address for bridge interface
==ip addr add 10.244.1.1/24 dev v-net==
3.To attach container to the network.
the below script follows CNI standards
```sh
ADD)
#create veth pair
ip link add .....

#Attach veth pair
ip link set ...

#Assign IP Address
ip -n <namespace> addr add ....
ip -n <namespace> route add ....
#Bring UP interface
ip -n <namespace> link set ....
DEL)
ip link del ...
```
run this script on all the container in the node so they can talk to each other

4.for the pod to connect to different pod in other nodes we need to route traffic to the private network of other nodr throuhg out node
==node1$ ip route add 10.244.2.2 via 192.168.1.12==
10.244.2.2(address of private network)
192.168.1.12(address of other node consisting that network)
now the  pod in node1 can ping other pod in node2

CNI in Kubernetes
CNI plugin must be invoked by the component  that is responsible for creating containers eg containerd or crio. The network plugins are installed in /opt/cni/bin , configuration of the plugins are stored in /etc/cni/net.d inside the net.d directory it consist of the conf file using CNI standards

IPAM(IP address management)
CNI  says its the role of the network plugin to care of assigning IP.
CNI comes with 2 plugins DHCP and host-local which help in managing the IP  in file /etc/cni/net.d/net-script.conf consist a section called IPAM where we can specify type of plugin to use , the subnet and routes to be used.

Service networking
when a service is created it is accessible from all pods on the cluster irrespective of node, when a service is used to connect to network to get a IP it is a ClusterIP type of service. 
If we want a pod to host a web app that can be accessed by external entity it will use the service NodePort. This service is also used to get a assigned ip and it also exposes the application on a port on all nodes in the cluster
we have seen the pods have containers and namespaces and ip assigned , with services its just a virtual object. When we create a service object it is assigned an ip from a predefined range 
to get the range 
==cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep cluster-ip-range==

the kube-proxy component gets that IP and creates forwarding rules on each node in the cluster(any request to this ip of service should go to the ip of the pod).
Now to create these rules kube-proxy, kube-proxy supports different ways such as 
- userspace where it listen on a port for each service and proxies connections to the pods  
- creating IPVS rules
- usign IP tables
to set proxy mode

==kube-proxy --proxy-mode userspace | iptables | ipvs==
to view the assigned proxy mode
==k logs -n kube-system kube-proxy==

DNS in kubernetes
Whenever a cluster is deployed it deploys a DNS automatically. In internal pod connectivity when we create a service to connect 2 pods DNS maps the service ip to a name resolution so any pod can         curl http://web-service to reach that pod provided both pod and service are in same namespace
if the service were in namespace apps we will use 
curl http://web-service.app
All the services are grouped together in another sub-domain called svc all the  services and pods are grouped together for a root domain of the cluster called cluster.local so the full qualified domain name would be 
curl http://web-service.app.svc.cluster.local
if we need to put DNS record of the pod as well the, kubernetes generates the name by replacing the . with - 
so its domain name would be 
http://1-244-2-5.app.pod.cluster.local

With kubernetes version 1.12 the recommended DNS server is CoreDNS 
The core-dns server is deployed as a pod in the kube-system namespace in the kubernetes cluster. They are deployed as 2 pof for redundancy as a part of a ReplicaSet. This pod runs coredns executable. to run it requires a Corefile located at /etc/coredns
This file is passed in the coredns pod as a configmap object .

When we create the coredns up and running it also create a service for other pods to reach it, the service is named kube-dns by default

INGRESS
it is like a layer 7 load balancer built into the kubernetes cluster that can be configured using native primitives just like any other object.
ingress is used by kubernetes for deploying the load balancing solution like nginx and traeffic in ingress controller and then define configuration rules in ingress resources.
kubernetes doesnt comes with ingress controller by default we need to deploy it. load balancer component are just part of controller it also has additional intelligence built into them , for new definition or ingress resources.
to create a ingress controller with nginx image you would need:
- load balancer deployment
- service to expose it
- configmap to feed nginx config data
- service account with configured roles and role binding to access data
Ingress Resources
it is set of rules applied to a ingress controller to route the traffic based on the domain or url to different apps deployed
ingress.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: ingress-wear
spec:
	rules:
	- http:
		paths:
		- path: /wear
			backend:
				service:
					name: wear-service
				port:
					number: 80
		- path: /watch
			backend:
				service:
					name: watch-service
				port:
					number: 80

```
Limitation of Ingress
- It would pose a challenge in a multi-tenancy environment
- It only supports http based rules such as host matching or path matching
- to pass specification we need to pass them through annotation where each third party supplier have their own different rule 
to pass these limitation gateway api are introduced
gateway api has 3 component which are:
GatewayClass:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cluster-gateway
spec:
  controllerName: "example.net/gateway-controller"

```
Gateway:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

HTTPRoute:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-httproute
spec:
  parentRefs:
  - name: example-gateway
  hostnames:
  - "www.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /login
    backendRefs:
    - name: example-svc
      port: 8080
```
we can also use it for some other protocols like TLSRoute, GRPCRoute etc