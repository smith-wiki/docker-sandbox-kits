# A Kit composition needs one workload and may add mixins

A **workload** supplies the root filesystem and launch configuration. A **mixin** overlays tools or declarations onto it: for example, a GitHub CLI, its network request, and a credential binding. A runnable composition has exactly one workload and zero or more mixins ([v3 overview](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#1-overview)).

`provides` and `requires` describe dependencies **between Kits**, not permissions from the host. Resolution is closed: include the provider explicitly; no dependency is downloaded to fill a gap. Conflicts and duplicate providers fail, and the dependency graph—not flag order—sets overlay order ([resolution rules](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#53-resolution-semantics-consumer-contract)).

To share a chosen combination, author `kind: set` with references to published Kits. Building it resolves and merges them into a single ordinary OCI Kit, retaining the component digests. `set` is an **authoring** kind: the published result is a workload or mixin ([set rules](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/docs/spec/SPEC-v3.md#34-kit-set)). Its combined requests still need host authorization; see [who enforces permissions](permissions.md).

Related: [what a Kit is](what-are-kits.md) · [run the official example](try-it.md) · [compare with Google AX](compare-ax.md).
