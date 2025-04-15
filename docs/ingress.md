# Configure ingress

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
