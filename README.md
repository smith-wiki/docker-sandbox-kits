# Docker Sandbox Kits package environments and access requests

Start with [a concrete Kit example](what-are-kits.md); follow the linked cards for details.

## Questions and answers

1. **What is the Docker Sandbox Kit spec, and how do I use it?** A Kit packages software and access requests as an OCI image; `sbx` interprets them when it creates a sandbox. See [the example](what-are-kits.md) and [local steps](try-it.md).
2. **How does it compare with Google AX?** Kits describe an environment and its requested access; AX creates and tracks tasks on a Kubernetes/Agent Substrate cluster. See [the division of jobs](compare-ax.md).
3. **What exactly is a Kit, and which products compete on which dimensions?** The nearest alternatives differ by layer: environment definitions, sandbox runtimes, and fleet orchestrators. See [the concrete model](what-are-kits.md), [alternatives and criteria](alternatives.md), and [AX's separate role](compare-ax.md).
4. **Is NVIDIA OpenShell an alternative?** Yes to Docker Sandboxes as an agent sandbox runtime with access policies, and partly to a Kit's permission declarations; it is not the same OCI Kit format. See [the direct comparison](compare-openshell.md).
