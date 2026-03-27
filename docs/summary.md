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

By migrating away from ingress-nginx, you not only eliminate technical debt, but are also avoiding a security issues with a release that is no longer maintained.

Perhaps just as importantly, you are now in a position to take advantage of the AI capabilities of the agentgateway project, for which we invite you to visit the agentgateway project [quickstart documentation](https://agentgateway.dev/docs/kubernetes/latest/quickstart/).