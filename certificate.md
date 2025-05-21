certificate is used to guarantee trust between 2 parties during transaction it ensures data encryption while transaction.
we use asymmetric encryption with a pair of public and private keys to secure ssh connectivity to the servers.
Public Key Infrastructure (PKI) works by establishing trust through digital certificates and asymmetric encryption. A trusted entity, called the **Certificate Authority (CA)**, issues digital certificates that bind a user’s identity to their **public key**. When a user requests a certificate, a **Registration Authority (RA)** verifies their identity before forwarding the request to the CA. The CA signs the certificate with its **private key**, making it verifiable by others using the CA’s **public key**. When a secure communication session is initiated, the recipient verifies the sender’s certificate and uses their public key for encryption, while the sender uses their private key for decryption. PKI also supports certificate revocation via **CRLs** or **OCSP**. This system ensures **authentication, encryption, integrity, and non-repudiation** in digital communications, making it essential for secure transactions, HTTPS, email encryption, and identity verification.

Securing cluster with TLS cert

public key              private key
sever.crt                  server.key
server.pem              server-key.pem
In a kubernetes cluster the comm between master and its worker should be encrypted to encrypt the all comm there are 2 requirements Server certificate & Client certificate
server in kubernetes
kube-apiserver: API serve rexposes http service that external user and other component use to manage kubernetes cluster . To secure comm we generate apiserver.crt and apiserver.key

ETCDserver: it stores info of cluster for it we will generate etcdserver.crt and etcdserver.key

KUBELETserver: present on worker node we will use kubelet.crt and kubelet.key
Clients like admin will also have their own pair admin.crt and admin.key
scheduler.crt and scheduler.key will be used by  kube-scheduler
Kube-controller also act as client which will require controller.crt and controller.key
last client is KUBE-proxy which will have roxy.crt and proxy.key
It sums up all certificates used in a cluster

Generating certificate
generating ca.cert
generate keys      ==openssl genrsa -out ca.key 2048==
certificate signing
request                     ==openssl req -new -kwy ca.key -subh "/CN= kube-CA" -out ca.csr==

sign certificate                  ==openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt==
when we setup a TLS  encryption in the comm we can use a easy method of using kubeadm which can on its own generate and configure the certificates required for the cluster

In an environment setup by kubeadm we can view the certificate used in /etc/kubernetes/manifests/kube-apiserver.yaml 
there we can get the location of each certificate
we can use 
==openssl x509 -in /etc/kubernetes/pki/apiserver.cert -text -nout==
to view the decoded certificate
if error occur due to improper setup of certificate we can view logs by
==kubectl logs master-etcd==
if the error occurs in kube-apiserver then we have to go one level down and use
==docker ps -a==
==docker logs containerid==

Certificate API in kubernetes
With the certificate api we can send a CertificateSigningRequest directly to kubernetes through API call .
instead of manually signing the certificate on the master node the admin can  create kubernetes object  CertificateSigningRequest .
Once object is created all certificates and request can be seen by admin using kubectl commands

In the kube-apiserver all the certificate related operations are carried out by controller manager  it has controllers in it called CSR-APPROVING and CSR-SIGNING which are resonsible for carrying out these task
steps to create CSR object for new user
we need user .csr file to create the yaml file
1.  cat new.csr | base64 -w 0
2. ```yaml
   apiVersion: certificates.k8s.io/v1 
	kind: CertificateSigningRequest 
	metadata: 
		name: akshay spec: 
	groups: 
		system:authenticated 
	request: <Paste the base64 encoded value of the CSR file> signerName: kubernetes.io/kube-apiserver-client 
	usages: 
		- client auth```
to approve csr request
==kubectl certificate approve csrname==
if you want to deny
==kubectl certificate deny csrname==

