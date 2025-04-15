# Setup

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
