# A Kit packages a sandbox environment and its access requests

> Source snapshot: [`docker/sandbox-kit-spec` at `5fabb0a` (2026-09-25)](https://github.com/docker/sandbox-kit-spec/tree/5fabb0a260db4fded6c2f2da76e89b7582c9b536). V3 is experimental ([project status](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/README.md#docker-sandbox-kit-specification-v3)).

A Kit is an ordinary OCI image: its layers carry software, while the manifest annotation `vnd.docker.sandbox.kit.descriptor` carries declarations such as network and credential requests. The image's digest pins both together. An ordinary container engine ignores the annotation, so **the image alone does not enforce its requests**; a supporting sandbox runtime must interpret and apply them ([v3 overview](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#1-overview), [Docker authoring guide](https://docs.docker.com/ai/sandboxes/customize/author/#capabilities)).

- [Composition](composition.md): how a workload, mixins, and a published set fit together.
- [Permissions](permissions.md): what the runtime must enforce, and current `sbx` limits.
- [Try it](try-it.md): a source-verified local `hello` + `gh` walkthrough.
- [Compare with Google AX](compare-ax.md): packaging and permissions versus orchestration.
