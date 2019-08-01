# Introduction

In the following you can find a short introduction and install instruction of Istio.
All the content is from: https://www.digitalocean.com/community/tutorials/how-to-install-and-use-istio-with-kubernetes

Here you can find just the important snippets and steps you need to install Istio and to test its functionalites by using a simple nodejs app.

## Install Istio
---

First add Istio Release Repository.  
`helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.1.7/charts/`

Check if repo was added.  
`helm repo list`

Install Istio's Custom Resource Definitions with istio-init chart.  
`helm install --name istio-init --namespace istio-system istio.io/istio-init`

This command commits 53 CRDs to the kube-apiserver, making them available for use in the Istio mesh. It also creates a namespace for the Istio objects called istio-system and uses the --name option to name the Helm release istio-init.  

Check that all of the required CRDs have been committed.  
`kubectl get crds | grep 'istio.io\|certmanager.k8s.io' | wc -l`

What are CRDs?  
In the Kubernetes API, a resource is an endpoint that stores a collection of API objects of a certain kind. For example, the built-in pods’ resource contains a collection of Pod objects. The standard Kubernetes distribution ships with many inbuilt API objects/resources. CRD comes into picture when we want to introduce our own object into the Kubernetes cluster to full fill our requirements. Once we create a CRD in Kubernetes we can use it like any other native Kubernetes object thus leveraging all the features of Kubernetes like its CLI, security, API services, RBAC etc.
The custom resource created is also stored in the etcd cluster with proper replication and lifecycle management. CRD allows us to use all the functionalities provided by a Kubernetes cluster for our custom objects and saves us the overhead of implementing them on our own.  
https://medium.com/velotio-perspectives/extending-kubernetes-apis-with-custom-resource-definitions-crds-139c99ed3477


You can now install the istio chart. To ensure that the Grafana telemetry addon is installed with the chart, we will use the --set grafana.enabled=true configuration option with our helm install command. We will also use the installation protocol for our desired configuration profile: the default profile. Istio has a number of configuration profiles to choose from when installing with Helm that allow you to customize the Istio control plane and data plane sidecars. The default profile is recommended for production deployments, and we'll use it to familiarize ourselves with the configuration options that we would use when moving to production.  
`helm install --name istio --namespace istio-system --set grafana.enabled=true istio.io/istio`

Verify Services:   
`kubectl get svc -n istio-system`

Verify Pods:   
`kubectl get pods -n istio-system`

The final step in the Istio installation will be enabling the creation of Envoy proxies, which will be deployed as sidecars to services running in the mesh.

Sidecars are typically used to add an extra layer of functionality in existing container environments. Istio's mesh architecture relies on communication between Envoy sidecars, which comprise the data plane of the mesh, and the components of the control plane. In order for the mesh to work, we need to ensure that each Pod in the mesh will also run an Envoy sidecar.

There are two ways of accomplishing this goal: manual sidecar injection and automatic sidecar injection. We'll enable automatic sidecar injection by labeling the namespace in which we will create our application objects with the label istio-injection=enabled. This will ensure that the MutatingAdmissionWebhook controller can intercept requests to the kube-apiserver and perform a specific action — in this case, ensuring that all of our application Pods start with a sidecar.  

Example: default namespace  
`kubectl label namespace default istio-injection=enabled`  
`kubectl get namespace -L istio-injection`

To control access to a cluster and routing to Services, Kubernetes uses Ingress Resources and Controllers. Ingress Resources define rules for HTTP and HTTPS routing to cluster Services, while Controllers load balance incoming traffic and route it to the correct Services.

Istio uses a different set of objects to achieve similar ends, though with some important differences. Instead of using a Controller to load balance traffic, the Istio mesh uses a Gateway, which functions as a load balancer that handles incoming and outgoing HTTP/TCP connections. The Gateway then allows for monitoring and routing rules to be applied to traffic entering the mesh. Specifically, the configuration that determines traffic routing is defined as a Virtual Service. Each Virtual Service includes routing rules that match criteria with a specific protocol and destination.

Though Kubernetes Ingress Resources/Controllers and Istio Gateways/Virtual Services have some functional similarities, the structure of the mesh introduces important differences. Kubernetes Ingress Resources and Controllers offer operators some routing options, for example, but Gateways and Virtual Services make a more robust set of functionalities available since they enable traffic to enter the mesh. In other words, the limited application layer capabilities that Kubernetes Ingress Controllers and Resources make available to cluster operators do not include the functionalities — including advanced routing, tracing, and telemetry — provided by the sidecars in the Istio service mesh.

To allow external traffic into our mesh and configure routing to our Node app, we will need to create an Istio Gateway and Virtual Service. In the 'application' folder you can find the corresponding files for that. 

  
## Example application and monitoring
---

node-isto.yaml - Gateway:  
In addition to specifying a name for the Gateway in the metadata field, we've included the following specifications:
- A selector that will match this resource with the default Istio IngressGateway controller that was enabled with the configuration profile we selected when installing Istio.
- A servers specification that specifies the port to expose for ingress and the hosts exposed by the Gateway. In this case, we are specifying all hosts with an asterisk (*) since we are not working with a specific secured domain.

node-isto.yaml - VirtualService:  
In addition to providing a name for this Virtual Service, we're also including specifications for this resource that include:
- A hosts field that specifies the destination host. In this case, we're again using a wildcard value (*) to enable quick access to the application in the browser, since we're not working with a domain.
- A gateways field that specifies the Gateway through which external requests will be allowed. In this case, it's our nodejs-gateway Gateway.
The http field that specifies how HTTP traffic will be routed.
- A destination field that indicates where the request will be routed. In this case, it will be routed to the nodejs service, which implicitly expands to the Service's Fully Qualified Domain Name (FQDN) in a Kubernetes environment: nodejs.default.svc.cluster.local. It's important to note, though, that the FQDN will be based on the namespace where the rule is defined, not the Service, so be sure to use the FQDN in this field when your application Service and Virtual Service are in different namespaces. To learn about Kubernetes Domain Name System (DNS) more generally, see An Introduction to the Kubernetes DNS Service.

Once you have created your application Service and Deployment objects, along with a Gateway and Virtual Service, you will be able to generate some requests to your application and look at the associated data in your Istio Grafana dashboards. First, however, you will need to configure Istio to expose the Grafana addon so that you can access the dashboards in your browser.

node-grafana.yaml:  
Our Grafana Gateway and Virtual Service specifications are similar to those we defined for our application Gateway and Virtual Service in Step 4. There are a few differences, however:

Grafana will be exposed on the http-grafana named port (port 15031), and it will run on port 3000 on the host.
The Gateway and Virtual Service are both defined in the istio-system namespace.
The host in this Virtual Service is the grafana Service in the istio-system namespace. Since we are defining this rule in the same namespace that the Grafana Service is running in, FQDN expansion will again work without conflict.

Create your Grafana resources with the following command:  
`kubectl apply -f node-grafana.yaml`

You can take a look at the Gateway and the VirtualService in the istio-system namespace with following commands:  
`kubectl get gateway -n istio-system`  
`kubectl get virtualservice -n istio-system`

Create the application Service and Deployment with the following command:  
`kubectl apply -f node-app.yaml`

Next, create your application Gateway and Virtual Service:  
`kubectl apply -f node-istio.yaml`

You can take a look at the Gateway and the VirtualService in the istio-system namespace with following commands:  
`kubectl get gateway`  
`kubectl get virtualservice`

We are now ready to test access to the application. To do this, we will need the external IP associated with our istio-ingressgateway Service, which is a LoadBalancer Service type.  
`kubectl get svc -n istio-system`

Shows NodeApp: http://ingressgateway_ip  
Shows Grafana: http://ingressgateway_ip:15031