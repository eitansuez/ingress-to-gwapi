# Summary

In this workshop, you migrated a variety of ingress-nginx configurations to agentgateway.

You utilized a custom migration tool that is both aware of many of the essential ingress-nginx annotations, and that can produce valid Gateway API and AgentGateway CRDs.

You worked through three basic scenarios:

- HTTPS ingress with TLS termination and http-to-https scheme redirect
- Local rate limiting
- Basic auth

The migration tool supports additional scenarios, including:

- Canary releases
- External auth
- CORS, and
- Backend TLS configurations.

For details, see [the examples](https://agentgateway.dev/docs/kubernetes/latest/migrate/examples/) in the documentation.

We also looked at taking a different approach, and building out a shared gateway for both applications.  This is an equally valid approach.

By migrating away from ingress-nginx, you not only eliminate technical debt, but are also avoiding security issues with a release that is no longer maintained.

Perhaps just as importantly, you are now in a position to take advantage of the AI capabilities of the agentgateway project, and other capabilities that will take your environment well beyond what the original system could do:

1. Use agentgateway's AI capabilities to proxy and secure agentic workloads:  agents, MCP servers, LLMs, etc..
1. Integrate agentgateway with your Istio service mesh to gain encrypted communication with mutual TLS from the gateway to your mesh backend workloads.
1. With ambient mesh, you can use agentgateway as your waypoint, unlocking agentgateway's capabilities for workloads running internally.
1. Use agentgateway as an egress gateway to control traffic exiting your environment.

For more information about these capabilities, visit [agentgateway.dev](https://agentgateway.dev/).