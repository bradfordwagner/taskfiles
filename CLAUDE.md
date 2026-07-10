# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal library of [go-task](https://taskfile.dev) `Taskfile.yml` snippets (`tasks/*.yml`), each one a standalone
helper for a specific tool (vault, kubectl, helm, terraform, tmux, docker, github, etc). There is no application code
to build — this is tooling/automation for day-to-day CLI workflows, synced into `~/.taskfiles` on the local machine.

## How tasks are invoked

The root `Taskfile.yml` is minimal (just an `ansible-playbook` wrapper for `default`). Individual taskfiles are NOT
included into it — each one is used standalone via task's `-t` flag, run from whatever project directory the user is
currently in:

```sh
task -t ~/.taskfiles/tasks/<file>.yml <task-name> [vars...]
task -t ~/.taskfiles/tasks/vault.yml status
task -t ~/.taskfiles/tasks/kubectl.yml <task> --list   # list tasks in a file
```

Because of this, `{{ .USER_WORKING_DIR }}` (a go-task builtin) refers to the caller's cwd — the project being worked
on — not this repo. Many tasks write/read files relative to `USER_WORKING_DIR` (e.g. `.secrets.sh`, `config.yaml`,
`generate_dockerfile.sh`), not relative to the taskfiles repo itself. Keep this in mind when editing tasks: they are
meant to be dropped into arbitrary project directories, not self-contained to this repo.

Some taskfiles compose others via `includes:` (e.g. `tasks/terraform.yml` and `tasks/github_secrets.yml` both
`includes: {vault: ./vault.yml}`, exposing `vault:status`-style namespaced subtasks).

## Deployment

`playbook.yml` (an Ansible playbook) rsyncs this repo's contents into `~/.taskfiles` — that's the mechanism by which
edits here become available for `task -t ~/.taskfiles/tasks/...` invocations. `tasks/work` is gitignored (local-only
overrides/private tasks, not tracked).

## Linting

CI (`.github/workflows/yaml_lint.yml`) runs `task -t tasks/yaml_lint.yml` (yamllint, 160-char line length) against
the whole repo on every push. Run the same locally before committing:

```sh
task -t tasks/yaml_lint.yml
```

## Vault taskfiles — two distinct targets

`tasks/vault.yml` and `tasks/vault_dev.yml` are easy to confuse — they point at different Vault instances:

- **`tasks/vault.yml`** — the remote/production Vault (`https://bradfordwagner.com:8200`, hardcoded). Assumes the
  caller is already authenticated (`vault login`). Tasks here manage KV secrets and AppRole credentials
  (`kv2_env`, `generate_approle`, `list_approle_accessors`, `delete_approle_accessors`, `token_policies_for_auth_role`).
- **`tasks/vault_dev.yml`** — a local/dev Vault (typically running in a local k8s cluster). Reads its address/token/
  kubeconfig from `~/.vault-addr`, `~/.vault-token`, `~/.vault-kubeconfig`. Owns cluster lifecycle tasks: `init`
  (single key-share/threshold `vault operator init`, writes root token + unseal key to `~/.vault-*`), `unseal`, and
  `reset` (destructively deletes all pods/PVCs in the current kube context, then re-inits and unseals — confirm the
  active kubeconfig context before running).

## Secrets handling convention

Tasks that need to hand secrets to a shell session write them as `export ...` lines to
`{{ .USER_WORKING_DIR }}/.secrets.sh` (see `vault.yml:kv2_env`, `vault.yml:generate_approle`,
`github_secrets.yml`), meant to be `source`d by the caller afterward. `vault.yml:clean_env_file` removes it.
