It watches for new pod without a nodeName assigned.finds node which meet pods need then assigns that node to the pod by adding it to the nodeName
pod-definition.yml
```yaml
apiVersion: v1
kind: Pod
metadata:
	 name: myapp-pod
	 labels:
		 app: myapp
		 type: front-end
spec:
	nodeName: node01
	containers:
		- name: nginx
		   image: nginx
```
kubernetes wont allow you to change the nodename property of a pod another way to assign a node is to create binding object and send post request
Pod-bind-definition.yaml
```yaml
apiVersion: v1
kind: Binding
metadata: 
	nginx: nginx
target:
	apiVersion: v1
	kind: Node
	name: node01
```
add the name of the node to property then send a post request to the pods binding object with data where pod-bind-definition.yaml in xml format
Taints & tolerations
It is used to set the restrictions on what pods can be scheduled on a node
if we apply a taint = blue on a node then no pods will be scheduled on it if want certain pod to be scheduled on it we will apply toleration on that pod
How to apply Taint 
==kubectl taint nodes node-name key=value:taint-effect==
taint-effects:
	Noschedule:pods will not be scheduled
	PreferNoSchedule: sys will try to not schedule but not guaranteed 
	NoExecute:new pods wont be schduled and current will be evicted if toleration is changed

How to apply Toleration
Pod-.yaml
```yaml
apiVersion: v1
kind: Pod
metadata: 
	nginx: nginx
spec:
	containers:
		-name: name
		  image: nginx
	tolerations:
		- key: "app"
		operator: "Equal"
		value: "blue"
		effect: "NoSchedule"
```
t&T only filters the pods
to restrict pods to certain nodes it can be attained through node-affinity 
Node Selector
if we want to run a pod on a selected node we can add nodeSelector paramerter to it and add a key:value pair to it
```yaml
apiVersion: v1
kind: Pod
metadata: 
	nginx: nginx
spec:
	containers:
		-name: name
		  image: nginx
	nodeSelector:
		size: Large
```
for it to work you need to label the nod ebeforehand
==kubectl label nodes node01 size=Large==

Node Affinity
node selector doesnt have advanced expressions like OR or NOT.
node Affinity provides us complex way to assign pods to node
```yaml
apiVersion: v1
kind: Pod
metadata: 
	nginx: nginx
spec:
	containers:
		-name: name
		  image: nginx
	affinity:
		nodeAffinity:
			requiredDuringSchedulingIgnoredDuringExecution:
				nodeSelectorTerms:
					- matchExpressions
						- key: size
						   operator: In / NotIn
						   values:
						   - Large
						   - Medium
```
 Resource request:
 minimum amount of cpu and memory requested by a container.kube-scheduler uses thes enu. to identify which node has sufficient amount of resources
 to add resource info in pod
 ```yaml
 apiVersion: v1
kind: Pod
metadata: 
	nginx: nginx
spec:
	containers:
		-name: name
		  image: nginx
		resources:
			requests:
				memory: "4Gi"
				cpu: 2
		limits:%%to set the upper limit of resources%%
			memory: "8Gi"
			cpu: "4"
```

by default kubernetes doesnt have any resource request config so any node can take as many resources and suffucate others 

to define a config for every pod created we can set limitRange
limit-range-cpu.yaml
```yaml

apiVersion: v1
kind: LimitRange
metadata: 
	name: name
spec:
	limits:
		-  default:
			cpu: 500m%%limit%%
		   defaultrequest:
			   cpu: 500m%%request%%
		   max:
			   cpu: "1" %%limit%%
		   min:
			   cpu: 100m%%request%%
	       type: Container
```
we can also set resource  quota 
compute-quota.yml
```yaml
apiVersion: v1 
kind: ResourceQuota
metadata: 
	name: compute-quota
spec:
	hard:
		pods:  "10"
		requests.cpu: "4"
		requests.memory: 5Gi
		limits.cpu: "10"
		limits.memory: 10Gi
```
DemonSets
The demon set ensures that one copy of the pod is always present in all nodes in the cluster
if you want to deploy monitoring agent or log collector in every node
the ndemon set is perfect for it as it will deploy your monitoring pod in every node. You dont need to worry about adding it manually
kubeproxy is required on every node . The kubeproxy component can be deployed as a demon set in the cluster
demon set creation 
demonset.yaml
```yaml
apiVersion: apps/v1
kind: DemonSet
metadata:
	name: monitoring-daemon
spec: 
	selector:
		matchLabels:
			app: monitoring-agent
	template:
		metadata:
			labels:
				app: monitoring-agent
		spec:
			containers;
				- name: monitoring-agent
				  image: monitoring-agent
```
DemonSet uses NodeAffinity to set each pod in every node 
STATIC pods
static pods are  created directly by the kubelet wintout any interferenc e from the apiserver .static pods can be used to deploy the kubernetes control plane components itself both static pods and daemonsets pods are ignored by kube-scheduler
The yaml files of all the static pods are generally in /etc/kubernetes/manifests(this address  can be configured)
Multiple scheduler 
in kubernetes we can create our own custom scheduler ,when cretaing a pod or deployment we can instruct the kubernetes to habe the pod scheduled by a specific scheduler
in multiple scheduling the ymust have different name to be identified
scheduler config file 
scheduler.yaml
```yaml
apiVersion: kubeschduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
	- schedulername: my-scheduler
LeaderElection:
	leaderElect: true
	resourcenamespace: kube-system
	resourceName: kube-system
```
leader elect option is used when you have multiple copies of the scheduler in different master nodes if multipe scheduler are there on the node so only one of them can be active at a time
 to run kubeschuler of your own to need to download kube-scheduler bianary and the point its configuration to our path
 ==ExecStart=/usr/local/bin/kube-scheduler \\
	 --config=/etc/kubernetse/config/my-scheduler.yaml==

Using kubeadm we can deploy Additional scheduler as a Pod
(which is generally the right way)
scheduler-config.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
	name: my-scheduler
	namespace: kube-system
spec:
	containers:
		-  command:
			- kube-scheduler
			- --adress=127.0.0.1
			- kubeconfig=/etc/kubernetes/scheduler.conf
		image: kubeschduler.config.k8s.io/v1
		name: my-schceduler
```
once we have deployed custom scheduler then we need to configure the pod for the scheduler
```yaml
apiVersion: v1
kind: Pod 
metadata:
	 name: myapp-pod
	 labels:
		 app: myapp
		 type: front-end
spec: 
	containers:
		- name: nginx
		- image: nginx
	schedulername: my-scheduler
```
To identify which scheduler is scheduling a particuler pod
==kubectl get events -o wide==

Scheduler Profiles
When a pod is first scheduled it is sent to scheduling queue where it is arranged based on its priorityclass
spec:
	priorityClassName: high-priority
Then the pod goes to filter phase where nodes which cannot run the pod are filtered out
Next phase is Scoring phase where the score to each node is based on the free space.Node with higher score is picked up
Last phase is binding phase where pod is assigned to the node

we can also run multiple scheduler by naming diferent profiles for them eg:
```yaml
apiVersion: kubeschduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
	- schedulerName: my-scheduler
		plugins:
			score:
				disabled:
				- name: '*'
				enabled:
				- name: '*'
	- schedulerName: my-scheduler-2
	- schedulerName: my-scheduler-3
```
