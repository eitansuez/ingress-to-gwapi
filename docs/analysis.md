# Analysis & Design

## Translating Ingress configurations

Before we dive in to creating the necessary Gateway API resources, we should consider options.

1. We can try to use a tool to translate the existing Ingress resources.
When the number or quantity of configuration files is large, and where the alternative of drafting the configurations by hand can be both tedious and error-prone, using a tool is usually the better option.
    On the other hand, tools are often incomplete, they don't always implement a spec fully, and are more mechanistic, in that they don't have intimate understanding of the original and target APIs and their nuances.

1. As long as the quantity of configuration is small, the translations can be performed manually.
    For large numbers of files, we can try to use a tool in combination with human review and testing to ensure correctness.

1. These days we have a third option:  leveraging an LLM to translate our configurations.
    This can be a viable option, but also comes with caveats:  it may be a security risk to divulge internal configurations if using an external provider.
    Also LLMs are not impervious to making mistakes, and so their output needs to be reviewed and often corrected.
    Finally, LLMs tend to produce different results each time they are run, they are non-deterministic.  Agents skills can be used to attempt to overcome such issues.

## Explore `ingress2gateway`

The Kubernetes community provides a tool named [ingress2gateway](https://kubernetes.io/blog/2023/10/25/introducing-ingress2gateway/) designed specifically for the purpose of migrating from Ingress.

Follow the instructions to install the tool's CLI on your machine.

The tool provides a number of options to control scope and source of configuration:  we can have it inspect a running cluster and look at Ingress objects that reside directly on the cluster, or we can point it at an input file on disk and have it translate that.  We can also scope the search for Ingress resources to a namespace, or have it look at all namespaces.

Here are a number of alternative, example invocations of the tool to generate configuration:

- Look for Ingress resources in the entire cluster:

    ```shell
    ingress2gateway print --providers ingress-nginx --all-namespaces
    ```

- Scope the search to the `httpbin` namespace:

    ```shell
    ingress2gateway print --namespace httpbin --providers ingress-nginx
    ```

- Or, for `bookinfo`:

    ```shell
    ingress2gateway print --namespace bookinfo --providers ingress-nginx
    ```

- Use an input file and translate that:

    ```shell
    ingress2gateway print --input-file ./httpbin-https-ingress.yaml \
      --namespace httpbin --providers ingress-nginx
    ```

- Or, for `bookinfo`:

    ```shell
    ingress2gateway print --input-file ./bookinfo-https-ingress.yaml \
      --namespace bookinfo --providers ingress-nginx
    ```

## Review and vet the generated configurations

The biggest difference by far between Ingress and Gateway API resources is that a single Ingress configuration maps to a pair of resources:  Gateway and an HTTPRoute.

Run the migration tool:

```shell
ingress2gateway print --providers ingress-nginx --all-namespaces
```

Study the output:

```yaml title="generated-config.yaml" linenums="1"
--8<-- "generated-config.yaml"
```

## Analysis

Like most tools, `ingress2gateway` performs direct translations:  for every Ingress resource, it produces a Gateway and an HTTPRoute "pair".
The tool does not perform intelligent analysis of the cluster, and will not take into account that there are two distinct applications where a plausible configuration would be to provision a single, shared gateway.

The other glaring difference between what the tool produced compared to what a human would can be seen in the generated HTTPRoute for the `bookinfo` application:
the repetition of the `backendRefs` section across multiple routing rules, failing to realize that the same configuration can be simplified to a single rule consisting of multiple `matches` clauses and a single `backendRef`.

Finally, the tool does not know what `gatewayClassName` to use for the translated resource, and so we must edit that value as well (to `agentgateway`).

In summary, using `ingress2gateway` is useful and insightful, but produces only a starting point for review and evaluation.
We must then make implementation decisions and, from the generated output, derive and craft the final Gateway API resource artifacts.

## Design decisions

We must decide between:

1. Using a shared gateway model, managed by cluster operators and giving them more control over ingress configuration.
2. Using a dedicated gateway per team, giving teams more autonomy, perhaps at a slightly higher cost.

The shared gateway model seems elegant but is complicated by these facts:

- The mapping from Ingress to Gateway API is not one-to-one.
- Automated migration tools struggle to provide a clean translation to this model.
- This model might require more coordination between the operator and development teams.

Using a dedicated gateway per team is a simpler approach:

- It's simpler for automated tools to generate the target configuration.
- Teams can migrate on their own schedule, in isolation from the activities of other teams.

The ability to gradually migrate to the Gateway API on a team-by-team basis is compelling.

We opt for the dedicated gateway model initially.
Transitioning to a shared gateway can be evaluated later, where it makes sense, and can be left as a follow-up refinement.

## AgentGateway specific migration tool

The `ingress2gateway` tool is [designed for extension](https://github.com/kubernetes-sigs/ingress2gateway/blob/main/docs/emitters.md) through the concept of providers and emitters.

AgentGateway supports migration from ingress-nginx with [a fork of the ingress2gateway](https://agentgateway.dev/docs/kubernetes/latest/migrate/) migration tool that supplies a bespoke emitter.

This tool knows how to translate ingress-nginx specific annotations to agentgateway-specific resources that naturally extend the Kubernetes Gateway API in places where the feature or capability is absent from the spec:  for features such as rate limiting, CORS support, authentication.

When invoking the tool, we specify an `--emitter` flag that tells the tool what implementation we are targeting.

Here is an example:

```shell
ingress2gateway print --providers ingress-nginx \
  --emitter agentgateway --namespace httpbin
```

This solves one of the problems we discussed previously:  the translated Gateway resources will bear the GatewayClass `agentgateway`.