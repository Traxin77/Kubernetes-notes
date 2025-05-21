One of the prerequisites of networking in kubernetes is that the pods should be able to connect to any pod in any node without any additional settings. 
kubernetes is by default configured with all allow rule that allow traffic from any pod to any other pod or services
If we have some pod name web-app and we dont want it to communicate with internal database we would need to setup 
network policy
Network policy is another object in kubernetes , we can link network policy to any pod to apply rule
eg of a networkpolicy object for  a db
Ingress= Incoming traffic
Egress= outgoing traffic(naming is only one directional response datastream is not named)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: db-policy
spec:
	podSelector:
		matchLabels:
			role: db
	policyTypes:
		- Ingress
	ingress:
		- from:
			- podSelector:
				matchlabels:
					name: api-pod
				namespaceSelector:
					matchLabels:
						name: prod
			- ipBlock:%%external server not deployed in our cluster%%
				cidr: 192.168.5.10/32
			ports:
				- protocol: TCP
				   port: 3306
	egress:%%Outgoing traffic%%
		- to:
			- ipBlock:
				cidr: 192.168.5.10/32
			   ports:
				- protocol: TCP
				   port: 80
```