# Hand one change through three clone-mode agents

This is a **hypothetical, source-verified walkthrough**, not a record of commands run here. Start in the main checkout of the Git repository, with the three local declaration-only mixins in `../sandbox-roles/{builder,tests,review}`. See [the companion role card](three-agent-role-kits.md) for what those mixins contribute.

## Prepare the host

```sh
sbx login
sbx secret set openai
git switch main
```

`sbx secret set openai` prompts on the host; proxy-managed credentials stay on the host rather than becoming files or environment variables in the VM ([credential storage](https://docs.docker.com/ai/sandboxes/configuration/credentials/#store-a-secret)). Do not configure a GitHub secret for these sandboxes: agents commit locally, while the host alone fetches, pushes, and opens the PR.

Every run below uses Docker's published v3 Codex workload and one compatible v3 local mixin, exactly the combination supported by `--kit` ([published workload and mixins](https://docs.docker.com/ai/sandboxes/customize/use-kits/#run-a-kit), [add mixins](https://docs.docker.com/ai/sandboxes/customize/use-kits/#add-mixins)). The role instructions guide the agent; they are not permission enforcement, so each interactive prompt also states the Git boundary.

`--clone` matters. It gives the agent a private writable clone and exposes the host repository only read-only; changes reach the host only after an explicit fetch ([clone mode](https://docs.docker.com/ai/sandboxes/workflows/git/#clone-mode)). The final `.` is the clone source, not a direct read-write workspace. Direct mode is rejected here because edits appear on the host immediately, and a host worktree is rejected because its edits also appear immediately while the agent has no Git access ([mode comparison](https://docs.docker.com/ai/sandboxes/workflows/git/)).

## 1. Builder

Start the first interactive session:

```sh
sbx run --clone --name builder \
  docker.io/docker/sbx-kit-codex:0.155.1 \
  --kit ../sandbox-roles/builder .
```

Enter this as the Codex prompt inside that session (it is not an `sbx` flag):

> Work only in the private clone. Create and switch to branch `agent/builder`. In this hypothetical CLI repository, implement `sync --dry-run` so it reports the changes it would make but performs no writes. Commit the completed change on `agent/builder`. Do not push or contact GitHub. Stop after reporting the commit hash.

While the `builder` sandbox is still running, use another **host** terminal:

```sh
git fetch sandbox-builder
git log --oneline main..sandbox-builder/agent/builder
git switch -c agent/tests sandbox-builder/agent/builder
```

`sbx` wires the private clone to the host as `sandbox-<name>`, so this fetch is the documented transfer mechanism ([sandbox remote behavior](https://docs.docker.com/ai/sandboxes/workflows/git/#sandbox-remote-behavior)). The last command is the handoff: it changes the host's checked-out ref to a new `agent/tests` branch based on the builder commit. The `main` ref is not advanced.

## 2. Tests agent

From that same main checkout, now on `agent/tests`, start a separate session:

```sh
sbx run --clone --name tests \
  docker.io/docker/sbx-kit-codex:0.155.1 \
  --kit ../sandbox-roles/tests .
```

Enter this prompt interactively:

> Confirm that the private clone is on `agent/tests` and includes the builder commit. Add observable tests proving that `sync --dry-run` reports the intended actions and leaves both repository state and files unchanged. Commit the tests on `agent/tests`. Do not push or contact GitHub. Stop after reporting the commit hash.

A clone-mode sandbox follows the ref checked out on the host **when that sandbox is created**, which is why the host switch precedes this run ([clone-mode constraints](https://docs.docker.com/ai/sandboxes/usage/#clone-mode)). There is no direct sandbox-to-sandbox transfer.

Fetch and establish the next base on the host:

```sh
git fetch sandbox-tests
git log --oneline sandbox-builder/agent/builder..sandbox-tests/agent/tests
git switch -c agent/review sandbox-tests/agent/tests
```

The host ref change makes the tests commit visible when the next sandbox clones the repository; nothing is implicitly shared between the two VMs.

## 3. Reviewer

Start the third independent session:

```sh
sbx run --clone --name review \
  docker.io/docker/sbx-kit-codex:0.155.1 \
  --kit ../sandbox-roles/review .
```

Enter this prompt interactively:

> Confirm that the private clone is on `agent/review` and includes the builder and tests commits. Review the implementation and tests, find and fix defects, and update the user-facing CLI help for `sync --dry-run`. Commit the completed review on `agent/review`. Do not push or contact GitHub. Stop after reporting the commit hash.

Fetch and inspect the complete result on the host before integrating it:

```sh
git fetch sandbox-review
git log --oneline main..sandbox-review/agent/review
git diff --stat main...sandbox-review/agent/review
git diff main...sandbox-review/agent/review
git switch -c agent/final sandbox-review/agent/review
git diff --check main...agent/final
```

Run the repository's documented targeted tests and a manual `sync --dry-run` check on `agent/final`; their exact commands depend on the hypothetical repository. Only after those checks pass, publish from the host:

```sh
git push -u origin agent/final
gh pr create --base main --head agent/final --fill
```

Keep each sandbox running until its host fetch completes: stopping it makes its Git daemon temporarily unreachable, and removing it deletes both the private clone and the `sandbox-<name>` remote ([remote lifetime](https://docs.docker.com/ai/sandboxes/workflows/git/#sandbox-remote-behavior), [persistence warning](https://docs.docker.com/ai/sandboxes/usage/#clone-mode)). Clone contents otherwise persist with the sandbox, while the host working tree changes only at the explicit `git switch` handoffs.