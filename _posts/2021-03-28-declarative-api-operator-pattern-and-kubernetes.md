# Declarative API, Operator Pattern, and Kubernetes

## Declarative API

There are a few online posts that describe the [declarative API](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/_print/#declarative-apis), it's an API programming paradigm that describes the current and intended state of the subject, leaves the control flow to the API provider. The other being commonly used paradigm is imperative API, which describes the operation towards the subject. Both paradigms are widely used, for example, all the SQL queries are imperative API, in which the user directly tell the database what do to, and most configuration management tools (e.g. puppet) are declarative, where the user of the API state the expected status of the resource, and the API provider owns how to make it happen. A real-world case might be, to open a door, a declarative API would ask for the intended state of the door, and an imperative API might ask for what you intend to do with such a door. If we change the door to a MySQL database, the case also stands.

There is no silver bullet in API design, each paradigm has the best scenario it fits. To decide which paradigm to use, one question often being asked whether the API user would decide how the service will be done. The Kubernetes community summarized as below:

In a Declarative API, typically:

* Your API consists of a relatively small number of relatively small objects (resources).
* The objects define the configuration of applications or infrastructure.
* The objects are updated relatively infrequently.
* Humans often need to read and write objects.
* The main operations on the objects are CRUD-y (creating, reading, updating and deleting).
* Transactions across objects are not required: the API represents the desired state, not an exact state.

Imperative APIs are not declarative. Signs that your API might not be declarative include:

* The client says "do this", and then gets a synchronous response back when it is done.
* The client says "do this", and then gets an operation ID back, and has to check a separate Operation object to determine completion of the request.
* You talk about Remote Procedure Calls (RPCs).
* Directly storing large amounts of data; for example, > a few kB per object, or > 1000s of objects.
* High bandwidth access (10s of requests per second sustained) needed.
* Store end-user data (such as images, PII, etc.) or other large-scale data processed by applications.
* The natural operations on the objects are not CRUD-y.
* The API is not easily modeled as objects.
* You chose to represent pending operations with an operation ID or an operation object.

Declarative API and imperative API often work together too, for example, stateful services often having imperative backend for record-keeping, and the user interaction part is often declarative, where the user checks the intended state and current state of the resource. Another example is, Ansible playbook itself is imperative, but for each task, it could be declarative.

Declarative API can also be composed together, for example, to make a webserver running, it could include ensuring an EC2 node, ensure Nginx is running on such EC2 node, and ensure an LB configured for such EC2 node.


## Operator Pattern

[Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) is an extension to the existing software shipment method - besides deploying an app that follows its configuration, it also having a control loop check the service runtime and adjusting the application consistently.

In a typical modern operation pattern, SWEs and SREs work out deployment automation that rolls out the service into production. When a production incident happens, the SRE will identify the issue and provide mediation, workout RCA with SWE. With the time being, more sites will be rolled out, SRE will need to run the deployment automation many times, as well as collecting and sharing the runbook within the team. Depends on the size of the team, an SRE team often run into the issue of balancing between fixing a production issue and automate a runbook. This situation might get even worse for the infrastructure team, which on one hand is accountable for a lot of fundamental services, and on the other hand, CVEs and security change consistently coming in with high priority. As a result, the bottleneck is not the velocity of developing a new feature, nor the velocity it could be deployed to production, it's at the velocity of bringing all team members on the same page with the runbooks.

The operator pattern is nothing fancy, it automates the process SREs need to do often manually or through push buttons. It handles pushing the service into production, performing upgrading, and also include the cycle of "monitor - pager - identify the issue - follow SOP runbook" into automation. In operator pattern, the operator of a service running a loop checking the service status, if it doesn't exist, deploy it, it is not the version is supposed to be, upgrade it, if the /health and /metric returns odd number, it knows how to handle it - by restarting the service or scaling out. It not a replacement of SRE, it codified the most routine work so that SRE could more focus on improving the stability (instead of telling the operator to restart the service every a few days). At the meantime, it encourages service owner to own more parts of the service, motivate them to improve the /heath or /metrics data and improve the service availability. For the infrastructure team, it pushes to offer a multi-tenancy and standardized interface, for other services want to minimize the operator dev cycle and more focus on service-specific operations.

Kubernetes community gave an example that you can use an operator to automate include:

1. deploying an application on demand
2. taking and restoring backups of that application's state
3. handling upgrades of the application code alongside related changes such as database schemas or extra configuration settings
4. publishing a Service to applications that don't support Kubernetes APIs to discover them
5. simulating failure in all or part of your cluster to test its resilience
6. choosing a leader for a distributed application without an internal member election process

The operator doesn't mean one operator needs to do all the things in a silo, and a few operators could be composed together. Take a DB operator as an example, for deployment, it could send a request to a chef/tf operator to finish a configuration run, and send a request to monitor operator to configure alert for it, only things directly related to DB service itself, like make a replica or extend volume when the disk is full.

## Kubernetes

Kubernetes uses both declarative API and operator pattern a lot, and encourages the user to customize/extend the kubernetes through [CustomResourceDefinitions](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/).

[Under the hood of kubernetes](https://kubernetes.io/docs/concepts/overview/components/), at the control plane, there are etcd for all the stateful data, and kube-api as a frontend api service, besides them, there are kube-scheduler, kube-controller-manager and cloud-controller-manager, with the popularity of CRDs, there are 3rd party addons like ingress-controller, LB-controller, certificate-controller, etc. Worth pointing out, the kube-api is the frontend for all scheduler and managers, in a declarative way, and all controllers and managers follow the operator pattern. At worker nodes, there are also `kubelet`, `kube-proxy` and `container runtime` that owns the container management.

What does it mean, here is an overly simplified explanation example.

1. Below is a deployment spec for nginx, when user run `kubectl apply -f spec.yaml`, the client convert it into POST/PATCH `/apis/<GROUP>/<VERSION>/namespaces/<NAMESPACE>/<RESOURCETYPE>` (See [API concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)). 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

2. `kube-apiserver` received such requests and save them into etcd.
3. The Deployment operator (as part of `kube-controller-manager`), which is watching `Deployment` resource, will create `Replicaset` as subresource, and will watch the status change of the children resource
4. The `Replicaset` operator (also part of `kube-controller-manager`), will create `Pod` resource and will watch the status change of the children resource
5. The `kube-scheduler`, which is watching the `Pod` resource without an assigned node, will find an available node that meets the need, and update the Pod resource
6. Some other services, like CNI controller that will also watch `Pod` resource without assigned network, do their job to fill in the required information
7. The `kubelet` found the `Pod` Spec has enough information, will work `kube-proxy` and `container runtime` to bring up the container.

As you can see in this whole process, each controller and manager only does a minimal thing to ensure (or a more frequently used word, reconcile) the intended state, all steps are worked asynchronized and can be scaled out individually. And if you look into the codes, during the reconcile loop of a given resource, it only moves the needle a little and delegates the work to a children resource.

The CRDs are in the same way, you can register your API, and having an operator watch it and perform accordingly.

It doesn't mean this is the silver bullet, k8s community has to workout `finializer` to create dependencies between resources having relations, as well as webhooks to handle some synchronized works. But in general, it works pretty well for such a scalable service.

## Summary

This post briefly touched on the philosophy behind the kubernetes API design and pattern being used. The declarative API and Operator are not only in kubernetes community but also in the community like react and configuration management tools. The understanding of the declarative API paradigm and operator pattern, it makes much easier to understand kubernetes services, and build platforms that could benefit from them.
