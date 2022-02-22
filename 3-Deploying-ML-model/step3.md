In the third step, we will see how a **Deployment** resource is defined and subsequently deploy both the DB and Label Studio.

### PostgreSQL Deployment

The resource **Deployment** is often more used than **Pod** due to the limited features available for the latter. In most cases, scaling, updating and high availability are needed in most Software-as-a-Service applications. Therefore, we usually go with **Deployment** with replicas or in some special cases **StatefulSet**.

Let's start with metadata of the deployment which is similar to what we already saw in earlier steps.

<pre class="file" data-filename="postgresql.yaml" data-target="replace">
apiVersion: apps/v1 #For k8s versions before 1.9.0 use apps/v1beta2 and before 1.8.0 use extensions/v1beta1
kind: Deployment
metadata:
  name: postgres-db
  namespace: label-studio
  labels:
    app: postgresql # Label is used as selector in the service.
</pre>

The next part of the manifest file is where we define the number of replicas, selector, containers, and volumes under **spec** or specification:

<pre class="file" data-filename="postgresql.yaml" data-target="append">
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql-container
          image: postgres:11.5
          imagePullPolicy: "IfNotPresent"
          ports:
            - containerPort: 5432 # Has to be the same port defined in the ConfigMap
          envFrom:
            - configMapRef:
                name: postgres-config
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-volume
      # Refer to the PVC created earlier
      volumes:
        - name: postgres-volume
          persistentVolumeClaim:
            claimName: postgres-pv-claim
</pre>

`kubectl apply -f postgresql.yaml`{{execute}}

We next need to create a service (ClusterIP type) to provide reliable networking (Cluster-IP Addr, Port) for the database within the cluster.

A service can be defined as follows:

<pre class="file" data-filename="postgresql-service.yaml" data-target="replace">
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: label-studio
spec:
  ports:
    - port: 5432 # This would be the same as the env variable POSTGRE_PORT.
      targetPort: 5432 #This has to be the same port of the container.
  selector:
   app: postgresql
</pre>

`kubectl apply -f postgresql-service.yaml`{{execute}}

### Label Studio Deployment

Similar to the database, deploying Label Studio 

<pre class="file" data-filename="labelstudio.yaml" data-target="replace">
apiVersion: apps/v1 #  for k8s versions before 1.9.0 use apps/v1beta2  and before 1.8.0 use extensions/v1beta1
kind: Deployment
metadata:
  name: labelstudio-app
  namespace: label-studio
  labels:
    app: labelstudio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: labelstudio
  template:
    metadata:
      labels:
        # Label is used as selector in the service.
        app: labelstudio
    spec:
      containers:
      - name: labelstudio-container
        image: heartexlabs/label-studio:latest
        command: ["./deploy/docker-entrypoint.sh"]
        args: ["label-studio"]
        envFrom:
          - configMapRef:
              name: labelstudio-config
        ports:
          - name: server
            containerPort: 8080
</pre>

`kubectl apply -f labelstudio.yaml`{{execute}}

Create a service to provide the networking:

<pre class="file" data-filename="labelstudio-service.yaml" data-target="replace">
apiVersion: v1
kind: Service
metadata:
  name: labelstudio-service
  namespace: label-studio
spec:
  ports:
    - port: 3200
      targetPort: 8080
  selector:
    app: labelstudio
</pre>

`kubectl apply -f labelstudio-service.yaml`{{execute}}

### Debugging, Logging, ...

Let’s verify that the application we deployed. We’ll use the `kubectl get` command and look for existing:

Deployments with `get deployments` command:

`kubectl get deployments -n label-studio`{{execute}}

Pods wtih `get pods` command:

`kubectl get pods -n label-studio`{{execute}}

Services with `get services`{{execute}}

`kubectl get services -n label-studio`{{execute}}

Pods that are running inside Kubernetes are running on a private, isolated network.
By default they are visible from other pods and services within the same kubernetes cluster, but not outside that network.
When we use `kubectl`, we're interacting through an API endpoint to communicate with our application.

We will cover other options on how to expose your application outside the kubernetes cluster in the next tutorial. 

Next, to view what containers are inside the running pods and what images are used to build those containers we run the `describe pods` command:

`kubectl describe pods`{{execute}}

We see here details about the Pod’s container: IP address, the ports used and a list of events related to the lifecycle of the Pod.

*Note: the describe command can be used to get detailed information about most of the kubernetes primitives: node, pods, deployments. The describe output is designed to be human readable, not to be scripted against.*

`kubectl describe deployment.apps/postgres -n label-studio`{{execute}}

We can retrieve logs from a pod using the `kubectl logs` command:

`kubectl logs $POD_NAME`

*Note: We don’t need to specify the container name, because we only have one container inside the pod.*

In the next tutorial, we will see how to expose this app to the internet.

