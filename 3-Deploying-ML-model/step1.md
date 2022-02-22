Let’s deploy our first app Label Studio on Kubernetes. Label Studio is an open source annotation tool written in Python with ReactJS for its frontend. 

It uses a Database in the backend to store project data and configuration information. 

This would a good first example on how to deploy an application with different components.

First step as always would be to create a new namespace. Copy this text into a yaml file.

<pre class="file" data-filename="namespace.yaml" data-target="replace">
apiVersion: v1
kind: Namespace
metadata:
  name: label-studio
</pre>

`kubectl apply -f namespace.yaml`{{execute}}

`kubectl get namespaces`{{execute}} 

To check that the namespace is created.
