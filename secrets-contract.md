# What a Kit credential declaration does

This card covers **local Docker Sandboxes**; cloud agents use a separate credential setup. A v3 Kit declares which service and phase need authentication and how the sandbox boundary presents it, but the OCI descriptor contains no key: the host credential store is the sole value source, and the user's credential binding authorizes the mechanism and domains ([Docker credential flow](https://docs.docker.com/ai/sandboxes/configuration/credentials/#how-credential-injection-works), [`credential@1` contract](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/capabilities/com.docker.sandbox/credential@1.md), [bindings](https://docs.docker.com/ai/sandboxes/configuration/credentials/#credential-bindings)). See [where the host stores those records](secret-storage.md).

## If an agent really needed the GitHub API

This is a complete declaration-only v3 mixin; it asks for runtime egress to `api.github.com` and a required `github` credential, but embeds no token:

```yaml
# syntax=docker/sandbox-kit:3
schemaVersion: "3"
kind: mixin
displayName: GitHub API access
capabilities:
  - type: com.docker.sandbox/network-policy@1
    config:
      runtime:
        allow: [api.github.com]
  - type: com.docker.sandbox/credential@1
    config:
      service: github
      phase: runtime
      apiKey:
        name: GH_TOKEN
        proxyManaged: true
        inject:
          - domain: api.github.com
            header: Authorization
            format: "Bearer %s"
```

Every injection domain must be in the matching phase's network allow-list; injection and egress permission are separate controls ([`network-policy@1` validation](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/capabilities/com.docker.sandbox/network-policy@1.md#validation)). On the host, the user would provide the `github` value—for example with `sbx secret set github`—and approve the local Kit's first-run binding; the binding records approval, not the secret ([Kit-declared services](https://docs.docker.com/ai/sandboxes/configuration/credentials/#services-declared-by-kits), [first-run approval](https://docs.docker.com/ai/sandboxes/configuration/credentials/#first-run-approval)).

The earlier [three-agent handoff](three-agent-handoff.md) deliberately needs no GitHub credential: each sandbox only commits in its private clone, while the host fetches, pushes, and opens the pull request. Its `docker.io/docker/sbx-kit-codex:0.155.1` workload already declares the `openai` runtime credential (API key or host-managed OAuth), so adding `openai` again in a role mixin would conflict with the one-entry-per-`(service, phase)` composition rule ([Codex v3 descriptor](https://github.com/docker/sandbox-kit-spec/blob/main/examples/codex/codex.yaml), [`credential@1` composition](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/capabilities/com.docker.sandbox/credential@1.md#composition)).

## What reaches the sandbox

With `name: GH_TOKEN` and `proxyManaged: true`, the sandbox receives a sentinel rather than the key. When a request matches `api.github.com`, the host proxy replaces the outbound `Authorization` header with `Bearer <host value>`; the declaration controls service, phase, domain, header, and format, not storage ([Docker injection flow](https://docs.docker.com/ai/sandboxes/configuration/credentials/#how-credential-injection-works), [`credential@1` runtime behavior](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/capabilities/com.docker.sandbox/credential@1.md#runtime-behavior)).

Keeping the raw key out of the VM does **not** make the API unusable to the agent: it can still issue requests that the host proxy authenticates within the approved domains and network policy. Grant only the services each sandbox actually needs ([Docker security boundary](https://docs.docker.com/ai/sandboxes/security/#trust-boundaries)).

OAuth follows the same isolation goal: the host handles sign-in, token refresh, and routing while the sandbox sees sentinels. Setting `oauth.passthrough: true` instead returns the real token to the sandbox and is explicitly a security downgrade ([Docker OAuth note](https://docs.docker.com/ai/sandboxes/configuration/credentials/#how-credential-injection-works), [`oauth.passthrough`](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/capabilities/com.docker.sandbox/credential@1.md#config)). Likewise, passing an actual key directly as an environment variable places that key inside the sandbox; Docker recommends proxy-managed stored secrets and placeholder values rather than manually setting API keys there ([credential best practices](https://docs.docker.com/ai/sandboxes/configuration/credentials/#best-practices), [placeholder values](https://docs.docker.com/ai/sandboxes/configuration/credentials/#custom-templates-and-placeholder-values)).

Role prompt text does not install, authorize, or confine a credential: `agent-context@1` has no permission surface and contributes only instructions ([`agent-context@1`](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/capabilities/com.docker.sandbox/agent-context@1.md)). The enforceable boundary is the resolved credential, binding, proxy, and network policy—not what a role tells the model.

One implementation caveat: the specification requires resolution to fail when a required credential has no binding, but current `sbx` non-interactive runs instead start with the credential withheld and print a warning; unattended workflows must pre-create the binding ([spec requirement](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/capabilities/com.docker.sandbox/credential@1.md#runtime-behavior), [`sbx` non-interactive limitation](https://docs.docker.com/ai/sandboxes/configuration/credentials/#first-run-approval)).
