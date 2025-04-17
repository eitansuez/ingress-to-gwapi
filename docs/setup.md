# Setup

## Provision a cluster

Feel free to provision a Kubernetes cluster any way you prefer.
The main requirement is that the environment support the assignment of an external IP address to LoadBalancer-type services.

=== "GCP"

    Here is an example command to provision a 3-node cluster on GCP (GKE) using the `gcloud` CLI:

    ```shell
    gcloud container clusters create my-k8s-cluster \
      --cluster-version latest \
      --machine-type "e2-standard-2" \
      --num-nodes "3" \
      --network "default"
    ```

=== "Local"

    Here is an example command to provision a local cluster with [k3d](https://k3d.io):

    ```shell
    k3d cluster create my-istio-cluster \
        --api-port 6443 \
        --k3s-arg "--disable=traefik@server:0" \
        --port 80:80@loadbalancer \
        --port 443:443@loadbalancer
    ```

## Deploy two distinct workloads

Deploy two distinct applications, each to its own namespace:

### `httpbin`

[`httpbin`](httpbin.org) is a sample application useful for testing HTTP communications.

```shell
k create ns httpbin
```

```shell
k apply -n httpbin -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml
```

### `bookinfo`

[`bookinfo`](https://istio.io/latest/docs/examples/bookinfo/) is one of Istio's sample applications, consisting of four microservices.

```shell
k create ns bookinfo
```

```shell
k apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/bookinfo/platform/kube/bookinfo.yaml
```

Both sets of workloads should now be running in their respective namespaces:

```shell
k get pod -n httpbin
```

And:

```shell
k get pod -n bookinfo
```

## Install ingress-nginx

Follow instructions from the project [quickstart](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start):

```shell
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

Verify that the controller is running (it can take a minute before the pod reaches READY state):

```shell
k get pod -n ingress-nginx
```

Also note the presence of a default ingress class named `nginx` that can be used to associate any Ingress resources we create with this particular controller:

```shell
k get ingressclass
```
