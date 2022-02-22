In the second step, we will prepare all dependencies needed for the two deployments.

### Persistent Volume Claims

First, we need to define two Persistent Volume Claims (PVCs) for the database and the label studio app.

A PVC is a request to configure and allocate a volume to persist the data in.

Copy the following text into pvc.yaml:

<pre class="file" data-filename="pvc.yaml" data-target="replace">
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pv-claim # This name uniquely identifies the PVC. Will be used in deployment below.
  namespace: label-studio
spec:
  # Read more about access modes here: http://kubernetes.io/docs/user-guide/persistent-volumes/#access-modes
  accessModes:
    - ReadWriteOnce
  storageClassName: openebs-local-rwo
  volumeMode: Filesystem
  resources:
    # This is the request for storage. Should be available in the cluster.
    requests:
      storage: 1Gi
</pre>

This would create the claim for the PostgreSQL.

`kubectl apply -f pvc.yaml`{{execute}}

We can have multiple resource definitions in the same manifest file by separating them with "---". 

### Environment variables and Config files
In Kubernetes, uploading files to your running Pod is not as straight forward as Docker where you can simply mount them using their local paths.

However, it's possible by using the **ConfigMap** resource in the data field.

For example, if we have a config json file we want mount in our deployment. Then the ConfigMap would be defined with,

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configfile
  namespace: example-namespace
  labels:
    app: APP_LABEL
data:
  config.json: |
    {
        "param_key" : "value"
    }
```

Back to our usecase, for both deployment we need to define some environment variables.

PostgreSQL configuration:

<pre class="file" data-filename="config.yaml" data-target="replace">
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: label-studio
  labels:
    app: postgres
data:
  POSTGRES_DB: postgresdb
  POSTGRES_USER: admin
  POSTGRES_PASSWORD: CrrBAXP7NzYz59de
</pre>

Label Studio configuration:

<pre class="file" data-filename="config.yaml" data-target="append">
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: labelstudio-config
  namespace: label-studio
  labels:
    app: labelstudio
data:
  DJANGO_DB: default
  POSTGRE_NAME: postgresdb
  POSTGRE_USER: admin
  POSTGRE_PASSWORD: CrrBAXP7NzYz59de
  POSTGRE_PORT: "5432"
  POSTGRE_HOST: postgres.label-studio.svc.cluster.local
  LABEL_STUDIO_HOST: http://localhost:3200
  LABEL_STUDIO_COPY_STATIC_DATA: "false"
</pre>

`kubectl apply -f config.yaml`{{execute}}

The **LABEL_STUDIO_HOST** needs to be changed in the step where we try to expose the app to the public internet with the same hostname used in the definition of the Nginx server.

``` yaml
LABEL_STUDIO_HOST = https://label-studio.infra-workshop.uni-passau.de
```

### Secrets
A better way to define config files or env variables in a more secure way (i.e Admin credentials,) is the use of the **Secret** resource.

For example, if we want to use a docker image from a private registry (i.e Gitlab registry) we create the secret by running

``` bash
echo -n 'super-secret-password' | base64
```

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
  namespace: example-namespace
data:
  .dockerconfigjson: Output of "base64 --input ~/.docker/config.json"
type: kubernetes.io/dockerconfigjson
```

In the next step we will deploy the PostgreSQL database.