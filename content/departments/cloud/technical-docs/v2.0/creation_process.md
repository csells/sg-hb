# Creating a managed instance

Creating a new [managed instance](./index.md) involves following the steps below.
For basic operations like accessing an instance for these steps, see [managed instances operations](../operations.md) what if there is some text here.

## Prereq

Follow https://github.com/sourcegraph/deploy-sourcegraph-cloud-controller#installation to install `mgv2`

```sh
git clone https://github.com/sourcegraph/deploy-sourcegraph-cloud-dev
cd deploy-sourcegraph-cloud-dev
```

## Steps

> See flow chart https://app.excalidraw.com/s/4Dr1S6qmmY7/2wUSD4kIxRo

```sh
export SLUG=<>
```

```sh
git checkout -b $SLUG/create-instance
```

### Create GCP Project

Open `projects/terraform.tfvars` and create a new entry depending on instance type

```hcl
managed_projects = {
  "slug" = {
    name = "slug_or_custom_display_name"
  }
}
```

```sh
terraform init
terraform apply
```

Notes the output `project_id` of the new project

```sh
export PROJECT_ID=<>
export DOMAIN=<>
```

### Init deployment artifacts

`mgv2 generate` will

- generate the terraform module and apply the terraform module
- generate the kustomization manifests and helm override based on output from the terraform module

```sh
mgv2 generate --project-id $PROJECT_ID --domain $DOMAIN
```

Above command will fail on the first run, follow the prompt to manually apply the terraform module. (or you can just run the command below)

```sh
cd deployments/$PROJECT_ID/terraform
terraform apply
```

Rerun the `generate` command to generate the kustomize manifests and helm overrides

```sh
mgv2 generate --project-id $PROJECT_ID --domain $DOMAIN
```

### Wrapping up

```sh
git add .
git commit -m "$SLUG: create instance"
```

Create a new pull request and merge it

## Troubleshooting

### How do I check my Sourcegraph deployment rollout progress?

Visit ArgoCD at https://argocd-cloud.sgdev.org

### I am unable to apply the terraform module due to permission error.

> `roles/owner` definitely grants excessive permissions, but it would make developing greenfield project much easier and we will revist permissions at a later day.

Ensure the Google Group you belong to is present here https://github.com/sourcegraph/infrastructure/blob/c8b6d3dd3f6312334281d8261523b4d6cad60c8e/gcp/projects/terraform.tfvars#L460-L467. Otherwise, consult [GCP access process](../../../engineering/dev/process/gcp_access_process.md#standard-access-for-permanent-access-to-resources-projects-or-assets) to obtain access.

### Any other questions?

Please reach out to #cloud