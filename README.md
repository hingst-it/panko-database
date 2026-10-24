# panko-database

OpenTofu module behind Panko's `Database` resource: a PostgreSQL database, an
isolated login role, optional extensions, and an owned connection Secret.

Panko's runner fetches this repository at a pinned commit and runs OpenTofu
against it. The `Database` CRD stays in the Panko repository; this module reads
the resource snapshot directly, without a separate field mapping.

## Contract with the runner

| Input | Source |
| --- | --- |
| `var.namespace`, `var.name` | Identity of the `Database` resource. |
| `var.resource` | Immutable snapshot (`metadata` and `spec`) from `resource.auto.tfvars.json`. `metadata.identity`, if present, is the original UID of a restored resource and replaces `uid` for naming; the owner reference keeps the real `uid`. |
| `PGPASSWORD` | PostgreSQL administrator password from the restricted admin Secret. |
| `KUBE_HOST`, `KUBE_TOKEN`, `KUBE_CLUSTER_CA_CERT_DATA` | Credentials of the team's `tofu-runner` ServiceAccount. |

The backend is configured by the runner (`kubernetes`, team state namespace).
Providers are installed with `-lockfile=readonly`, so `.terraform.lock.hcl` must
contain checksums for every platform the runner uses.

## Changing the module

```bash
mise exec -- tofu fmt -check
mise exec -- tofu init -backend=false -lockfile=readonly -input=false
mise exec -- tofu validate
```

After a provider change, record checksums for all platforms:

```bash
mise exec -- tofu providers lock -platform=linux_amd64 -platform=linux_arm64 \
  -platform=darwin_amd64 -platform=darwin_arm64
```

A new module version reaches Panko only when its commit SHA is set as
`MODULE_REVISION` of the hook. Every `Database` is then applied once with the
new revision.
