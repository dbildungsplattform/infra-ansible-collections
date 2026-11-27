# generate_kubeconfigs

Creates ServiceAccounts, (Cluster)RoleBindings and token based kubeconfigs for users and uploads them to 1Password
(personal kubeconfigs are encrypted with the user's SSH public key). Resources and 1Password items of absent users are removed.
On AWX the admin kubeconfig is fetched first via `get_ionos_kubeconfig`.

## Variables

See `defaults/main.yml` and `meta/argument_specs.yml`. Defaults fall back to the equivalent inventory variables
(`kubeconfig`, `present_users`, `cluster_name`, `proxy_vault`, `kubeconfigs_vault`, `op_credentials`, ...), so existing inventories work unchanged.

| Variable | Required | Description |
|---|---|---|
| `generate_kubeconfigs_kubeconfig` | yes | Admin kubeconfig |
| `generate_kubeconfigs_cluster_name` | yes | Cluster name |
| `generate_kubeconfigs_op_service_account_token` | yes | 1Password service account token |
| `generate_kubeconfigs_kubeconfigs_vault` | yes | Vault for personal kubeconfigs |
| `generate_kubeconfigs_present_users` / `_absent_users` / `_regenerate_users` | no | User lists |
| `generate_kubeconfigs_kubernetes_roles` | no | ClusterRoles to create |
| `generate_kubeconfigs_proxy_url`, `_proxy_headers` | no | Proxy for API calls |
| `generate_kubeconfigs_proxy_domain`, `_proxy_port` | no | Proxy written into the kubeconfigs |
| `generate_kubeconfigs_proxy_vault`, `_infra_vault` | no | Further 1Password vaults |
| `generate_kubeconfigs_sshkeys_dir` | no | Default `{{ playbook_dir }}/files/sshkeys` |

## Permissions

Each entry in `permissions.kubernetes` of a user needs `hosts` and `role` (a ClusterRole).
`namespace` is optional and can be a string or a list:

- not set: a `ClusterRoleBinding` is created (cluster-wide access)
- `namespace: dev` or `namespace: [dev, test]`: a `RoleBinding` is created in every listed namespace.
  The ServiceAccount (and its token) lives in the first namespace, so each user still gets one kubeconfig per cluster.

```yaml
permissions:
  developer:
    kubernetes:
      - hosts: "stage-dev"
        role: developer
        namespace:
          - dev
          - test
```

## Example

```yaml
- name: Generate personalized Kubeconfigs
  hosts: stages
  gather_facts: false
  any_errors_fatal: true
  pre_tasks:
    - name: Load vault secrets in case of local rollout
      ansible.builtin.include_vars:
        file: "/localhome/ansible_vault_secrets.yml"
      when: "'/localhome/ansible_vault_secrets.yml' is file"
  roles:
    - role: dbildungscloud.infra.generate_kubeconfigs
```

Requires the collections `kubernetes.core`, `community.general` and `dbildungscloud.onepwd`.
