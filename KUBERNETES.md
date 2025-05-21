[[some imp commands]]
container:
Completley isolated environment with shared OS kernel.It is used to rectify the compatibility issue. technologies like docker use containerization to run each code in seperate container with its own configurations
container orchestration:
managing connectivity between different containers and automatically scale up or down is called container orchestration.
-multiple instances of the application running
-user traffic is load balanced across various containers

architecture
-node: worker machine thats where container will be launched by kubernetes
-custer: set of nodes grouped together
-master: another node which manages the nodes inside a cluster responsble for  actual orchestration
components:
APIserver: frontend, conatiner runtime : docker , etcd: key-value store, scheduler, controller, kubelet: agent
master             vs            worker nodes
kube-apiserver               kubelet
etcd                                container runtime
controller
scheduler
kubectl: command line tool used to manage cluster
eg run ,nodes, cluster-info

[[CLUSTER architecture]]

POD
containers are encapsulated in kubernetes objects known as PODs.it is a single instance of a application.Smallest object in kubernetes.
node can have multiple PODs
A POD can have multiple different container.
PODs usually have one to one relation with containers running in your application.
==kubetcl run nginx --image nginx==

YAML in kubernetes
kubernetes definition file contains 4 top level fields

pod-definition.yml
```yaml

apiVersion:version of api used to create the object
kind: type of object{POD,deployment,service}
metadata:
	 name: myapp-pod
	 labels:
		 app: myapp
		 type: front-end
spec: %%additional info about object%%
	containers:
		- name: nginx
		- image: nginx
```
 
view pods 
==kubectl get pods==
==kubectl describe pod myapp-pod==

MULTICONTAINER PODS
when we need to divide large monolithic code to several microservices we use containers
When we have 2 functionality that work together like login agent and web server we use multicontainer pods
This containers are created and destroyed together and also share same network space. we do not have to establish services or volumes to communicate between them
pod-definition.yml
```yaml

apiVersion: v1
kind: POD
metadata:
	 name: myapp-pod
	 labels:
		 app: myapp
		 type: front-end
spec: 
	containers:
		- name: nginx
		   image: nginx
		   ports:
			   - containerPort: 8080	
		- name: log-agent
		   image: log-agent
```
CONTROLLERs
brain behind kubernetes. processes that monitor kubernetes objects
replication controller
helps us run multiple instances of a  pod in a kubernetes cluster.
in a single pod it helps in bringing out next pod if first one fails 
-to create multiple pod to balance the load 
replica set is the new way to set up the controller
create replicaton controller:
re-definition.yml
```yaml
apiVersion: apps/v1
kind: ReplicationCotroller
metadata:%%Replication controller parent%%
	name: rc
	labels:
		type: test
spec:
	template:
		metadata:%%POD child%%
		name: myapp-pod
		 labels:
			 app: myapp
			 type: front-end
		spec:
			containers:
			- name: nginx
			- image: nginx
	replicas: 3
```
==kubectl get replicationcontroller==
create replicaton set:
replicaset-definition.yml
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:%%Replication controller parent%%
	name: rc
	labels:
		type: test
spec:
	template:
		metadata:%%POD child%%
		name: myapp-pod
		 labels:
			 app: myapp
			 type: front-end
		spec:
			containers:
			- name: nginx
			- image: nginx
	replicas: 3
	selector:%%major difference%%
		matchLabels:
			type: test
```
==kubectl get replicaset==
to make changes in running pod
==kubectl replace -f test.yml==
To scale the no. of replica
==kubectl scale --replicas=6 -f replicaset-definition.yml== 
to see changes the the replicaset delete the currently running pods
==kubectl delete pod pod_name==
Kubernetes supports self-healing applications through ReplicaSets and Replication Controllers. The replication controller helps ensure that a POD is re-created automatically when the application within the POD crashes. It helps in ensuring enough replicas of the application are running at all times.

DEPLOYMENT
deplyment allows us to upgrade the underlying instances seamlessly
using rolling updates ,undo changes and pause or resume changes as required
it is a kubernetes object that comes higher in hierarchy than replicasets.
deployment-definition.yml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      name: httpd-frontend
  template:
    metadata:
      labels:
        name: httpd-frontend
    spec:
      containers:
	   -    name: httpd-frontend
		 image: httpd:2.4-alpine
```
ROLLOUT
when a container version is update a new rollout is triggered and a new deployment revision is created 
==kubectl rollout status deployment/name_deployment==
==kubectl rollout history deployment/d1==

Two deployment strategies
1. recreate: destroy older one and create newer instance(application down time)
2. rolling update: destroy older instance and create new instances one by one
==kubectl apply -f d1.yml== to apply the changes (trigger a new rollout)

Upgrading 
when initiating the deployment creates and  replicaset and deploys pods in it. During an update it creates new replicaset and deploy pods in it one by one and destroy pods from older replicaset one by one

to rollback changes
==kubectl rollout undo deployment/d1==
if we some misconfiguration in the deployment yaml file kubernetes will keep the older instances up and newer errored instances waiting

KUBERNETES networking
in kubernetes each ip address in assigned to a pod.
when kubernetes is initially configured it gets a ip address 10.244.0.0
when different pods are initiated they get there own ip address 10.244.0.3 , 10.244.0.2 etc
kubernetes with multiple nodes each node will obtain its own ip but its internal ip adressing will be same
*kubernetes expects us to setup a networking solution*
custom networking can be setup by usign flannel or calico

KUBERNETES services
it enable communication within different components of the applications and other external components
services allow connectivity between groups of pods
if the PODs are distributed across different nodes , kubernetes automatically make the service span across all the nodes
==kubectl get services==

nodePort service: service listens to request on that port on the node and then forward the request to the pods.ranges 30000-32767
Nodeport32008----service:80----POD:80
service-definition.yml
```yaml
apiVersion: v1
kind: Service
metadata:
	name: service
spec:
	type: NodePort
	ports:
		- targetPort: 80
		   port: 80
		   nodePort: 32008
	selector:%%to specify POD%%
		app: my-app
		type: front-end
```
clusterIP: service creates a virtual ip inside the cluster to enable communication between different services
service-definition.yml
```yaml
apiVersion: v1
kind: Service
metadata:
	name: backend
spec:
	type: ClusterIP
	ports:
		- targetPort: 80
		   port: 80
	selector:%%to specify POD%%
		app: my-app
		type: front-end
```
LoadBalancer: provides load balancer for our application. Using nodeport we give many IPs to the user(IPs of the node) to centralize this Loadbalancer is used
service-definition.yml
```yaml
apiVersion: v1
kind: Service
metadata:
	name: backend
spec:
	type: LoadBalancer
	ports:
		- targetPort: 80
		   port: 80
		   nodePort: 32008
```
[[Command & Arguments]]

NAMESPACES
it is created automatically in kubernetes when a cluster is first setup.
kubernetes creates a set of pods for internal services to isolate them from the user it creates those pods in another namespace called kube-system. 
==kubectl get all --namespace=kube-system==
to switch namespace
==kubectl config set-context $(kubectl config current-context) --namespace=kubesystem==
resources made available to all user is there in kube-public
we can create new namespaces define there own rules and policies and also defines the resources provided to those namespaces
to connect to a service in different namespace
servicename.namespace.service.domain
db-service.dev.svc.cluster.local

To limit resources in a namespace we create a resource quota
compute-quota.yml
```yaml
apiVersion: v1 
kind: ResourceQuota
metadata: 
	name: compute-quota
	namespace: dev 	
spec:
	hard:
		pods:  "10"
		requests.cpu: "4"
		requests.memory: 5Gi
		limits.cpu: "10"
		limits.memory: 10Gi
```
[[some imp commands]]
 
[[Kubernetes Scheduling]] 

Admission controller:
They help us to configure better security measures to enforce how a cluster is used. Admission controller can be used to do some additional operation before creating the pod
There are 2 types of admission controller:
	-mutating 
	-validationg
some prebuilt admission controller:
	AlwaysPullImages
	DefaultStorageClass
	EventrateLimit
	NamespaceExist
To see Admission comtroller
==kube-apiserver -h | grep enable-admission-plugins==

to add a enable admission controller we need to update them in kube-apiserver.service or /etc/kubernetes/manifests/kube-apiserver.yaml
==--enable-admission-plugins=NodeRestriction==

[[LOGGING & MONITORING]]

To view comsumption
==kubectl top pod==

Environmental variables
to set a variable we need to set a env property where we need to define the name and value of the variable
```yaml
apiVersion: v1 
kind: ResourceQuota
metadata: 
	name: compute-quota
	namespace: dev 	
spec:
	containers:
		- name: simple-webapp-color
		   image: simple-webapp-color
		   env:
			   - name: color
			   value: blue
```
ConfigMap
when we need to use lots of environmental variables in the cluster we use configMap to manage them in form key value pair
==kubectl create configmap==
to use configmap
```yaml
apiVersion: v1 
kind: ResourceQuota
metadata: 
	name: compute-quota
	namespace: dev 	
spec:
	containers:
		- name: simple-webapp-color
		   image: simple-webapp-color
		   env:
			   - name:  APP_COLOR
			   valueFrom:
			   configMapKeyRef:
				   name:  name of config map
				   key: key
```
SECRETS
used to store useful information
2 step to use secret
1. create it
2. inject it in pod
eg
==kubectl create secret generic  \ ==
	==app-secret --from-literal=DB_HOST=mysql==
```yaml
apiVersion: v1 
kind: Secret
metadata: 
	name: compute-quota
data:
	DB_Host: mysql
	DB_User: root
injecting in pod
apiVersion: v1 
kind: ResourceQuota
metadata: 
	name: compute-quota
	namespace: dev 	
spec:
	containers:
		- name: simple-webapp-color
		   image: simple-webapp-color
		   env:
			   - name:  APP_COLOR
			   valueFrom:
			   secretKeyRef:
				   name:  name of secret
				   key: key
				   %%OR%%
		   volumes:
			   - name: app-secret
				   secret:
					   secretName: app-secret
```
Secrets are not encrypted only encoded
dont push secret on github
Secrets are not encrypted in ETCD
Secrets should be present in inaccesssible namespace

AutoScaling
It consist of 2 type of scaling 
horizontal: increasing the no. of pod(HP Autoscaler)
vertical : increasing the resources in the webserver(VP Autoscaler)

horizontal scaling
To automatically scale the pods

==kubectl autoscale deployment my-app \ ==
	==--cpu-percent=50 --min=1 --max=10==
when this command is run kubernetes deploys a HPA that continously pulls metric server to monitor the usage and when usage surpass 50% it create replicas as required
hpa keeps existing pods
==k get hpa==
used for web app,microservice 

vertical scaling
similar to hpa vpa continously observes the metrics adjust the pod and manage the threshold. uunlike hpa vpa doesnt come built it we need to git clone it
VPA has 3 parts 
VPA admission controller: it intervenes the pod creation process and applies changes on it based on recommendation values
VPA Updater: updater identifies pod which needs resources and index them when update is needed it terminates them
VPA Recommender: responsible for continously monitor metric server and provide recommendation
vpa restarts the pod to apply changes
used for memory heavy apps

[[Cluster Management]]

[[SECURITY]]

[[STORAGE]]

[[NETWORKING]]

[[HELM]]


