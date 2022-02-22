In this scenario you will learn how to expose Kubernetes applications outside the cluster.
The previously deployed Label Studio app is only available from within the cluster since we used the **ClusterIP** service type.

To expose the app publicly, we can use the other two types of services **NodePort** and **LoadBalancer**.

### NodePort

It exposes the service outside of the cluster by exposing it on each Node's IP at a static port (the NodePort in the range of 30000–32767). The service endpoint would then be available on **http://NodeIP:NodePort**.

<pre class="file" data-filename="labelstudio-service.yaml" data-target="replace">
apiVersion: v1
kind: Service
metadata:
  name: labelstudio-service
  namespace: label-studio
spec:
  type: NodePort
  ports:
    - port: 3200
      targetPort: 8080
      nodePort: 30001 # 30000-32767, Optional field
  selector:
    app: labelstudio
</pre>

To update the changes, simply re-apply this manifest file.

`kubectl apply -f labelstudio-service.yaml` {{execute}}

`curl $(minikube ip):$30001`{{execute}}

And we get a response from the server. The Service is exposed.

### LoadBalancer

LoadBalancer service is an extension of NodePort service. NodePort and ClusterIP Services, to which the external load balancer routes, are automatically created. 

Traffic from the external load balancer is directed at the backend Pods. The cloud provider decides how it is load balanced. Traffic from the external load balancer is directed at the backend Pods. The cloud provider decides how it is load balanced

**Use this service is when you are using a cloud provider to host your Kubernetes cluster.**

### Ingress
While the aforementioned methods can expose your service, they are not the most secure/reliable way to do it.

In a Kubernetes cluster, the common practice is have an ingress controller running which is responsible for fulfilling the **Ingress**, usually with a load balancer.

**Ingress** exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.

<pre class="file" data-filename="ingress.yaml" data-target="replace">
# Ingress configuration
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: labelstudio-ingress
  namespace: label-studio
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt # For automatic certificate issuance and revocation.
    ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  ingressClassName: nginx
  rules:
  - host: label-studio.infra-workshop.uni-passau.de # this has to be the same as LABEL_STUDIO_HOST
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: labelstudio-service
              port:
                number: 3200 # Same as port in the service definition
  tls: # < placing a host in the TLS config will indicate a certificate should be created
  - hosts:
    - label-studio.pads.fim.uni-passau.de
    secretName: labelstudio-cert # This certificate will be created and stored to be used for encrypting HTTP trafic.
</pre>

**Ingress rules**

Each HTTP rule contains the following information:

    An optional host. In this example, `label-studio.infra-workshop.uni-passau.de` is specified, so the rule applies only to that host.
    A list of paths (for example, /testpath), each of which has an associated backend defined with a service.name and a service.port.name or service.port.number. Both the host and path must match the content of an incoming request before the load balancer directs traffic to the referenced Service.
    A backend is a combination of Service and port names as described in the Service doc or a custom resource backend by way of a CRD. HTTP (and HTTPS) requests to the Ingress that matches the host and path of the rule are sent to the listed backend.
