Solution Requirements
---------------------

* NGINX as Kubernetes Ingress
* Automatic Let's Encrypt TLS certificate renewals 
* F5 Big-IP as front-end load-balancer to the Kubernetes cluster
* F5 Big-IP configured with Kubernetes manifests via Container Ingress Services


Deploy NGINX Ingress and Cert Manager
-------------------------------------

The solution for using Cert-Manager to automatically renew Let's Encrypt TLS certificates on an NGINX Ingress is well documented here:

https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes

The solution uses the Kubernetes _ingress_ resource to configure NGINX ingress.

The deployment will be called _nginx-ingress_ in the _nginx-ingress_ namespace.


Create NGINX Ingress NodePort services for HTTPS (443) and HTTP (80)
--------------------------------------------------------------------

Create two _nodeport_ services in the _nginx-ingress_ namespace:

* nginx-ingress-443
* nginx-ingress-80

Example _nginx-ingress-nodeport.yaml_:
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-443
  namespace: nginx-ingress
spec:
  type: NodePort 
  ports:
  - port: 443
    targetPort: 443
    name: nginx-ingress-443
  selector:
    app: nginx-ingress
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-80
  namespace: nginx-ingress
spec:
  type: NodePort 
  ports:
  - port: 80
    targetPort: 80
    name: nginx-ingress-443
  selector:
    app: nginx-ingress
```
The **spec.selector.app = nginx-ingress** label will map these services to the **nginx-ingress** deployment already running.

Deploy F5 Container Ingress Service
-----------------------------------

We'll have to make some changes to the current CIS deployment args and redeploy.
 
**custom-resource-mode: true**
CIS will monitor the kube-api and watch for changes to "TransportServer" Custom Resource Definitions (CRDs). The TransportServer CRDs will create a Virtual Servers on the Big-IP. All other resource types will be ignored (ConfigMaps with AS3 or ingress).
 
**pool-member-type: nodeport**
CIS will monitor the kube-api and dynamically discover and populate the pool members on the Big-IP with Kubernets worker nodes and high port (3XXXX) exposing the NGINX Ingress service.

Example _cis_deployment.yaml_:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-bigip-ctlr-deployment
  namespace: kube-system
spec:
# DO NOT INCREASE REPLICA COUNT
  replicas: 1
  selector:
    matchLabels:
      app: k8s-bigip-ctlr-deployment
  template:
    metadata:
      labels:
        app: k8s-bigip-ctlr-deployment
    spec:
      # Name of the Service Account bound to a Cluster Role with the required
      # permissions
      containers:
        - name: k8s-bigip-ctlr
          image: "f5networks/k8s-bigip-ctlr:latest"
          env:
            - name: BIGIP_USERNAME
              valueFrom:
                secretKeyRef:
                # Replace with the name of the Secret containing your login
                # credentials
                  name: bigip-login
                  key: username
            - name: BIGIP_PASSWORD
              valueFrom:
                secretKeyRef:
                # Replace with the name of the Secret containing your login
                # credentials
                  name: bigip-login
                  key: password
          command: ["/app/bin/k8s-bigip-ctlr"]
          args: [
            # See the k8s-bigip-ctlr documentation for information about
            # all config options
            # https://clouddocs.f5.com/containers/latest/
            "--bigip-username=$(BIGIP_USERNAME)",
            "--bigip-password=$(BIGIP_PASSWORD)",
            "--bigip-url=https://x.x.x.x:443",
            "--bigip-partition=k8s-bigip-ctlr",
            "--insecure",
            "--pool-member-type=nodeport",
            "--custom-resource-mode=true"
            ]
      serviceAccountName: bigip-ctlr
```
			
Create TransportServer CRDs
---------------------------

Create two TransportServer CRDs in the nginx-ingress namespace:

* transport-server-80
* transport-server-443 

The TransportServer CRDs will create the F5 Big-IP Virtual Servers. For service discovery to work the name of the nginx-ingress nodeport services we created previously must match the spec.pool.service in the TransportServer manifest: 

_spec.pool.service = nginx-ingress-443_

Example _crd-transport-server.yaml_:
```
apiVersion: "cis.f5.com/v1"
kind: TransportServer
metadata:
   name: transport-server-443
   namespace: nginx-ingress
   labels:
     f5cr: "true"
spec:
  virtualServerName: "vs1_443"
  virtualServerAddress: "10.150.0.34"
  virtualServerPort: 443
  mode: standard
  snat: auto
  pool:
    service: nginx-ingress-443
    servicePort: 443
    monitor:
      type: tcp
      interval: 10
---
apiVersion: "cis.f5.com/v1"
kind: TransportServer
metadata:
   name: transport-server-80
   namespace: nginx-ingress
   labels:
     f5cr: "true"
spec:
  virtualServerName: "vs1_80"
  virtualServerAddress: "10.150.0.34"
  virtualServerPort: 80
  mode: standard
  snat: auto
  pool:
    service: nginx-ingress-80
    servicePort: 80
    monitor:
      type: tcp
      interval: 10
```