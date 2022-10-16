# Kubecon North America 2022 :: Keynote Demo

In this step-by-step tutorial you will install and configure a set of Open Source projects that create an Internal Development Platform (IDP) for a fictisous company. 

The tutorial's section [Creating a new Environment]() uses [Crossplane](), the [Crossplane Helm Provider]() and [VCluster] to enable developers to request for new environments to do their work. The [Creating and deploying a function]() section uses [Knative Serving]() and [Knative Functions]() to create and deploy functions into these environments. Finally, the [Our function goes to production]() section uses [ArgoCD]() to promote the function that we have created to the production environment without requiring any teams to interact with the production cluster manually. 

You can read more about these projects and how they can be combined to build platforms in the  blog posts titled: **The challenges of building paltforms [1](),[2](),[3]() and [4]()**.

This step-by-step tutorial is divided into 5 sections
- [Use Case/Story](use-case.md)
- [Prerequisites and Installation]()
- [Requesting a new Environment]()
- [Creating and deploying a function]()
- [Our function goes to production]()



## Prerequisites and Installation 

This tutorial creates interact with Kubernetes clusters as well as install Helm Charts hence the following tools are needed: 
- [Install `kubectl`](https://kubernetes.io/docs/tasks/tools/)
- [Install `helm`](https://helm.sh/docs/intro/install/) 

For this demo we created two separate Kubernetes Clusters: one for the Platform which will host development environments and one for Production. while this tutorial is using KinD for simplicity, we encourage people to try the tutorial on real Kubernetes Clusters. [You can always get free credits here](https://github.com/learnk8s/free-kubernetes).

- [Creating the Platform Kubernetes Cluster](creating-a-kind-cluster.md)
- [Installing Crossplane into the Platform Cluster](installing-crossplane.md)
- [Installing Knative Serving into the Platform Cluster](installing-knative-serving.md)
- [Installing Command-line tools](installing-clis.md)
- [Creating the Production Kubernetes Cluster]()
- [Installing Knative Serving into the Production Cluster](installing-knative-serving.md)
- [Installing ArgoCD into the Production Cluster](installing-argocd.md)


### Configuring our Platform cluster

For this demo, our platform will enable development teams to request new `Environment`s.

These `Environment`s can be configured differently depending what the teams needs to do. For this demo, we have created a Crossplane Composition that uses VCluster to create one virtual cluster per Development environment requested. This enable the teams with their own isolated cluster to work on features without clashing with other teams work. 

For this to work we need to create the Custom Resource Definition (CRD) that defines the APIs for creating new `Environment`s and the Crossplanae Composition that define the resources that will be created every time that a new `Environment` resource is created. 

Let's apply the Crossplane Composition and our **Environment Custom Resource Definition (CRD)** into the Platform Cluster:
```
kubectl apply -f crossplane/environment-resource-definition.yaml
kubectl apply -f crossplane/composition-devenv.yaml
```

The Crossplane Composition that we have defined and configured in our Platform Cluster uses the Crossplane Helm Provider to create a new VCluster for every `Environment` with `type: development`. The VCluster will be created inside the Platform Cluster, but it will provide its own isolated Kubernetes API Server for the team to interact with. 

The VCluster created for each development `Environment` is using the Knative Serving Plugin to enable teams to use Knative Serving inside the virtual cluster, but without having Knative Serving installed. The VCluster Knative Serving plugin shares the Knative Serving installation in the host cluster with all the virtual clusters.

## Requesting a new Environment 

First, make sure that you are connected to the `platform` cluster.

Requesting a new `Environment` to the platform is easy, you just need to create a new `Environment` resource like this one: 

```
apiVersion: salaboy.com/v1alpha1
kind: Environment
metadata:
  name: arachnid-env
spec:
  compositionSelector:
    matchLabels:
      type: development
  parameters: 
    spidersDatabase: true
    cacheEnabled: true
    secure: true
    
```

And send it to the Platform APIs using `kubectl`:

```
kubectl apply -f arachnid-env.yaml
```

You can now treat your created `Environment` resource as any other Kubernetes resource, you can list them using `kubectl get environments` or even describing them to see more details. 



Because we are using VCluster you can go back and check if there is now a new VCluster with:

```
vcluster list 
```

Notice the VCluster is there but it shows not Connected.


We can now create a function and deploy it to our freshly created **Arachnid Environment**.


## Creating and deploying a function

Before creating a function, let's make sure that we are connected to our **Arachnid Environment**: 

```
vcluster connect arachnid-env --server https://localhost:8443 -- bash
```
or

```
vcluster connect arachnid-env --server https://localhost:8443 -- zsh
```

Now you are interacting with the VCluster, so you can use `kubectl` as usual. But instead of using `kubectl` we will use the [Knative Functions]() CLI to enable our developers to create functions without the need of writing Dockerfiles or YAML files. 

```
mkdir spiderize/
cd spiderize/

func create -r https://github.com/salaboy/func -l go -t spiders

func deploy -v --registry docker.io/<YOUR DOCKERHUB USER>

```

## Our function goes to production

We will now create a separate KinD Cluster to represent our **Production Environment**. This new Cluster will use ArgoCD to promote functions into the cluster. 

The idea here is not to interact with this cluster manually to deploy new functions, but use ArgoCD and a Git repository to host all the environment configurations. 

```
cat <<EOF | kind create cluster --name production --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31080 # expose port 31380 of the node to port 80 on the host, later to be use by kourier or contour ingress
    listenAddress: 127.0.0.1
    hostPort: 80
EOF

```

Then let's install Knative Serving: 

https://knative.dev/docs/install/yaml-install/serving/install-serving-with-yaml/#prerequisites


Now let's install ArgoCD