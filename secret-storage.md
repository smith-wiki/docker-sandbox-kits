# Where local Docker Sandbox secrets live

This card covers **local** Docker Sandboxes; cloud agents use a separate credential setup. In the [three-agent handoff](three-agent-handoff.md), the only agent-service secret is OpenAI's, entered on the host with:

```sh
sbx secret set openai
```

The builder, tests, and review sandboxes do not need a GitHub credential: they commit only in their private clones, while the host performs every fetch, push, and PR operation. `sbx login` is a separate Docker sign-in, not an OpenAI or GitHub secret ([local usage: sign in](https://docs.docker.com/ai/sandboxes/usage/#sign-in), [Docker Hub registry behavior](https://docs.docker.com/ai/sandboxes/configuration/credentials/#registry-credentials)).

## Physical storage on the host

`sbx secret set` stores a value—or a reference to a dynamic source—under a service identifier in the host credential store ([stored secrets](https://docs.docker.com/ai/sandboxes/configuration/credentials/#stored-secrets)). Docker documents the backing store as follows:

| Host | Documented location |
| --- | --- |
| macOS | The system Keychain. Docker does **not** document an underlying Keychain database file path, so do not guess one. |
| Windows | Windows Credential Manager. Docker does not document a backing file path. |
| Linux with a desktop keyring | The Secret Service exposed by the desktop keyring, such as GNOME Keyring or KDE Wallet. |
| Linux without a running Secret Service | A file somewhere under `$XDG_CONFIG_HOME/com.docker.sandboxes`, defaulting to `~/.config/com.docker.sandboxes`. Docker promises a `0700` containing directory but does **not** specify a fixed secret filename. |

These locations and the Linux fallback's permission model are specified in [Where secrets are stored](https://docs.docker.com/ai/sandboxes/configuration/credentials/#where-secrets-are-stored). The fallback protects data with filesystem permissions rather than a password; any user or process able to read the file can recover the credentials. If a Secret Service later becomes available, new secrets go to it again.

Do not confuse that store with `~/.config/sbx/credentials.yaml` on macOS/Linux or `%APPDATA%\sbx\credentials.yaml` on Windows. That file contains **approval bindings**—the permitted service mechanisms and domains—not secret values and not pointers locating them ([credential bindings](https://docs.docker.com/ai/sandboxes/configuration/credentials/#credential-bindings)). Likewise, a Kit's OCI descriptor only declares which service it needs and how the runtime presents it; the host credential store remains the source of the secret ([credential capability specification](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/capabilities/com.docker.sandbox/credential@1.md)). See [the Kit-side contract](secrets-contract.md) for that declaration.

## Global or one sandbox

A service secret is global by default:

```sh
sbx secret set openai
```

Scope a different value to one named sandbox with:

```sh
sbx secret set openai --sandbox builder
```

A sandbox-scoped value takes precedence over the global value. Removing the scoped value exposes the global value again, if one exists; changes apply to existing local sandboxes without restart ([store a secret](https://docs.docker.com/ai/sandboxes/configuration/credentials/#store-a-secret), [list and remove secrets](https://docs.docker.com/ai/sandboxes/configuration/credentials/#list-and-remove-secrets)). Thus the three-agent recipe can share one global OpenAI key, or set separate `builder`, `tests`, and `review` values with `--sandbox <name>`.

## What reaches the agent

For a proxy-managed credential, the host HTTP/HTTPS proxy matches the service and destination declared by the Kit, then rewrites the outbound request with the real credential. The sandbox normally sees only a sentinel such as `proxy-managed`, not the secret ([how credential injection works](https://docs.docker.com/ai/sandboxes/configuration/credentials/#how-credential-injection-works)). Host environment variables never auto-inject. A named API-key declaration receives a sentinel environment value; an inject-only declaration has no in-container environment presence at all. OAuth `passthrough: true` is an explicit isolation downgrade that returns the real token ([credential runtime behavior](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/capabilities/com.docker.sandbox/credential@1.md#runtime-behavior)).

A dynamic source keeps the same boundary. For example, `--ref` can store a 1Password reference or AWS Secrets Manager ARN; resolution happens through an authenticated tool on the **host**, and the resolved service credential is cached for 55 minutes by default. The store contains the reference rather than the resolved value, while the sandbox still receives only the proxy-managed placeholder ([dynamic secret sources](https://docs.docker.com/ai/sandboxes/configuration/credentials/#use-a-dynamic-secret-source)).

Finally, service secrets are not registry credentials. Private OCI registries use `sbx secret set --registry <host>` with their own host-only, all-sandboxes, or named-sandbox scopes; Docker Hub instead reuses the `sbx login` session ([registry credentials](https://docs.docker.com/ai/sandboxes/configuration/credentials/#registry-credentials)).
