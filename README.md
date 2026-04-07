# Meticulous reusable workflows (catalog)

YAML workflows under [`.stable/workflows/`](.stable/workflows/) are meant for import into an organization’s **global workflow catalog** in Meticulous. Each file’s top-level **`name`** matches `global/<name>` when referenced from a pipeline (for example, `workflow: global/git-checkout`).

For pipeline composition patterns—**agent-affinity**, **share-workspace**, **depends-on**, and invocation outputs—see the Meticulous control-plane repo at `design/pipelines.md`.

## Importing a catalog workflow (Git)

Use the Meticulous API to pin a workflow definition from this repository.

- **Method / path:** `POST /api/v1/projects/{project_id}/workflows/catalog/import-git`
- **JSON body (no secrets in the body—only references to stored credentials):**

```json
{
  "repository": "owner/Meticulous-Reusable-Workflows",
  "git_ref": "main",
  "workflow_path": ".stable/workflows/git-checkout.yaml",
  "credentials_path": "path/to/stored-github-app-or-token-secret-in-your-project"
}
```

- **`repository`:** GitHub `owner/name` or URL understood by the API.
- **`git_ref`:** Branch, tag, or commit to resolve.
- **`workflow_path`:** Path to the YAML file in that repo.
- **`credentials_path`:** Project secret that supplies GitHub access for fetch (installation or token as configured in Meticulous)—**never** put tokens in this JSON.

List imported catalog entries (organization-scoped) via `GET /api/v1/workflows/catalog` (see OpenAPI / API docs for pagination).

## Example pipeline: shared workspace + affinity + outputs

After workflows are in the catalog, reference them by `global/<name>` and version. Use **`agent-affinity`** and **`share-workspace: true`** when one job must leave files for the next (for example checkout → build). Order invocations with **`depends-on`**.

**Public (non-secret) values** from an upstream invocation can be passed to downstream **`inputs`** with `${{ workflows.<invocation_id>.outputs.<name> }}` once the platform has recorded those outputs (for example via `met-output var` inside the reusable workflow). **Secret outputs** require an explicit pipeline opt-in before they may appear as plaintext environment in dependent jobs; see `design/workflow-invocation-outputs.md` in the Meticulous control-plane repository.

Example (illustrative only—substitute your catalog names, versions, and vars; do not hardcode registry passwords or tokens):

```yaml
name: build and deploy
agent-affinity:
  default-group: ci
  share-workspace: true
vars:
  GIT_REPO: https://github.com/example/example.git
workflows:
  - name: Checkout
    id: checkout
    workflow: global/git-checkout
    version: "1.0.0"
    inputs:
      repo: "${GIT_REPO}"
  - name: Build image
    id: build
    workflow: global/docker-buildx-push
    version: "1.0.0"
    depends-on: [checkout]
    inputs:
      image_ref: ghcr.io/example/app:ci
      registry_host: ghcr.io
  - name: Deploy
    id: deploy
    workflow: global/run-script
    version: "1.0.0"
    depends-on: [build]
    inputs:
      script_path: deploy/ci_deploy.sh
      shell: bash
```

Inside a reusable workflow, after computing a value you want to expose downstream (same run):

```bash
met-output var IMAGE_URI="${REGISTRY}/${IMAGE}:${TAG}"
```

Downstream **`inputs`** may then use:

```yaml
image: "${{ workflows.build.outputs.image_uri }}"
```

Use `met-output secret ...` only for sensitive material; treat it according to platform policy—**never** log or echo those values.

## Per-workflow prerequisites

Each file under `.stable/workflows/` begins with a short comment block listing **agent prerequisites** (tools, optional Docker, pool hints). Default **`runs-on`** tags prefer **Linux** unless noted.

### Meticulous self-build (validate against homelab stacks)

For a **checkout → four image builds** loop covering **`Dockerfile.agent`**, **`Dockerfile.met-api`**, **`Dockerfile.met-controller`**, and **`Dockerfile.sqlx-migrate`** (same clone / workspace), combine:

| Catalog workflow           | Agent needs |
| -------------------------- | ----------- |
| `global/git-checkout`      | `git`, HTTPS egress to GitHub; optional `GITHUB_TOKEN` in **pipeline secrets** for private repos. |
| `global/setup-rust`        | `curl`, `sh`; writes toolchain under the workspace (`RUSTUP_HOME` / `CARGO_HOME`). Use when running `cargo` *on the agent* (not required if Dockerfiles compile everything). |
| `global/docker-buildx-push` | OCI-capable host: **Docker or Podman** with **buildx**; registry login via `REGISTRY_USERNAME` / `REGISTRY_PASSWORD` from **pipeline secrets**; egress to registry. See the probe snippet the workflow installs under `.meticulous_probe_docker_host.sh`. |
| `global/trivy-scan`         | **`trivy` on PATH**; for `mode: image`, a working Docker/Podman if you set **`registry_host`** so the workflow can `login` with the same pipeline registry secrets before scanning private images. Writes CycloneDX to **`sbom.cdx.json`** (or `cyclonedx_path`) under `METICULOUS_WORKSPACE`; the agent sends that document to the control plane when the job completes. Catalog **1.1.0** adds optional **`registry_host`**. |
| `global/run-script`        | `bash` or `sh`; executes only a **checked-in script path** under the workspace—add a repo script for `argocd`, `kubectl`, or Git ops deploy. |

An illustrative pipeline lives at [`examples/meticulous-self-build.pipeline.yaml`](examples/meticulous-self-build.pipeline.yaml). It chains **checkout → build/push → Trivy image scan** per image so `agent-affinity.share-workspace` stays valid (total `depends-on` order) and each scan can write **`sbom.cdx.json`** without races. Re-import **`trivy-scan`** if your catalog still has **1.0.0**. Adjust `REGISTRY_PREFIX`, `IMAGE_TAG`, and registry inputs to match your environment.

**Run SBOM tab:** `GET /api/v1/runs/{run_id}/sbom` returns the **first** SBOM artifact on the run (by `created_at`); multiple scan jobs in one run each still record SBOMs on their **job** status—see artifacts / job views if you need every image’s document in the UI.

## Crypto and secrets

Catalog YAML and this README use **placeholders** and pipeline **`secrets`** / stored credentials only. Do not commit API keys, passwords, or long-lived tokens into this repository. For encryption and algorithm expectations, follow the Meticulous control-plane crypto rules referenced from `design/workflow-invocation-outputs.md`.
