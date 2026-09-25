# Try a v3 workload with a GitHub CLI mixin locally

On an Apple Silicon Mac running macOS 14+, install `sbx` **0.45 or later**; Docker Desktop/Engine is not required for `sbx`. [Docker's install guide](https://docs.docker.com/ai/sandboxes/install/) covers other platforms. Built-in names such as `codex` select v2 Kits, so use the explicit v3 workload below with a v3 mixin ([version compatibility](https://docs.docker.com/ai/sandboxes/customize/#version-compatibility)).

```sh
brew trust docker/tap
brew install docker/tap/sbx
sbx login
git clone https://github.com/docker/sandbox-kit-spec.git
cd sandbox-kit-spec
git checkout 5fabb0a260db4fded6c2f2da76e89b7582c9b536
cd examples
sbx run ./hello --kit ./gh .
```

The official `hello` workload supplies the sandbox-ready filesystem; `gh` adds the GitHub CLI and its requests. Local sources build on demand ([upstream local example](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/README.md#how-to-get-started), [`hello` recipe](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/examples/hello/hello.dockerfile), [`gh` descriptor](https://github.com/docker/sandbox-kit-spec/blob/5fabb0a260db4fded6c2f2da76e89b7582c9b536/examples/gh/gh.yaml)). Inside the sandbox, `gh --version` checks the overlay and `cat /etc/motd` checks the workload's built content.

The final `.` shares the checkout **read-write**; use a disposable checkout. On the host, `sbx policy ls` shows effective network rules ([workspace boundary](https://docs.docker.com/ai/sandboxes/security/#trust-boundaries), [security guidance](https://docs.docker.com/ai/sandboxes/security/#security-considerations)). This procedure is source-verified, **not locally executed**: `sbx` was unavailable here.

Related: [composition](composition.md) · [permissions and limits](permissions.md) · [compare with Google AX](compare-ax.md).
