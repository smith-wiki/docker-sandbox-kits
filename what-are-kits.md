# A Kit is a software image with an attached access request

**Think “package + permission slip,” not “new kind of VM.”** A Kit is an ordinary OCI image. Its layers may contain an agent, a tool, or a base environment; a mixin may add only declarations. One manifest annotation describes what it needs from outside: network destinations, a credential binding, a volume, startup behavior, or instructions. The image digest pins content and requests together ([v3 artifact](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#1-overview), [declaration-only mixin](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#35-content-rules-by-kind)).

The [official `hello` + `gh` example](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/README.md#how-to-get-started) makes this concrete:

1. `hello` is the **workload**: a filesystem and Bash launch command ([recipe](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/examples/hello/hello.dockerfile)).
2. `gh` is a **mixin**: the GitHub CLI plus requests to reach GitHub and optionally use a host-held token ([descriptor](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/examples/gh/gh.yaml)).
3. `sbx run ./hello --kit ./gh .` combines them and starts the sandbox. **`sbx` is the runtime** that decides and enforces access; the Kit does not create a sandbox or grant itself permissions ([run guide](https://docs.docker.com/ai/sandboxes/customize/use-kits/), [capability contract](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#7-capabilities)).

Ordinary `docker run` ignores the Kit annotation, so the requested protection does not follow the image into an engine that does not implement it ([v3 overview](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#1-overview)).

Related: [how Kits combine](composition.md) · [what is actually enforced](permissions.md) · [who competes at each layer](alternatives.md) · [why AX is different](compare-ax.md).
