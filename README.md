# multi-repository-setup

This repository is the caller for [Litch](https://github.com/toconaskevin/litch). Litch reads a YAML file and brings a GitHub repository to that state: access, rulesets, repository settings, variables, secrets, CI workflows, and `CODEOWNERS`. The workflow in this repository checks out the configs and calls the `toconaskevin/litch` composite action. Litch writes the result into each target repository.

Each file in `configs/` describes one repository:

- `configs/single-app.yaml` uses the `single-app` kind for `toconaskevin/single-app-repo-for-litch`. It publishes one workflow, `ci`, from `templates/ci.yml`.
- `configs/multi-app.yaml` uses the `multi-app` kind for `toconaskevin/multi-app-repo-for-litch`. The repository holds two apps: `api` (`ci-api`) and `web` (`ci-web`).
- `configs/custom.yaml` uses the `custom01` kind for `toconaskevin/custom-repo-for-litch`. That kind does not publish workflows.

`kinds/custom01.yaml` supplies `custom01`. A file with the same `metadata.name` replaces the kind shipped inside Litch. This one is based on `single-app`, requires `access`, `rulesets`, and `codeowners`, defaults the secret provider to Azure, sets `delete_branch_on_merge` and disables merge commits, and forbids `actions.workflows`.

`templates/` holds the CI workflow files. Litch copies a template to `.github/workflows/<name>.yml` in the target repository and leaves the file contents unchanged.

## OIDC

Secrets live in Azure Key Vault. The workflow does not store an Azure client secret.

`.github/workflows/litch.yml` runs on `workflow_dispatch` with `plan`, `apply`, or `drift`, and on a daily schedule that runs `drift` against every file under `configs/`. An empty `config` input on a manual run also covers every file; a path runs that file only. The schedule uses the default branch, so the Azure federated credential subject must match `refs/heads/main` (including owner and repository IDs when GitHub issues immutable OIDC subjects).

The job requests `id-token: write`, so GitHub Actions can mint an OIDC token for the run. Litch exchanges that token with Azure AD for an access token. The audience is `api://AzureADTokenExchange`. The exchange uses `AZURE_TENANT_ID` and `AZURE_CLIENT_ID`, then reads secrets from `AZURE_KEY_VAULT_URL`. Those three values are repository variables. GitHub API access uses a GitHub App: Litch reads `litch-github-app-client-id`, `litch-github-app-id`, `litch-github-app-installation-id`, and `litch-github-app-pem` from the vault and mints an installation token for the run.
