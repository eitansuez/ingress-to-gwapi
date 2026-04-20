# Introduction

In the early days of Kubernetes, teams started migrating their workloads from VMs to Kubernetes.
Teams set to work on containerizing and publishing their applications and services to container registries.
They had to develop Kubernetes deployment manifests, and figure out how to templatize and package their manifests, perhaps as a Helm chart.

Another piece of the puzzle had to do with configuring the routing of ingress traffic to their applications and APIs now residing inside a Kubernetes cluster.

One of the early "go-to" solutions for handling ingress traffic was the [ingress-nginx controller](https://kubernetes.github.io/ingress-nginx/).  As its name implies, this controller was designed to use the [nginx proxy](https://nginx.org/) as the runtime gateway that routed traffic to backend workloads.

As far as configuration was concerned, at the time, Kubernetes provided the [Ingress API](https://kubernetes.io/docs/concepts/services-networking/ingress/).

## Issues with the Ingress API

Many lessons were learned in the first few years of running workloads on Kubernetes, which culminated in the now-standard API for configuring ingress on Kubernetes:  the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/).

To better understand the issues with the original Ingress API, we must appreciate the myriad of reasons that organizations use proxies in the first place.
While it is true that fundamentally, the job of a proxy is to route external traffic to target backend workloads, proxies provide added benefits, in the form of:

- As its name implies, a proxy acts as a separation layer, a facade that hides internal details, which is desirable from a security standpoint.
- TLS termination:  the proxy can assume the responsibility of terminating TLS communication with the client, handling certificate creation, rotation, and configuration tasks on behalf of application teams.
- Load balancing:  distributing traffic across multiple backend instances increases availability and supports horizontal scaling.
- Rate limiting: enforces limits on requests per client.
- Request and response transformation: proxies can rewrite URLs, map paths, add or remove headers, they can act as a sort of adapter to backend services.
- Caching: static assets and other candidate requests can be handled at the proxy, lowering latency and lowering the load on backend servers.

The Ingress API supports very few of these use cases, and provides no mechanism for extension.
The only mechanism for configuring a proxy with capabilities not defined by the Ingress API was through ad-hoc annotations on the Kubernetes resource itself.

The ingress-nginx controller resorted to defining [annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/) as a means of configuring rate limiting, or any other proxy capability.

The Kubernetes Gateway API on the other hand is designed for extension, with clear extension points for implementations to associate their own Custom Resource Definitions with Kubernetes Gateway API resources, such as HTTPRoutes.

## The retirement of ingress-nginx

In late 2025 the Kubernetes community [announced the retirement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/) of the Ingress NGINX project, slated for March of 2026, with the recommendation that users migrate to an implementation of the Kubernetes Gateway API.

The implications of the retirement of ingress-nginx is that users can no longer expect new releases, and that includes patch releases which, as their name implies, exist to patch newly-discovered bugs and security vulnerabilities.  And so the risk and liability of running such a project increases dramatically.

The issue is no longer one of technical debt alone.
Teams that today still use the venerable Ingress API and the ingress-nginx controller must migrate, as the alternative now impacts the security of our systems.

## Which proxy?

The Kubernetes Gateway API gives users a standard, and many alternative implementations to choose from, as attested by the long [list of implementations](https://gateway-api.sigs.k8s.io/implementations/#gateway-controller-implementation-status).

Certain implementations leverage NGINX, others HA-Proxy or something else. Those proxies existed long before Kubernetes.
A relatively newer proxy (and open-source CNCF project) is [Envoy](https://www.envoyproxy.io/), which has been described as a "cloud-native" proxy for the Kubernetes age.
Envoy is a "batteries-included" proxy that supports many protocols, and that can manage being hot-reconfigured when routing rules or other configuration details change.

Many vendors opt to use Envoy as their cloud-native gateway of choice, and so even within this microcosm, users have a number of vendors they can choose from.

### Beyond Envoy

Envoy is a relatively newer proxy dubbed "cloud-native" in that it was designed specifically with microservice architectures in mind.
But several years have passed since the introduction of Envoy, and the technology landscape continues to evolve and to change.
Organizations today are building and deploying agentic solutions.
Aside from traditional microservices, organizations are now onboarding agentic workloads:  agents, MCP servers, and LLMs.

The [agentgateway project](https://agentgateway.dev/), started by Solo.io, represents a new generation of proxy servers designed specifically to accommodate agentic workloads.
In addition, agentgateway is fully conformant with the latest version of the Kubernetes Gateway API (at the time of this writing, version 1.5).

## About this workshop

The following pages contain instructions that will guide you through migrating an example project.
You will start out using the ingress-nginx controller with the Ingress API, with the objective of migrating to the Gateway API.

As of the time of this writing, Solo.io offers two Gateway API conformant implementations:

- [kgateway](https://kgateway.dev), whose controller is designed to program Envoy proxies.  kgateway is performant, mature, and sports many features.
- agentgateway, written in Rust, AI-native, is the younger and more opinionated of the two projects.

In this workshop, we will explore migrating to agentgateway, though the procedure for migrating to kgateway is very similar.

## Next

In the next section, you will begin constructing the initial environment.
You will:

- Deploy two distinct applications to a Kubernetes cluster.
- Install the ingress-nginx controller.
- Apply Ingress resources to configure the flow of ingress traffic to your APIs and applications.

You will next configure ingress with HTTPS, to simulate a setup that is more aligned with a real-world environment.

From there, we explore migrating to the Kubernetes Gateway API.