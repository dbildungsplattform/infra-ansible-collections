# before_rollout_checks

Reusable role with safety checks to run at the start of a rollout, locally and on AWX.

| Check | Task file | Default | What it does |
|---|---|---|---|
| Input validation | `asserts.yml` | always | Checks types of all inputs and that required values are set |
| Scope | `scope_checks.yml` | on | Local-run guard, 1Password token, single stage, single project |
| Stale branch | `stale_branch.yml` | on | Fails if the rolled-out revision is behind `origin/main` |
| Rollout lock | `lock_check.yml`, `lock_acquire.yml`, `lock_release.yml` | **off** | S3-based lock that blocks rollouts by others |

Checks run in this order: asserts, scope, stale branch, lock.

## Requirements

- `ansible-core` with the `amazon.aws` collection (lock only) and `boto3`/`botocore` on the controller or execution environment.
- `git` on the machine running the check (AWX execution environment included).
- Hosts must define the variables `project` (project name) and `stage`.

## What the checks do

### Scope checks
- **Local guard:** a non-AWX run (`awx_job_id` undefined) without check mode (`-C`) fails unless the host is in a group listed in `before_rollout_checks_local_allowed_groups`. Override in an emergency with `-e "{force_local_rollout: True}"`.
- **1Password token:** `OP_SERVICE_ACCOUNT_TOKEN` must be a non-empty string (disable with `before_rollout_checks_op_token_required: false`).
- **Single stage / single project:** all hosts in the play must share the same `stage` and the same `project`.

These checks inspect `ansible_play_hosts_all`, so the role must be applied in the play that targets the real rollout hosts. In a `localhost`-only play they always pass.

### Stale branch check
Fails if the revision being rolled out is behind `main` on the remote.
- **Local:** `git fetch` then `git rev-list --left-right --count HEAD...origin/main` in the playbook directory.
- **AWX:** the project directory has no `.git`, so the role makes a temporary blobless bare clone of `before_rollout_checks_repo_url` (removed afterwards) and compares against it. The compared ref is `awx_job_scm_branch` if the job overrides the branch, otherwise `awx_project_revision`.
- Every run prints the compared range and the ahead/behind counts.

Known limitation (AWX, branch override): the branch tip is read at check time. A push between the project sync and the check can make a stale job pass.

### Rollout lock (optional)
An object in an S3 bucket (`before_rollout_checks_lock_object`, default `.lock.json`) containing JSON:

```json
{"locked_by": "user@example.com", "locked_since": "2026-01-01T10:00:00Z", "locked_approx_until": "", "comment": ""}
```

- **Exists and held by someone else:** the rollout fails, showing who, since when and the comment.
- **Held by the current user:** the rollout continues.
- **Acquire:** `before_rollout_checks_lock_acquire: true` writes the lock if none exists (`overwrite: never`) and verifies it afterwards.
- **Release:** the owner can release. A forced release (`before_rollout_checks_lock_force: true`) works regardless of the owner and logs who forced it.

The lock owner is `before_rollout_checks_user`: `awx_user_email` on AWX, otherwise `op_credentials.OP_EMAIL`, otherwise `$USER`. Locks set locally and on AWX only match if these resolve to the same value, so override the variable to use one identifier everywhere.

## Variables

All variables are prefixed with `before_rollout_checks_`.

| Variable | Default | Description |
|---|---|---|
| `scope_enabled` | `true` | Run the scope checks |
| `stale_enabled` | `true` | Run the stale branch check |
| `lock_enabled` | `false` | Check the rollout lock |
| `local_allowed_groups` | `[]` | Inventory groups where local non-check rollouts are allowed. **Set per project** |
| `force_local_rollout` | `force_local_rollout` or `false` | Skip the local guard |
| `op_token_required` | `true` | Require `OP_SERVICE_ACCOUNT_TOKEN` |
| `remote_name` | `origin` | Git remote (local runs) |
| `remote_branch` | `main` | Branch to compare against |
| `repo_url` | `""` | Clone URL, **required on AWX** |
| `user` | see above | Identity stored in and compared with the lock |
| `s3_endpoint_url` | `dbcmetrics_ionos_tf_state_s3_bucket_url` | S3 endpoint |
| `s3_bucket` | `dbcmetrics_ionos_tf_state_s3_bucket_name` | Bucket |
| `s3_access_key` | `aws_access_key_id` | Access key |
| `s3_secret_key` | `aws_secret_access_key` | Secret key |
| `lock_object` | `.lock.json` | Object key of the lock |
| `lock_acquire` | `false` | Acquire the lock if none exists |
| `lock_approx_until` | `""` | Informational expected end of the lock |
| `lock_comment` | `""` | Informational reason |
| `lock_force` | `false` | Release regardless of the owner |

Required values (checked in `asserts.yml`):
- Always: the `*_enabled` flags are booleans.
- Scope enabled: `project` and `stage` on every host, `local_allowed_groups` is a list of strings, and the 1Password token when required.
- Stale enabled: `remote_name`, `remote_branch`; `repo_url` when running on AWX.
- Lock enabled: `s3_endpoint_url`, `s3_bucket`, `s3_access_key`, `s3_secret_key`, `lock_object`, `user`.

Booleans may be real booleans or strings such as `true`/`false`/`yes`/`no` (as passed via `-e` or AWX surveys).

## Usage examples

### 1. Standard rollout play (scope + stale check)
```yaml
- name: Roll out
  hosts: all
  roles:
    - role: before_rollout_checks
      vars:
        before_rollout_checks_local_allowed_groups: [project_dev, project_infradev]
        before_rollout_checks_repo_url: "https://{{ before_rollout_checks_git_user }}:{{ before_rollout_checks_git_token }}@github.com/org/repo.git"
    - role: my_application
```

### 2. As the first tasks of an existing play
```yaml
- name: Roll out
  hosts: all
  pre_tasks:
    - name: Run the pre-rollout checks
      ansible.builtin.include_role:
        name: before_rollout_checks
      vars:
        before_rollout_checks_local_allowed_groups: [project_dev]
        before_rollout_checks_repo_url: "{{ project_repo_url }}"
  roles:
    - my_application
```

### 3. Project with the lock enabled
```yaml
- name: Roll out
  hosts: all
  roles:
    - role: before_rollout_checks
      vars:
        before_rollout_checks_local_allowed_groups: [project_dev]
        before_rollout_checks_repo_url: "{{ project_repo_url }}"
        before_rollout_checks_lock_enabled: true
        before_rollout_checks_s3_endpoint_url: "{{ myproject_s3_url }}"
        before_rollout_checks_s3_bucket: "{{ myproject_s3_bucket }}"
        before_rollout_checks_s3_access_key: "{{ myproject_s3_access_key }}"
        before_rollout_checks_s3_secret_key: "{{ myproject_s3_secret_key }}"
```

### 4. Project without 1Password, scope checks only
```yaml
- name: Roll out
  hosts: all
  roles:
    - role: before_rollout_checks
      vars:
        before_rollout_checks_stale_enabled: false
        before_rollout_checks_op_token_required: false
        before_rollout_checks_local_allowed_groups: [project_dev]
```

### 5. Lock and unlock playbooks (use `tasks_from`)
```yaml
- name: Acquire rollout lock
  hosts: localhost
  gather_facts: false
  tasks:
    - ansible.builtin.include_role:
        name: before_rollout_checks
      vars:
        before_rollout_checks_scope_enabled: false
        before_rollout_checks_stale_enabled: false
        before_rollout_checks_lock_enabled: true
        before_rollout_checks_lock_acquire: true
        before_rollout_checks_lock_comment: "Database migration"

- name: Release rollout lock
  hosts: localhost
  gather_facts: false
  tasks:
    - ansible.builtin.include_role:
        name: before_rollout_checks
        tasks_from: lock_release
```
Force release from the command line: `-e before_rollout_checks_lock_force=true`. On AWX, add a boolean survey question with that variable name to the unlock job template.

This repository contains ready-made versions: `playbook-lock.yml`, `playbook-unlock.yml` and `playbook-pre-rollout-checks.yml` (targets `rollout_hosts`, default `all`).

## Running the lock and unlock playbooks directly

You can run the playbooks directly from this collection by FQCN,
for example `ansible-playbook dbildungscloud.infra.lock`. The role is then referenced by its FQCN.

`playbook-lock.yml` and `playbook-unlock.yml` run on `localhost` and only talk to S3, so no inventory hosts are needed. They enable the lock themselves, so `before_rollout_checks_lock_enabled` doesn't have to be set.
### Prerequisites
- The S3 variables resolve to non-empty values: `before_rollout_checks_s3_endpoint_url`, `_s3_bucket`, `_s3_access_key` and `_s3_secret_key`. By default they come from `dbcmetrics_ionos_tf_state_s3_bucket_url`, `dbcmetrics_ionos_tf_state_s3_bucket_name`, `aws_access_key_id` and `aws_secret_access_key`. Override them for other projects.
- Local runs load `/localhome/ansible_vault_secrets.yml` if it exists. Add `--ask-vault-pass` or `--vault-password-file` if it is vault-encrypted.
- Your identity (`before_rollout_checks_user`) must be set. Locally it is `op_credentials.OP_EMAIL` (from the vault secrets), falling back to `$USER`. On AWX it is `awx_user_email`.
- The `amazon.aws` collection and `boto3` are installed.
### Lock
```bash
ansible-playbook dbildungscloud.infra.lock \
  -e before_rollout_checks_lock_comment="Database migration" \
  -e before_rollout_checks_lock_approx_until="2026-01-01T12:00:00Z" \
  -i hosts --limit "enrichment_infradev" --diff --vault-id enrichment@prompt
```
- Both `-e` options are optional and informational. They are shown to others who are blocked.
- If no lock exists, it is created. If you already hold it, the run is a no-op.
- If someone else holds it, the playbook fails and shows who, since when and the comment.
### Unlock
```bash
ansible-playbook dbildungscloud.infra.unlock
```
- **Your own lock:** it is deleted.
- **No lock:** nothing happens, no error.
- **Someone else's lock:** the playbook fails and tells you how to force.
Force release regardless of the owner, for example when the owner is unavailable:
```bash
ansible-playbook dbildungscloud.infra.unlock -e before_rollout_checks_lock_force=true
```
The run logs `<you> forces release of lock held by <owner>`.
### Passing project-specific S3 settings
```bash
ansible-playbook dbildungscloud.infra.lock \
  -e before_rollout_checks_s3_endpoint_url=https://s3.example.com \
  -e before_rollout_checks_s3_bucket=my-project-state \
  -e before_rollout_checks_s3_access_key="$ACCESS_KEY" \
  -e before_rollout_checks_s3_secret_key="$SECRET_KEY"
```
Prefer vault variables over command-line secrets, since the latter end up in shell history.
### Inspecting a lock
Use the AWS CLI against the same bucket:
```bash
aws --endpoint-url https://s3.example.com s3 cp s3://my-project-state/.lock.json -
```

## AWX setup

- **Project:** enable "Update revision on job launch".
- **Repo access:** set `before_rollout_checks_repo_url`. For a GitHub PAT, create a custom credential type that injects the username and token as extra vars (for example `before_rollout_checks_git_user` / `before_rollout_checks_git_token`) and build the URL from them. Use a dedicated read-only token (fine-grained, Contents: read). The clone task uses `no_log`.
- **Lock:** map the S3 variables to credentials of your project, for example via a custom credential type.
- **Branch override:** enable "Allow branch override" on the job template if you roll out feature branches. The check then compares `awx_job_scm_branch`.
