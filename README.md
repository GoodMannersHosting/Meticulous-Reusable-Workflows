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

## Crypto and secrets

Catalog YAML and this README use **placeholders** and pipeline **`secrets`** / stored credentials only. Do not commit API keys, passwords, or long-lived tokens into this repository. For encryption and algorithm expectations, follow the Meticulous control-plane crypto rules referenced from `design/workflow-invocation-outputs.md`.
