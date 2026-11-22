# azure-lab-workflows

Reusable GitHub Actions workflows for Terraform solution repos that deploy to Azure with OIDC
and keep state in an Azure Storage account.

| Workflow | Purpose |
|---|---|
| `tf_plan.yml` | On a PR: plan each changed solution, comment the plan, save the plan file |
| `tf_apply.yml` | On push to the default branch: apply the plan that was reviewed on the merged PR |
| `tf_apply_manual.yml` | Recovery: plan and apply one solution from the default branch |

A *solution* is any directory containing `main.tf`. A PR touching files below it triggers a plan for it.

## Using them

1. Copy the files from [`examples/`](examples) into the solution repo's `.github/workflows/`.
2. Edit the `uses:` owner/repo and `@v1` ref, and the state inputs.
3. Tag this repo (`git tag v1 && git push --tags`) and pin callers to the tag.
4. If this repo is private: Settings > Actions > General > Access > "Accessible from repositories owned by the user".

### State inputs (all three workflows)

| Input | Required | Meaning |
|---|---|---|
| `state_resource_group` | yes | Resource group of the state storage account |
| `state_storage_account` | yes | Storage account name |
| `state_container` | yes | Blob container |
| `state_key_strip_prefix` | no | Removed from the solution path when deriving the key |

State key = `<solution path minus strip prefix>.tfstate`. For example, with prefix `terraform/solutions/`,
`terraform/solutions/identity/tf-state` uses key `identity/tf-state.tfstate`, matching existing
`azure-lab-entra` state. `tf_apply_manual.yml` also accepts an explicit `state_key`.

Use the same state and prefix inputs in plan, apply and manual callers.

### Other inputs

- `terraform_version` (default `1.15.8`)
- `tf_plan.yml`: `environment` (default `terraform-plan`)
- `tf_apply.yml` and `tf_apply_manual.yml`: `plan_environment`, `apply_environment` (defaults `terraform-plan`, `terraform-apply`); `tf_apply.yml` also `plan_workflow_file` (default `tf_plan.yml`)

### Caller permissions

Called workflows can't exceed the caller's permissions, so the caller must grant:

- plan: `id-token: write`, `contents: read`, `pull-requests: write`
- apply: `id-token: write`, `contents: read`, `actions: read`, `pull-requests: read`
- manual: `id-token: write`, `contents: read`

## Terraform requirement

Solutions must declare an empty backend; the workflows supply the settings at `terraform init`:

```hcl
terraform {
  backend "azurerm" {}
}
```

Auth uses Entra ID (`use_azuread_auth=true`), so the identity needs **Storage Blob Data Contributor**
on the state container/account. If the storage account is network-restricted, the runner must be able to reach it.

## Secrets and environments

Set in each solution repo (repo or org secrets; callers use `secrets: inherit`):

- `TF_CLIENT_ID`, `TF_TENANT_ID`, `TF_SUBSCRIPTION_ID`

Create environments `terraform-plan` and `terraform-apply` (add required reviewers to `terraform-apply`).

### Federated credentials

Issuer `https://token.actions.githubusercontent.com`, audience `api://AzureADTokenExchange`.
A reusable workflow keeps the **caller's** subject, so credentials are created for the solution repo, one per environment:

```
repo:<owner>/<solution-repo>:environment:terraform-plan
repo:<owner>/<solution-repo>:environment:terraform-apply
```

Or the immutable-ID form if your repo/org uses it: `repo:<owner>@<ownerId>/<repo>@<repoId>:environment:<env>`.
Matching is exact and case-sensitive.

## Notes

- The plan artifact is `tfplan-<solution path with / replaced by ->`, kept 14 days. Apply needs the
  plan run for the PR's head commit to have succeeded; re-run the plan if it expired or the PR changed.
- Plan files can contain secrets; restrict who can read Actions artifacts.
