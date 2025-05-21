If we need to monitor pods deployed in a cluster kubernetes doesnt provide in built monitoring system. 3rd party softwares like metricsserver, prometheus and elastic stack is used
Metric server is uses cAdvisor agent which is present in kubelet to fetch metrics from the pods and to the metrics server in master node
you need to git clone it and then deploy it to run it

Logging
we can simulate randaom event using event smulater to test logging
view logs in kubernetes
==kubectl logs -f event-simulator-pod==
if the pod consist of multiple containers specify the containers too
==kubectl logs -f event-simulator-pod event-simulator1==