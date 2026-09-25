# Kits make a sandbox environment and its requested authority one OCI image

> Source snapshot: [`docker/sandbox-kit-spec` at `5fabb0a` (2026-09-25)](https://github.com/docker/sandbox-kit-spec/tree/5fabb0a260db4fded6c2f2da76e89b7582c9b536). V3 is **experimental**, with a final version targeted for Q4 2026 ([project status](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/README.md#docker-sandbox-kit-specification-v3)).

A Docker Sandbox Kit packages software plus declarations about the environment outside that software: network access, credentials, volumes, ports, setup hooks, agent instructions, and similar runtime needs. It does **not** grant those things to itself. A host reads the declarations and decides whether to grant, refuse, or ask about each request; the capability-specific pages define what a runtime claiming support must enforce ([Docker's introduction](https://www.docker.com/blog/docker-sandbox-kit-spec/), [v3 capabilities](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#7-capabilities)).

This separates two jobs:

- a **sandbox runtime** supplies the isolation boundary and applies policy;
- a **Kit** is the portable, versioned description of the content and authority the workload asks for.

An ordinary container engine can still run the image, but it ignores the Kit annotation: no Kit network policy, credential mediation, lifecycle hooks, or other host behavior is thereby enforced ([artifact definition](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#1-overview), [authoring guide](https://docs.docker.com/ai/sandboxes/customize/author/#capabilities)).

## The exact v3 model

Authors write YAML beginning with `# syntax=docker/sandbox-kit:3`; the BuildKit frontend validates it and publishes an ordinary OCI image. The manifest annotation `vnd.docker.sandbox.kit.descriptor` carries the authoritative JSON declarations, the standard image config carries the launch settings, and the layers carry the workload filesystem or mixin overlay. One digest pins all three; Kit sources are also staged under `/usr/share/sandbox/kit/` for inspection ([artifact definition](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#1-overview), [OCI layout](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#10-the-oci-layout)).

### Workload, mixin, and set

| Form | Meaning |
|---|---|
| `kind: workload` | Supplies the root filesystem and standard image launch configuration. Exactly one is required in a runnable composition. |
| `kind: mixin` | Supplies an overlay and/or declarations: for example, a CLI plus its network and credential requests. A composition can have zero or more. |
| `kind: set` | Authoring form that names other published Kits. Building it checks and merges them into one ordinary published `workload` or `mixin`; `set` is never a published runtime kind. |

`provides`, `requires`, `integrates`, and `conflicts` describe relationships **between Kits**. Resolution is closed: every requirement must already have a provider in the selected set; nothing is fetched automatically. The resolver rejects conflicts, duplicate providers, and anything other than one workload, then orders overlays by the dependency graph rather than `--kit` flag order ([resolution contract](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#53-resolution-semantics-consumer-contract)).

A `capabilities` entry instead asks the **host** for a typed, independently versioned behavior. `optional` defaults to false: under the specification, an unsupported required request fails closed, while an unsupported optional request is skipped and recorded ([capability grammar](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#7-capabilities)). V3 defines types for network policy, credentials, volumes, ports, USB devices, compute resources, privileged mode, lifecycle hooks, agent context, agent sessions, agent skills, Kit-registry access, and `sbx` workload behavior; the `@N` suffix versions that type's config schema, not the Kit format or CLI ([well-known types](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#72-well-known-types), [release version axes](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/RELEASES.md#version-axes)).

Composition reconciles declarations rather than letting the last flag win. In particular, network allows and denies are unioned per phase, deny wins, hooks concatenate in dependency order, and incompatible singleton requests fail. A published set freezes that reconciliation and records its components by digest ([set merge rules](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#95-merging-a-set), [`network-policy@2` composition](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/capabilities/com.docker.sandbox/network-policy@2.md#composition)).

## Authority is reviewable, but enforcement is runtime behavior

The specification normalizes requests into a permission surface: a runtime that gates updates must stop for approval when a new version widens access, including by removing a deny rule ([permission surface and gate](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#74-permission-surface-and-the-gate)). Diffable, digest-pinnable authority does not make an untrusted image safe by itself.

Important limits in the current Docker implementation:

- V3 needs `sbx` **0.45 or later**. Built-in shortcuts such as `claude` and `codex` still select v2 Kits and cannot be combined with a v3 mixin; use an explicit v3 workload reference or directory ([version compatibility](https://docs.docker.com/ai/sandboxes/customize/#version-compatibility)).
- Docker's authoring docs say `sbx` does not apply `usb-device@1`, `privileged@1`, or `agent-sessions@1`, and may create a sandbox even when such an unsupported capability is required. That differs from the specification's fail-closed contract, so do not treat “sandbox created” as proof that every required request was enforced ([documented implementation gaps](https://docs.docker.com/ai/sandboxes/customize/author/#capabilities), [runtime conformance requirement](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/conformance.md#22-verbs)).
- A Kit's network list is not necessarily the sandbox's complete effective policy. Mixins union their allows, Docker's defaults include broad wildcards, and organization policy can further constrain access. Inspect the active result with `sbx policy ls`; do not infer “only these hosts” from one descriptor ([network composition](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/capabilities/com.docker.sandbox/network-policy@2.md#composition), [Docker security considerations](https://docs.docker.com/ai/sandboxes/security/#security-considerations), [runtime access](https://docs.docker.com/ai/sandboxes/customize/use-kits/#runtime-access-and-instructions)).
- The microVM protects the rest of the host, not resources deliberately shared with it. Passing `.` normally mounts that directory read-write; Kit install commands run as root inside the sandbox, and changes to executable project files can later affect the host when a human runs them. Review the publisher, pin image digests where appropriate, prefer signed OCI Kits when trust policy requires it, and review workspace changes before executing them ([trust boundary](https://docs.docker.com/ai/sandboxes/security/#trust-boundaries), [security considerations](https://docs.docker.com/ai/sandboxes/security/#security-considerations), [signature verification](https://docs.docker.com/ai/sandboxes/customize/use-kits/#verify-kit-signatures)).

## Try the canonical local example

The official repository's smallest walkthrough composes the `hello` workload with the `gh` mixin. `hello` supplies a sandbox-ready root filesystem and Bash entrypoint; `gh` overlays a pinned GitHub CLI closure plus network, optional proxy-managed credential, and agent-context declarations ([`hello.yaml`](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/examples/hello/hello.yaml), [`hello.dockerfile`](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/examples/hello/hello.dockerfile), [`gh.yaml`](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/examples/gh/gh.yaml)).

### Prerequisites and install

Use `sbx` 0.45 or newer and sign in to Docker. On macOS, local sandboxes require macOS 14+ and Apple silicon; Docker Desktop/Engine is not required for `sbx`. For Windows and Linux requirements/installers, see Docker's [installation guide](https://docs.docker.com/ai/sandboxes/install/).

```bash
# macOS
brew trust docker/tap
brew install docker/tap/sbx
```

### Run it

This pins the repository checkout used for this page, then uses the exact local-directory command published by both the Docker article and repository README ([article](https://www.docker.com/blog/docker-sandbox-kit-spec/#try-it), [repository walkthrough](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/README.md#how-to-get-started)):

```bash
sbx login
git clone https://github.com/docker/sandbox-kit-spec.git
cd sandbox-kit-spec
git checkout 5fabb0a260db4fded6c2f2da76e89b7582c9b536
cd examples
sbx run ./hello --kit ./gh .
```

The final `.` shares the checked-out `examples` directory as the sandbox workspace; in local direct mode that mount is read-write, so use a disposable checkout if you do not want the agent editing another working tree ([workspace behavior](https://docs.docker.com/ai/sandboxes/security/#trust-boundaries)).

`sbx` builds unchanged local Kit sources once and reuses its cache. At the Bash prompt inside the sandbox, a small observable check is:

```bash
gh --version
cat /etc/motd
cat /usr/share/sandbox/kit/hello/kit.yaml
```

The first command checks that the mixin landed, the second checks the workload's built content, and the third shows the staged published descriptor ([local loop](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/README.md#how-to-get-started), [artifact source staging](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#10-the-oci-layout)). In another host terminal, use `sbx policy ls` to inspect the effective network rules rather than assuming `gh.yaml` is the whole policy.

The command sequence above was checked against the official article, install guide, repository walkthrough, and example files at the pinned revision. It was not executed here because `sbx` is not installed on this machine; it is a source-verified procedure, not a claim of a successful local launch.
