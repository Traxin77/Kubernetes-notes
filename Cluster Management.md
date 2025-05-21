when we need to take down any node during maintenance we use drain , cordon and uncordon commands
- drain is used to gracefully remove the node from service and evicting all pods running on that node 
(drain will be stpped if the pod doesnt belong to replicaset)
- cordan is used t o disable scheduling of new pods on  that node
- uncordon enables the scgeduling of new pods previously drained
(when we uncordon the node the pods wont automatically come back to that node new pods will be assigned to that node)

 cluster upgrading
 In kubernetes there are 3 ways of upgrading cluster:
 strategy 1: taking down all the nodes at the same time and launching the upgraded nodes after downtime
 startegy 2: Taking down nodes one at a time to upgrade them 
 strategy 3: adding new version nodes in the cluster first shifting the older version node's pods to it and then taking that node down
 
 steps for Updating cluster using kubeadm

==apt update
apt-cache madison kubeadm
apt-get install kubeadm=1.32.0-1.1
kubeadm upgrade plan v1.32.0
kubeadm upgrade apply v1.32.0
systemctl daemon-reload
systemctl restart kubelet==

Backup 
In kubernetes we need importnat info about cluster to be backed up like:
resource config
resource config like pod.yaml are important to redeploy the application used their yaml files to back up the config files we can simply upload them to a github repo or save their copy in the apiserver
to get the resource config files
==kubctl get all --all-namespaces -o yaml > all.yaml==
ETCD
stores info about state of the cluster
to save the ETCD backup we can use snapshot
==ETCDCTL_API=3 etcdctl \
	snapshot save snapshot.db==
To restore from the snapshot
1. ==Service kube-apiserver stop==
2. ==ETCDCTL_API=3 etcdctl \
		snapshot restore snapshot.db \
		--data-dir /var/lib/etcd-from-backup==
then we configure the --data-dir /var/lib/etcd-from-backup in the etcd.service
then
==systemctl daemon-reload
systemctl etcd restart
service kube-apiserver start==
