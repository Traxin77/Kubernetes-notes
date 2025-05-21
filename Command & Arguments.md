Containers are not meant to host vm they are meant to host task like web server  or database once the task the complete it exits.
To define the process inside container we need commands.
In docker we can set entry command and its argument using ENTRYPOINT["sleep"] and CMD["5"] in kubernetes we can do the same my using method below:
pod.yml
```yaml
apiVersion: v1
Kind: pod
metadata:
	name: ubuntu
spec:
	containers:
		- name: ubuntu
		   image: ubuntu
	       command: ["sleep"]
	       args: ["10"]
```
