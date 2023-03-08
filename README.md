## Introduction to Istio
Istio is an open-source service mesh that provides a uniform way to connect, manage, and secure microservices. It's designed to help you manage the complexities of a large and complex distributed system, by providing features such as traffic management, security, and observability.

With Istio, you can control and manage the flow of traffic between microservices, including load balancing, traffic routing, and fault injection. Istio also provides strong authentication, authorization, and encryption for microservices traffic, helping to secure your system against attacks and data breaches.

In addition, Istio can collect telemetry data and generate metrics, logs, and traces to provide visibility into your microservices system. This helps you to monitor and troubleshoot your system, and make data-driven decisions about how to improve performance and reliability.

Istio achieves these features by injecting a sidecar proxy into each pod in your Kubernetes cluster. This proxy intercepts all network traffic and routes it through the Istio control plane, which manages traffic flows and applies policies.

Istio is designed to be flexible and extensible, with a range of configuration options and integrations with other tools such as Prometheus, Grafana, and Jaeger. It can be used to manage microservices across multiple clusters, clouds, and environments, making it a powerful tool for building and managing distributed systems.

## Installing istio on kubernetes cluster:

 Note:
 ```
 If you face error in installating egress gateway or ingress gateway try with -1 versions or restart master node of cluster.
 ```

### steps:

 - setup the kubernetes cluster
 - download the tar file of your version [release](https://github.com/istio/istio/releases)
 ```
 wget https://github.com/istio/istio/releases/download/1.17.1/istio-1.17.1-linux-amd64.tar.gz
 ```
 - untar it
 ```
 tar -xvzf istio-1.17.1-linux-amd64.tar.gz
 ```
  - add bin directory of istio to the path
  ```
   export PATH=$PATH:/home/nmaster-0/istio/istio-1.17.1/bin
  ```
  - install istio using any of the provided profiles,Here i am using demo profile
  ```
  istioctl install --set profile=demo -y
  ```
  
  
###  Uninstall istio :
- purge it
```
istioctl x uninstall --purge
```
- delete istio-system namespace
```
kubectl delete ns istio-system
```
 
### install one microservices application to test istio

 - enable label istio-injection=enabled in default namespace
 ```
 kubectl label namespace default istio-injection=enabled
 ```
 - create a sample manifest file for application(here iam using googlecloud boutiq app) [link](https://github.com/GoogleCloudPlatform/microservices-demo/blob/main/release/kubernetes-manifests.yaml)
 ```
 kubectl apply -f kuberenetes-manifests.yaml
 ```
 - To delete that label 
 ```
 kubectl label namespace default istio-injection-
 ```
 
### Install istio addons to enable monitoring
 - All addons will present in istio-1.17.1/samples/addons/
 - I am deploying all the addons but you can install only required addons using respective manifests
 ```
 kubectl apply -f istio-1.17.1/samples/addons/
 ```
 - check all the pods in istio-system namespace
```
  kubectl get pods -n istio-system
```
 - check their respective services up and running
```
 kubectl get svc -n istio-system
```
 - if you can browse using your master machine you can use portforward to access kiali dashboard else you have to use ingress or nodeport

##### Note:
To enable monitoring using istio in every deployment or svc there should be a label with key app.(app:value)

- you can learn about 
	- virtual services (internal communication)
	- Destination rules
	- Gateways (external communication)
	- service entries
	- Sidecars
- network resilency
  - timeouts
  - retries
  - circuit breakers
  - fault Injection
  
- reference: https://istio.io/latest/docs/concepts/traffic-management/

### Metallb setup

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
kubectl apply -f metallb-config.yaml
kubectl apply -f metallb-l2advert.yaml
```
 - file metallb-config.yaml contents:
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
```
 - file metallb-l2advert.yaml contents:
```
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

### Setting up application using Virtual service and Gateway for testing canary deployment

 - we have to create a service ,two deployments(two versions),virtual service,gateway,destination rules.
 - I am using two versions of a image fgrom my docker hub which i build from digitaloceans reference repos
   1. git clone https://github.com/do-community/nodejs-image-demo.git istio_project (naveensmily79/node-demo)
   2. git clone https://github.com/do-community/nodejs-canary-app.git node_image (naveensmily79/node-demo-v2)
   
 - I will create two files node-app.yaml & node-istio.yaml
 contents of node-app.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: nodejs
  labels:
    app: nodejs
spec:
  selector:
    app: nodejs
  ports:
  - name: http
    port: 8080

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-v1
  labels:
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs
  template:
    metadata:
      labels:
        app: nodejs
        version: v1
    spec:
      containers:
      - name: nodejs
        image: naveensmily79/node-demo
        ports:
        - containerPort: 8080

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-v2
  labels:
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs
  template:
    metadata:
      labels:
        app: nodejs
        version: v2
    spec:
      containers:
      - name: nodejs
        image: naveensmily79/node-demo-v2
        ports:
        - containerPort: 8080
```
 - contents of node-istio.yaml
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: nodejs-gateway
spec:
  selector:
    istio: ingressgateway 
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"

---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nodejs
spec:
  hosts:
  - "*"
  gateways:
  - nodejs-gateway
  http:
  - route:
    - destination:
        host: nodejs
        subset: v1
      weight: 80
    - destination:
        host: nodejs
        subset: v2
      weight: 20

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: nodejs
spec:
  host: nodejs
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
	  
```
 
- apply both yamls and test the service with istio ingress IP address

Note:
```
Reference1: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-istio-with-kubernetes
Reference2: https://www.digitalocean.com/community/tutorials/how-to-do-canary-deployments-with-istio-and-kubernetes
```


### Cleaning

```
kubectl delete -f  node-istio.yaml
kubectl delete -f node-app.yaml
kubectl delete -f kubernetes-manifests.yaml
kubectl delete -f istio-1.17.1/samples/addons/
istioctl x uninstall --purge (if istioctl is not in PATH add it)
export PATH=$PATH:/home/nmaster-0/istio/istio-1.17.1/bin
kubectl delete ns istio-system
rm -rf istio-1.17.1 istio-1.17.1-linux-amd64.tar.gz
```


### Istio setup using helm

 - install helm on the cluster
 ```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh 
 ```
 Note:
  Reference for installing helm: [link](https://helm.sh/docs/intro/install/)
  
  - Installing the istio using helm
  - add helm repos
```
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```
  
This section describes the procedure to install Istio using Helm. The general syntax for helm installation is:
```
helm install <release> <chart> --namespace <namespace> --create-namespace [--set <other_parameters>]
```
The variables specified in the command are as follows:

- 'chart' A path to a packaged chart, a path to an unpacked chart directory or a URL.
- 'release' A name to identify and manage the Helm chart once installed.
- 'namespace' The namespace in which the chart is to be installed.

- Install the Istio base chart which contains cluster-wide Custom Resource Definitions (CRDs) which must be installed prior to the deployment of the Istio control plane:
```
helm install istio-base istio/base -n istio-system --create-namespace
```
To learn more about the release, try:
```
  helm status istio-base -n istio-system
  helm get all istio-base -n istio-system
  helm ls -n istio-system
```

- Install the Istio discovery chart which deploys the istiod service:
```
helm install istiod istio/istiod -n istio-system --wait
helm ls -n istio-system
helm status istiod -n istio-system
```
- Check istiod service is successfully installed and its pods are running:
```
 kubectl get deployments -n istio-system --output wide
```
- Install an ingress gateway:
```
helm install istio-ingress istio/gateway -n istio-system --wait
```

- To list the available repos 
```
helm repo list
```
- to list available charts in a repo 
```
helm search repo istio
```
- to know the components of a chart 
```
 helm template istio/cni
```

Note:
```
Reference: https://istio.io/latest/docs/setup/install/helm/
```

 



 


