# Basic Auth

In this exercise, we demonstrate the translation of an ingress-nginx "basic auth" configuration to AgentGateway.

Review the following Ingress configuration, and specifically the addition of two ingress-nginx annotations on lines 9 and 10:

```yaml linenums="1" title="httpbin-ratelimit-https-ingress.yaml" hl_lines="9-10"
--8<-- "httpbin-basicauth-https-ingress.yaml"
```

The `auth-type: basic` annotation turns on basic auth, while `auth-secret: basic-auth` relates the name of the secret containing the "password file."

Apply the configuration:

```shell
kubectl apply -f httpbin-basicauth-https-ingress.yaml
```

## Configure the access credentials

Basic auth is enforced by reading the password file from a specific named key inside a Kubernetes secret.

ingress-nginx expects the name of the key inside the secret to be `auth`, while in agentgateway, the key is `.htaccess`.

First, create the `.htaccess` file using the [`htpasswd`](https://httpd.apache.org/docs/current/programs/htpasswd.html) utility:

```shell
htpasswd -bcs .htaccess admin password
```

Interpret the above flags as follows:

- `b` - supply the password via the command line
- `c` - create the file `.htaccess`
- `s` - use the SHA-1 algorithm when hashing the password.  Only SHA-type hashed passwords are supported in agentgateway.  Although this particular algorithm is insecure, since the route is over a TLS listener, the credentials will be encrypted in transit.

Next, when we create the Kubernetes secret, we are careful to preserve the name for the secret.
On the other hand, the name of the `--from-file` argument is used by the command as the key name, which is exactly what agentgateway expects:

```shell
kubectl create secret generic -n httpbin basic-auth --from-file=.htaccess
```

## Run the migration tool

```shell
ingress2gateway print --providers ingress-nginx \
  --emitter agentgateway --namespace httpbin > generated-basicauth.yaml
```

The tool produces a notification warning us about the difference in conventions between the two ingress controllers.

Study the generated resources:

```yaml title="generated-basicauth.yaml" linenums="1" hl_lines="76-92"
--8<-- "generated-basicauth.yaml"
```

Like before, the new bits are the last listed resource:  The AgentgatewayPolicy:

- It binds to the HTTPRoute resource so that the policy applies to the requests to `httpbin`.
- It configures agentgateway for basic auth and specifies the reference to the secret containing the password file.

## Apply the generated resources

```shell
kubectl apply -f generated-basicauth.yaml
```

Check the status of the applied policy and make sure it was _Accepted_:

```shell
kubectl get agentgatewaypolicies -n httpbin httpbin-ingress -o yaml
```

Time to put it to the test..

## Test basic auth

Capture the gateway IP address:

```shell
export GW_IP=$(kubectl get gtw -n httpbin nginx -ojsonpath='{.status.addresses[0].value}')
```

Call `httpbin` once without authentication, a second time with incorrect credentials, and a third time with the correct credentials:

1. The response to the following request should be a 401 "Unauthorized":

    ```shell
    curl --head -s --insecure https://httpbin.example.com/get \
      --resolve httpbin.example.com:443:$GW_IP
    ```

1. Here too, we expect another 401 "Unauthorized" response (wrong password).

    ```shell
    curl --head -s -u "admin:wrong" --insecure https://httpbin.example.com/get \
      --resolve httpbin.example.com:443:$GW_IP
    ```

1. This request should produce an HTTP 200 "Success"!

    ```shell
    curl --head -s -u "admin:password" --insecure https://httpbin.example.com/get \
      --resolve httpbin.example.com:443:$GW_IP
    ```

## Summary

In this scenario, we let the migration tool interpret the basic auth configuration, and produce valid agentgateway resources that capture that configuration, in addition to the other generated Gateway API resources.