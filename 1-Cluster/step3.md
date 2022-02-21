Let’s view the cluster details. We’ll do that by running *kubectl cluster-info*:

`kubectl cluster-info`{{execute}}

During this tutorial, we’ll be focusing on the command line for deploying and exploring our application.
To view the nodes in the cluster, run the *kubectl get nodes* command:

`kubectl get nodes`{{execute}}

This command shows all nodes that can be used to host our applications. Now we have only one node, and we can see that its status is ready (it is ready to accept applications for deployment). 


`kubectl get namespaces`{{execute}}

This command shows all the namespaces defined in the cluster. Namespaces in Kubernetes are the equivalent of conda environments in the sense that any changes made within a namespace has no effect outside of it.

`kubectl get pods --all-namespaces`{{execute}}

This will return all the current running pods of our cluster in all the namespaces.