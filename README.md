# Docker Sandbox Kits package environments and access requests

Start with [a concrete Kit example](what-are-kits.md); follow the linked cards for details.

## Questions and answers

1. **What is the Docker Sandbox Kit spec, and how do I use it?** A Kit packages software and access requests as an OCI image; `sbx` interprets them when it creates a sandbox. See [the example](what-are-kits.md) and [local steps](try-it.md).
2. **How does it compare with Google AX?** Kits describe an environment and its requested access; AX creates and tracks tasks on a Kubernetes/Agent Substrate cluster. See [the division of jobs](compare-ax.md).
3. **What exactly is a Kit, and which products compete on which dimensions?** The nearest alternatives differ by layer: environment definitions, sandbox runtimes, and fleet orchestrators. See [the concrete model](what-are-kits.md), [alternatives and criteria](alternatives.md), and [AX's separate role](compare-ax.md).
4. **Is NVIDIA OpenShell an alternative?** Yes to Docker Sandboxes as an agent sandbox runtime with access policies, and partly to a Kit's permission declarations; it is not the same OCI Kit format. See [the direct comparison](compare-openshell.md).
5. **Can Kit descriptors describe sandboxes run outside Docker?** Yes in principle: other runtimes can consume the OCI artifact, but must implement Kit resolution and enforce the capabilities they claim. See [the portability assessment](portability.md).
6. **Will Kits become independent like Dockerfiles?** Possible, not established: the spec is experimental and provides conformance tests, but published images alone do not make their access policy portable. See [the adoption signals and Dockerfile distinction](portability.md).
7. **Which existing languages compete?** Dev Container JSON, Compose YAML, OpenShell policy YAML, and Kubernetes manifests cover different parts of environment construction and access control. See [the descriptor comparison](descriptor-alternatives.md).
8. **How would three agents edit one repository in Docker Sandboxes?** Give an implementer, test author, and reviewer separate clone-mode sandboxes with v3 Codex plus small role mixins; carry their commits forward through the host, never a shared writable checkout. See [the role Kits](three-agent-role-kits.md) and [the Git handoff](three-agent-handoff.md).
9. **Where do the three agents' secrets live, and what does a Kit declare?** The v3 descriptor names a service and delivery rules, not the secret value. Local `sbx` keeps values in a host credential store and approvals in a separate non-secret bindings file; the host proxy supplies authorized requests. See [the Kit contract](secrets-contract.md) and [physical storage and scope](secret-storage.md).
