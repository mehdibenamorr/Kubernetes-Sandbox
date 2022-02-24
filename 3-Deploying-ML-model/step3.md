In the third step, we will try to deploy the model by defining a multi-container pod.

### Model Deployment

We will define two containers where one of them is an initializing containers.

Let's start with metadata of the deployment which is similar to what we already saw in earlier steps.

<pre class="file" data-filename="deployment.yaml" data-target="replace">
apiVersion: apps/v1 #  for k8s versions before 1.9.0 use apps/v1beta2  and before 1.8.0 use extensions/v1beta1
kind: Deployment
metadata:
  # This name uniquely identifies the Deployment
  name: classifier-app
  namespace: penguin-classifier
  labels:
    app: penguin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: penguin
  template:
    metadata:
      labels:
        # Label is used as selector in the service.
        app: penguin
    spec:
</pre>

**Init Container**

The next part of the manifest file is where we define the init container which will download the data from minio and persist it in the volume shared by both of the containes. We used the [minio/mc](https://hub.docker.com/r/minio/mc) docker image for the minio client.

<pre class="file" data-filename="deployment.yaml" data-target="append">
      initContainers:
      - name: "download-files"
        image: minio/mc
        command: ["/bin/sh", "-c"]
        args:
        - /usr/bin/mc alias set share https://share.pads.fim.uni-passau.de $MINIO_ACCESSKEY $MINIO_SECRETKEY;
          /usr/bin/mc cp share/public/Infrastructure_Workshop/ML_Model/classifier/pytorch_model.pt /data/pytorch_model.pt;
          /usr/bin/mc cp share/public/Infrastructure_Workshop/ML_Model/classifier/requirements.txt /data/requirements.txt;
          /usr/bin/mc cp share/public/Infrastructure_Workshop/ML_Model/classifier/classes.txt /data/classes.txt;
          /usr/bin/mc cp share/public/Infrastructure_Workshop/ML_Model/classifier/deploy_model.py /data/deploy_model.py;
          /usr/bin/mc cp share/public/Infrastructure_Workshop/ML_Model/classifier/entrypoint.sh /data/entrypoint.sh;
          exit 0;
        volumeMounts:
            - mountPath: /data
              name: shared-data
        env:
        - name: MINIO_ACCESSKEY
          valueFrom:
            secretKeyRef:
              name: minio-client-secret
              key: accesskey
        - name: MINIO_SECRETKEY
          valueFrom:
            secretKeyRef:
              name: minio-client-secret
              key: secretkey
</pre>

**Main Container**

This container is simply a minimum debian system with miniconda3 that installs the required python libraries and start serving the Streamlit server.

We wrote a bash script that would be executed when the container is created. Similar to ENTRYPOINT in docker.

<pre class="file" data-filename="entrypoint.sh">
#! /bin/bash
pip install -r /data/requirements.txt
streamlit run /data/deploy_model.py 
</pre>

<pre class="file" data-filename="deployment.yaml" data-target="append">
      containers:
      - name: model-container
        image: registry.gitlab.pads.fim.uni-passau.de/padim/infrastruktur/it-infrastructure/model-app:latest
        image: continuumio/miniconda3
        imagePullPolicy: Always
        command: ["/bin/sh", "-c"]
        args: 
        - chmod +x /data/entrypoint.sh;
          ./data/entrypoint.sh;
        volumeMounts:
            - mountPath: /data
              name: shared-data
        ports:
          - name: server
            containerPort: 8501
      volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: model-pvc
</pre>

`kubectl apply -f deployment.yaml`{{execute}}

We next need to create a service (NodePort type) to expose the model on **http://NodeIP:NodePort**.

<pre class="file" data-filename="model-service.yaml" data-target="replace">
apiVersion: v1
kind: Service
metadata:
  name: classifier-service
  namespace: penguin-classifier
spec:
  type: NodePort
  ports:
    - port: 3300
      targetPort: 8501
  selector:
    app: penguin
</pre>


`kubectl apply -f model-service.yaml`{{execute}}

`export NODE_PORT=$(kubectl get services/classifier-service -n penguin-classifier -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT`{{execute}}

`curl -v $(minikube ip):$NODE_PORT`{{execute}}

And we get a response from the server. The Service is exposed.

Let's see what is running in the 

`kubectl get all -n penguin-classifier`{{execute}}