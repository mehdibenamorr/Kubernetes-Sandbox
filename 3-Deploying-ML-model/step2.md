In the second step, we will prepare all dependencies needed for the deployment.

### Namespace

<pre class="file" data-filename="namespace.yaml" data-target="replace">
apiVersion: v1
kind: Namespace
metadata:
  name: penguin-classifier
</pre>

This would create the namespace for the model deployment.

`kubectl apply -f namespace.yaml`{{execute}}

### Persistent Volume Claim

<pre class="file" data-filename="pvc.yaml" data-target="replace">
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-pvc
  namespace: penguin-classifier
spec:
  # Read more about access modes here: http://kubernetes.io/docs/user-guide/persistent-volumes/#access-modes
  accessModes:
    - ReadWriteOnce
  #storageClassName: ""
  volumeMode: Filesystem
  resources:
    # This is the request for storage. Should be available in the cluster.
    requests:
      storage: 1Gi
</pre>

This would create the claim for the penguin classifier app.

`kubectl apply -f pvc.yaml`{{execute}}


### Secrets
A better way to define config files or env variables in a more secure way (i.e Admin credentials,) is the use of the **Secret** resource.

In our case, we want to use a pass minio service account credentials to the deployment in order to download the files within the pod.

``` bash
echo -n 'super-secret-password' | base64
```

<pre class="file" data-filename="secret.yaml" data-target="replace">
## Secret to be used for minio client
apiVersion: v1
kind: Secret
metadata:
  namespace: penguin-classifier
  name: minio-client-secret
type: Opaque
data:
  accesskey: "MUlZQTBENUZYTTlOUENRUThCQTE="
  secretkey: "ZXRnK1U0MUh5bFM1dnRJQUpQK0kyUHlubWhMNUJrSENQc3kyN3Racg===="
</pre>

`kubectl apply -f secret.yaml`{{execute}}

After creating the secret, we will see how we can use those credentials in the deployment manifest in the next step.