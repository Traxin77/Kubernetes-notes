kubernetes doesn't visualize the whole app it only understands different objects which are interconnected to each other like pods, services , secrets etc . helm is like a package manager for the app it looks at the objects as a group for a bigger package. If we want to make any changes we don't tell it what object it need to touch we just tell it the app name like wordpress and it manages all the changes in different object automatically
some helm commands :
==helm install/uninstall wordpress
helm upgrade release-name wordpress --version 2.0.1
helm rolback wordpress
helm list
helm search hub/repo wordpress==
helm 3 consist a intelligent feature of rollback where it compares the previous and current state of the chart and also the live state of the objects and if we updated any object using kubectl it can detect that and can perform rollback.
Components:
Charts: it is the collection of file which consist of all the information helm needs to know about the cluster. 

Release: When a chart is applied to the cluster a release is created each release consist the different revisions of the app
we can create multiple releases of the same chart 
==helm install release-name bitnami/wordpress==
when we create a new release it assigns it a default values.yaml file we can change it or some of its values by passing our own file 
==helm --values custom.yaml my-release bitnami/wordpress==
we can make independent changes in different releases even if they belong to same chart to check the release history
==helm history release-name==
Secret: helm  uses kubernetes secrets to save the metadata so it can be accessible by different users 
