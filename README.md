# Ansible for VCF Automation CCI

This repository manages a small, declarative VMware Cloud Foundation Automation
CCI environment. The roles intentionally use the CCI custom resources, rather
than similarly named upstream Kubernetes objects.

## Prerequisites

* Ansible Core 2.15 or newer and Python's Kubernetes client.
* The `vcf` CLI, logged-in endpoint details, and permission to create the listed
  CCI resources.
* A kubeconfig made current by the VCF CLI and access to the target CCI API.

Install the collection with `ansible-galaxy collection install -r requirements.yml`.
For development, install `ansible-lint` and `yamllint` too.

## API contract

The manifests follow the resource definitions in the
[Broadcom CCI API documentation](https://developer.broadcom.com/xapis/cci-api/latest/api-docs.html):

| Resource | API and kind | Scope / ownership | Required role inputs |
| --- | --- | --- | --- |
| Project | `project.cci.vmware.com/v1alpha2`, `Project` | cluster | `name` |
| VPC | `networking.cci.vmware.com/v1alpha1`, `VPC` | project namespace | `regionName`, `vpcClassName` |
| Supervisor Namespace | `infrastructure.cci.vmware.com/v1alpha2`, `SupervisorNamespace` | project namespace | `regionName`, `className` |
| VM | `vmoperator.vmware.com/v1alpha3`, `VirtualMachine` | Supervisor Namespace | `className`, `imageName`, `storageClass` |
| Address binding | `networking.cci.vmware.com/v1alpha1`, `AddressBinding` | Supervisor Namespace | `vpcRef`, `workloadRef` |

An AddressBinding deliberately omits an address so that CCI allocates one from
the selected VPC. Its returned status is available as
`vcf_address_binding_statuses`. VM cloud-init is stored in a same-namespace
Secret and referenced through `spec.bootstrap.cloudInit.rawCloudConfig`; the
example contains no password, key, or token.

Every list item accepts `metadata` (including `labels` and `annotations`),
`spec`, and `manifest_overrides`. The last value is recursively merged over the
whole generated object and is intended for API additions not yet represented by
the role. Site-specific region, class, image, and storage names in the example
must be replaced with values advertised by the target deployment.

## Variables and secrets

Standard Ansible precedence applies: role defaults are lowest, then inventory,
play variables/`vars_files`, and extra variables. Consequently callers can keep
the reusable example and override deployment-specific values from inventory or
`-e`. See the [Ansible precedence rules](https://docs.ansible.com/ansible/latest/reference_appendices/general_precedence.html).

The context role defaults `vcf_context_api_token` from `VCF_API_TOKEN`. Supported
secret approaches are:

```bash
export VCF_API_TOKEN='...'
ansible-playbook site.yml -e vcf_context_endpoint=https://example \
  -e vcf_context_tenant_name=my-tenant

# Or put vcf_context_api_token in an encrypted vars/vault.yml, then:
ansible-playbook site.yml -e @vars/vault.yml --ask-vault-pass
```

An extra variable (`-e vcf_context_api_token=...`) also works but can leak via
shell history and is discouraged. Token validation and the token-bearing create
task use `no_log`.

## Create

Edit or override `vars/resources.yml`, then run:

```bash
ansible-playbook site.yml
```

The context is inspected first and only created if missing. Set
`vcf_context_state=replace` to delete and recreate it. Resources are reconciled
in ownership/dependency order: Project, VPC, Supervisor Namespace, VM, then
AddressBinding.

## Destroy

```bash
ansible-playbook destroy.yml
```

The same declarations are removed in reverse order. The CLI context remains by
default; add `-e vcf_destroy_context=true` to remove it after the resources.
Deletion is idempotent through `kubernetes.core.k8s` state handling.

## Validation

```bash
yamllint .
ansible-lint
ansible-playbook --syntax-check site.yml
ansible-playbook --syntax-check destroy.yml
```
