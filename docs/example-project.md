# Example project

## Provision a cluster

```shell
k3d cluster create my-istio-cluster \
    --api-port 6443 \
    --k3s-arg "--disable=traefik@server:0" \
    --port 80:80@loadbalancer \
    --port 443:443@loadbalancer
```

## Deploy two distinct workloads

`httpbin`:

```shell
k create ns httpbin
```

```shell
k apply -n httpbin -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/httpbin/httpbin.yaml
```

`bookinfo`:

```shell
k create ns bookinfo
```

```shell
k apply -n bookinfo -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/bookinfo/platform/kube/bookinfo.yaml
```

## Install ingress-nginx

Follow instructions from the project [quickstart](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start):

```shell
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

## Configure ingress

`httpbin`:

```yaml linenums="1" title="httpbin-ingress.yaml"
--8<-- "httpbin-ingress.yaml"
```

`bookinfo`:

```yaml linenums="1" title="bookinfo-ingress.yaml"
--8<-- "bookinfo-ingress.yaml"
```

## TLS

- configure ingress for both apps using hostnames httpbin.example.com and bookinfo.example.com
  - with tls
- test it

## Migration:

provision kgateway
 install ingress2gateway
 generate gw api artifacts from ingress api resources
 review generated artifacts
 configure kgateway for httpbin: apply generated artifacts

switch dns for httpbin to new gw ip
configure ingress for bookinfo
switch dns for bookinfo to new gw ip

decommission ingress-nginx