# sbx-kit-mem0-v3

A [Docker Sandbox Kit](https://github.com/docker/sandbox-kit-spec) (**v3**,
`schemaVersion: "3"`) that adds the [Mem0](https://github.com/mem0ai/mem0)
memory layer to an agent sandbox, pre-wired to a **local Docker Model Runner**
for both the LLM and the embedder — no cloud credentials, no external vector
database.

Ported from the v1/v2 `spec.yaml` grammar to the v3 kit descriptor.

## Files

This is a **mixin** in companion-pair form:

| File | Role |
|---|---|
| `mem0.yaml` | the v3 descriptor — declares capabilities (network policy, agent context, lifecycle hooks) |
| `mem0.dockerfile` | a `FROM scratch` overlay that carries the runtime `ENV` onto the composed image |
| `mem0-context.md` | agent guidance, staged via `agent-context@1` |

## What it does

- Installs `mem0ai[nlp]==2.0.5` (+ the spaCy English model) into the composed
  workload's Python, at sandbox-create time, via `lifecycle@1` install hooks.
- Writes `~/.mem0/config.json` wired to the Docker Model Runner at
  `host.docker.internal:12434` (editable; not overwritten if present).
- Phase-scoped egress: PyPI + GitHub open only during install; the running
  agent may reach the Model Runner.
- No `credential@1` — the DMR `api_key` is the sentinel `"dmr"`, not a secret.

## Prerequisites

- Ensure that you have sbx 0.45.1 installed

```
❯ sbx version
sbx version: v0.45.1 9d79d90ee4c5d297fb3d36b75384e8cea7a4fbcb
```

- [`sbx` CLI](https://docs.docker.com/ai/sandboxes/install/) (supports Kits v3)
- A running Docker daemon with **Docker Model Runner** enabled, serving the
  models named in `mem0.yaml` (`ai/gemma3`, `ai/mxbai-embed-large`)

## Use it

```sh
# Validate the descriptor (fast fail-on-bad-field; builds nothing to export):
docker buildx build . -f mem0.yaml --output type=cacheonly

# Run it composed onto a shell workload (source form — no registry/push needed):
sbx run docker/sbx-kit-shell:1.0.0 --kit . .

# Inside the sandbox, the kit is self-describing:
cat /usr/share/sandbox/kit/mem0/kit.yaml     # the published descriptor
cat /home/agent/.mem0/config.json            # the DMR-wired config
python3 -c "import mem0; print('mem0', mem0.__version__)"
```

## Publish (optional)

```sh
docker buildx build . -f mem0.yaml --platform linux/amd64,linux/arm64 --push \
  -t docker.io/ajeetraina/sbx-kit-mem0:2.0.5 \
  -t docker.io/ajeetraina/sbx-kit-mem0:latest
sbx run docker/sbx-kit-shell:1.0.0 --kit docker.io/ajeetraina/sbx-kit-mem0:2.0.5 .
```

## Notes for adopters

- If your base workload's Python packages differ, the kit still works because
  the install runs against the **composed base** at create time.
- Before adding `requires: ["deb/python3", "deb/python3-pip"]` (currently
  commented in `mem0.yaml`), verify those packages exist in the base's dpkg
  database — an unprovidable requirement refuses composition everywhere:
  ```sh
  docker run --rm docker/sbx-kit-shell:1.0.0 dpkg-query -W -f='${Version} ${Status}\n' python3
  ```

---

Built with the `migrate-kit-to-v3` skill from
[docker/sandbox-kit-spec](https://github.com/docker/sandbox-kit-spec).
