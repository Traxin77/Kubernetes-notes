 To secure the kubernetes environment we need to secure its central point which is kube-apiserver there we have to secure it by 
 Authentication:

 kubernetes doesn't manage the user accounts natively it relies on external source file with user details . In case of service accoints which are used for 3rd party integration kubernetes can manage them using API
 ==kubectl create svcacc sv1==
 All the user access is manages by api-server 
 we can perform secure authentication using:
 - username and password: we create a csv file containing user:pass and use it in kube-apiserver.service --basic-auth-file=user.csv
   then to authenticate we would need to mention it in curl command
 - username and tokens: in this case we can have a static token file
   --token-auth-file=user.csv we can add token in Authorization bearer section in header of the request
 - certificates:  [[certificate]]

KUBECONFIG
when we need to set some additional configuration again and again we can save them in a kubeconfig file like $HOME\.kube\config
==kubectl get pods --kubeconfig KubeConfig==

KubeConfig 
--server my-kube-playground:6443
--client-key admin.key
--client-certificate admin.crt
--certificate-authority ca.crt
A KubeConfig file consist of 3 component:
Cluster: it consist of all the cluster like development production and testing  
Context:It defines which user will access which cluster eg Admin@Development. this allow us to not define user certificate and server address everytime we use kubectl
Users: It consist of all the different user account with different privileges
kubeconfig.yaml
```yaml
apiVersion: v1
kind: Config
clusters:
-name: my-kube-playground
	cluster: 
		certificate-authority: ca.crt
		server: https://my-kube-playground:6443
contexts:
	- name: my-kube-admin@my-kube-playground
	   context:
		   cluster: my-kube-playground
		   user: my-kube-admin
users:
	- name: my-kube-admin
	   user:
		   client-certificate: admin.crt
		   client-key: admin.key
```
to view current config file
==kubectl config view==
view current context
==kubectl config current-context --kubeconfig my-kube-config==
to set other context as current context
==kubectl config --kubeconfig=my-kube-config use-context research==

API groups
when we interact with the cluster using kubectl commands we are just use kube-apiserver to run the command we can also access the cluster using curl request
curl http://localhost:6443/versions
The API are categorized into 2 
1. core group:Its where all the core functionality like namespaces , pods ,replication,services etc are present
2. named group: It is more organized and has the newer features it has groups under it like apps , networking, storage authentication etc 
when we curl to the api sometime we might require to mention the certificate and key we can set up a kubectl proxy which is a http proxy used to interact to apiserver it will directly import the credential from kubeconfig file 

AUTHORIZATION:
when we want to a cluster to have many different user account but not all of them to have same privileges we use authorization. Authorization can also help you with a user have access to its namespace alone 
types of authorization:
Node: when a kube-server is accessed by a kubelet the kubeserver have access to node information of the kubelet or user . It consist of nodeauthorizer and if the node belong to the right group (system-node) it is granted the privilege

ABAC: In attribute based access control we map  the user to the actions he can perform in a json file and save it in a server. we would need to create this json file for each user or group 

RBAC: rather than defining individual user there privileges we define roles with certain level of privileges and map the user to those role.
provide more standard approach
AlwaysAllow & AlwaysDeny

webhook: we can outsource the authorization mechanism like open policy agent . The kube api webhook sends request to the agent and the response decides we should give access or not
we can set auth mode in apiserver
--authorization-mode= AlwaysAllow
when multiple modes are set it authorises through all of them in a sequence

RBAC
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: 
	name: developer
rules:
	- apiGroups: [""]
	  resources: ["pods"]
	   verbs: ["list","get","create","update","delete "]
	  resourceNames: ["blue", "orange"]%%this will allow you to give access to only certain pods %%
```
now to linke the user to role we need to creatae another object
devuser.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: 
	name: userrole
subject:%%user details%%
	- kind: User
	  name: dev-user
	  apiGroups: rbac.authorization.k8s.io
roleRef:%%role detail%%
	kind: Role
	name: developer
	apiGroup:  rbac.authorization.k8s.io
```
some commands
==kubectl get role==
==kubectl get rolebinding==

as a user if you want to creack your access 
==kubectl auth can-i delete nodes==

if you want to run a a test to check other user privileges
==kubectl auth can-i cretae pods --as dev-user==
this would only give answer is yes & no

Cluster Roles
cluster scoped resources are those where you dont specify namespaces and you create them like nodes, persistent volumes , persistent cluster roles etc to view the non namespaced resources 
==kubectl --api-resources namespaced=true==
To authorize a user to cluster wide resources we use clusterroles and clusterbindings

Clusterroles
These are roles defined for cluster scoped resources 
clusterrole-definition.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: 
	name: cluster-administrator
rules:
	- apiGroups: [""]
	  resources: ["nodes"]
	   verbs: ["list","get","create","delete "]
now to map the user to role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: 
	name: clusterrole-binding
subjects:
	- kind: User
	  name: cluster-admin
	  apiGroup: rbac.authorization.k8s.io
roleRef:
	kind: ClusterRole
	name: cluster-administrator
	apiGroup:  rbac.authorization.k8s.io
```
With cluster role when you authorize the user to access the pods the user gets access to all pods across the cluster

Service Account
account used by application to interact with the cluster
 to create service account
 ==kubectl create serviceaccount name==
 when account is created a token is also created which should be used to authenticating to the kubernetes api
 token is stored as a secret object
 to view token
 ==kubectl describe secret token name==
 whenever a pod is created a default service account and token is mount to its volume 
 In the newer version of kuberenetes(1.22) the tokens can be requested from tokenrequest api which can generate expiry defined token for the service

IMAGE Security
When we create pods in the cluster we should make sure to use only verified docker hub images
image :   docker.io/ library /nginx
         registry    user    image/repo
 like docker.io many of the kubernetes images are in gcr.io managed by google 
 if we are using a image from a private repository we need to mention its full address and for authrntication we need to store the values in a secret and provide that secret in imagePullSecrets paramerter
 Security Context 
 In kubernetes we can set securityContext parameter to set a rule either for one container or all the container inside the pod
 secure.yaml
 ```yaml
 apiversion: v1
 kind: Pod
 metadata:
	 name: web
 spec:
	cotainers:
		- name: ubuntu
		   image: ubuntu
		   command: ["sleep","3600"]
		   securityContext:
			   runAsUser: 1000
			   capabilities:
				   add: ["MAC_ADMIN"]
```
[[NETWORK SECURITY]]

