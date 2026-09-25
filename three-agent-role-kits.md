# Three role Kits for one Codex handoff

This example requires `sbx` 0.45 or later for v3 Kits ([Docker: Kits](https://docs.docker.com/ai/sandboxes/customize/)). Nothing in the example has been run here.

Use Docker's published v3 Codex workload, `docker.io/docker/sbx-kit-codex:0.155.1`. It is a v3 workload; the built-in `codex` shortcut remains v2 and cannot be combined with these v3 mixins ([Docker: Use kits](https://docs.docker.com/ai/sandboxes/customize/use-kits/#run-a-kit), [version rule](https://docs.docker.com/ai/sandboxes/customize/use-kits/#add-mixins)). Create one sandbox per role and add only that role's local directory as a mixin. The host supplies the repository-specific task prompt, performs the sequential Git handoff described in [three-agent-handoff.md](three-agent-handoff.md), and keeps GitHub tokens out of the sandboxes.

Place the role directories at these exact paths relative to the hypothetical repository root. They are deliberately **outside the mounted checkout**, so the role descriptors do not become repository changes:

**`../sandbox-roles/builder/builder.yaml`**

```yaml
# syntax=docker/sandbox-kit:3
schemaVersion: "3"
kind: mixin
displayName: Builder role
description: Guidance for implementing the sync dry-run behavior
capabilities:
  - type: com.docker.sandbox/agent-context@1
    config:
      content: |
        Builder role for the current host task. This is guidance, not an
        enforced permission boundary; follow the host task as authoritative.
        In the hypothetical repository, implement CLI `sync --dry-run`.
        Preserve normal `sync` behavior. In dry-run mode, report the operations
        that would occur while making no filesystem, remote-service, or other
        persistent-state changes. Follow existing conventions, make the smallest
        maintainable change, and leave Git integration to the host.
```

**`../sandbox-roles/tests/tests.yaml`**

```yaml
# syntax=docker/sandbox-kit:3
schemaVersion: "3"
kind: mixin
displayName: Tests role
description: Guidance for testing the sync dry-run behavior
capabilities:
  - type: com.docker.sandbox/agent-context@1
    config:
      content: |
        Tests role for the current host task. This is guidance, not an enforced
        permission boundary; follow the host task as authoritative.
        Work from the builder handoff. Add observable tests for CLI
        `sync --dry-run`: assert the user-visible plan and prove that relevant
        files, remote-service state, and other persistent state are unchanged.
        Exercise plausible side-effect paths without asserting implementation
        details, and leave Git integration to the host.
```

**`../sandbox-roles/review/review.yaml`**

```yaml
# syntax=docker/sandbox-kit:3
schemaVersion: "3"
kind: mixin
displayName: Review role
description: Guidance for reviewing and finishing the sync dry-run behavior
capabilities:
  - type: com.docker.sandbox/agent-context@1
    config:
      content: |
        Reviewer role for the current host task. This is guidance, not an
        enforced permission boundary; follow the host task as authoritative.
        Review the accumulated builder and tests handoffs. Find and fix defects
        in `sync --dry-run`, its observable tests, and edge cases that could
        still cause side effects. Update the user-facing CLI help so dry-run's
        reporting and no-side-effect contract are clear. Leave Git integration
        to the host.
```

A local directory is a supported Kit source, and `--kit` accepts the same source types, so the three mixin references are `../sandbox-roles/builder`, `../sandbox-roles/tests`, and `../sandbox-roles/review` ([local Kit sources](https://docs.docker.com/ai/sandboxes/customize/use-kits/#choose-a-kit-source)). Each directory needs only the descriptor shown: v3 permits a mixin with no companion recipe, producing a declaration-only Kit rather than requiring a Dockerfile or tool layer ([v3 content rules](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/SPEC-v3.md#35-content-rules-by-kind)).

`agent-context@1` is instruction metadata, not an access-control mechanism: its permission surface is explicitly “no.” Inline `content` is the supported form for a content-free Kit; the workload owns the profile filename, while each mixin only contributes text ([agent-context@1](https://github.com/docker/sandbox-kit-spec/blob/main/docs/spec/capabilities/com.docker.sandbox/agent-context@1.md)). Docker's v3 Codex workload descriptor selects `AGENTS.md`; these role mixins therefore must not declare `filename` ([Codex workload descriptor](https://github.com/docker/sandbox-kit-spec/blob/main/examples/codex/codex.yaml)). The role names and instructions shape agent priorities but do not restrict files, commands, network access, or inherited workload authority. Enforce real boundaries with the sandbox's runtime policy and keep the host task prompt explicit.
