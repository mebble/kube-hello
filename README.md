# kube-hello

## Requirements

- kubernetes
- minikube
- nginx ingress controller

### Deployment

To deploy the gateway, deployment and service for the first time, or to apply any changes made to their configuration files after deploying, run the command:

```sh
$ ./apply-all.sh
```

After deploying, find the deployed objects' names with:

```sh
$ kubectl get (pods | deployments | services | ingresses)
```

Make a request directly to `<service-name>` from the browser

```sh
$ minikube service <service-name>
```

Find all ingress controller pods
```sh
$ kubectl get pods --all-namespaces | grep -i ingress
```

### Make Requests

In this project, both the `node-app-service` service and `node-app-ingress` gateway are hosted on the lone host in the minikube cluster. Hence, they both have the same IP address. Running `$ kubectl get ingresses` and `$ kubectl get services` shows us their respective port on this lone host under the `PORTS` column. The `PORTS` column of the service includes a port mapping from a `NodePort` to a service `Port`.

To hit a service directly, make a request to the service `NodePort`, assigned by k8s. That will do port mapping to the service `Port`, then loadbalancing to one of the matched pods (according to the selector), and port forwarding to the pod's port, specified by the service's `targetPort`.

```sh
$ curl <minikube-host>:<service-nodeport>/
$ curl <minikube-host>:<service-nodeport>/pingpong/hii
```

To hit a service through the Ingress gateway, make a request to the port the Ingress is running on, assigned by k8s. Ingress forwards the request to the appropriate port on the appropriate service using the spec rules. There's a one-one mapping between the request URL path and the (serviceName, servicePort) pair, specified by `serviceName` and `servicePort` in the spec rules. Then the loadbalancing and so on proceeds.

Note that in this request forwarding, the URL path of that request is passed on to the target service, and ultimately to the target pod. This URL path could also be rewritten/modified before the request forwarding, according to the ingress annotation, and that modified path would be the one passed on. The ingress object/resource itself doesn't apply any semantics to the annotations. It's the ingress controller that receives and enforces these annotations.

```sh
$ curl <minikube-host>:<ingress-port>/node-app
$ curl <minikube-host>:<ingress-port>/node-app/pingpong/hii
```

#### Update: Proxy Sidecar

In each pod, a proxy "sidecar" container was created, that simply routes requests to the main container of the pod. The Ingress gateway was made to forward requests to this proxy sidecar instead of the main container.

### Bad Requests

Here are some ways we might make a bad request:

```sh
$ curl <minikube-host>:<ingress-port>/
// returns 404 Not Found
```

Here, the gateway didn't find any service to handle a request to that path.

```sh
$ curl <minikube-host>:<ingress-port>/node-app-faulty
// returns 502 Bad Gateway
```

Here, the gateway found a service to forward the request to, but that service didn't have any server running at the service's pod's target port.

```sh
$ curl <minikube-host>:<ingress-port>/node-app/foo
// returns 404 Not Found
```

Here, the gateway found a service to forward the request to, and a server was running at the service's pod's target port, but that server didn't find any handler for a request to that path.

### Through Services, To Pods

There's a one-one mapping between a service and a set of pods matched by that service. When a request hits a `LoadBalancer` service, the service's load-balancing "resolves" the set of matched pods to a single pod. If the pods in this set are fungible (they might not always be), this fungibility plus the load-balancing allows us to think of it as a one-one mapping between a service and a pod. This mental model is useful because it simplifies the tracking of how requests are forwarded -- if two requests are forwarded to the same service, they are forwarded to the same pod (technically different replicas of the same pod, but with replicas, we can abstract them away as being one entity). Since a pod represents a single computer/host, all that's left to be concerned about is the port number.

Hence, if ingress forwards a request to serviceA port xxxx, that will just forward the request to podA port yyyy, where serviceA and podA are in a one-one relationship. This xxxx->yyyy port mapping is specified by the `port` and `targetPort` fields in the service yaml configuration.

### Non-fungible Pod Sets

_Not sure if I understood this correctly_

We could look at a situation where there are two deployments, `D1` and `D2`. `D1` deploys pods, each of which whose container is based on `image1`, and for `D2`, `image2`. If all the pods of both deployments have the same metadata labels, and `D1`, `D2` and even by some service `S` selects those labels, then all pods will be matched by `D1`, `D2` and `S`. In this case, all pods of the matched set (from the perspective of `D1`, `D2` and `S`) aren't fungible.

### "Prefix style" gateway mapping

Suppose we have an ingress gateway that could forward requests to two k8s services, `serviceA` and `serviceB`. We want the gateway forward our request to one of these services based on the path we provide in the URL, as follows:

```
<gatewaydomain>/service-a                    --> <serviceAdomain>/
<gatewaydomain>/service-a/                   --> <serviceAdomain>/
<gatewaydomain>/service-a/some/more/path     --> <serviceAdomain>/some/more/path

<gatewaydomain>/service-b                    --> <serviceBdomain>/
<gatewaydomain>/service-b/                   --> <serviceBdomain>/
<gatewaydomain>/service-b/some/more/path     --> <serviceBdomain>/some/more/path
```

That is, whatever service we hit depends on the `service-x` portion of the path, that functions as a gateway mapping prefix. Everything beyond this prefix is the actual path of the web resource we are requesting. In this situation, we could configure ingress by capturing the portion of the URL path after the prefix, and using this captured group in the re-written path (specified in the `rewrite-target` annotation).

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-gateway
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - http:
      paths:
      - path: /service-a(/|$)(.*)
        backend:
          serviceName: serviceA
          servicePort: <somePort>
      - path: /service-b(/|$)(.*)
        backend:
          serviceName: serviceB
          servicePort: <somePort>
```

Notice how the regex patterns require the `(/|$)` portion, because if they didn't, a request of the form `<gatewaydomain>/service-a` (without the trailing slash) wouldn't match any of these patterns. So it wouldn't be mapped to `<serviceAdomain>/` like we intended it to be.

## Commands

```sh
$ kubectl get <resource-type>
$ kubectl describe <resource-type> <resource-name>
$ kubectl logs -f <pod-name>
$ kubectl delete <resource-type> <resource-name>
```

## TODOs

- PORT env overriding
- `containerPort` doesn't enforce expose
- deployment and service name are from yaml, pod name is deploymentName-replicasetHash-podHash
- delete a pod, see the deployment creating a new one
- get service port mappings written backwards
- port mapping from NodePort to Port. port forwarding from Port to TargetPort
- service port mapping is needed if i'm hitting the minikube cluster host at nodeport. Else, if service has external ip address, we can hit the service Port directly ??
