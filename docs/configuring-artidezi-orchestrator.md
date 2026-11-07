<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up the ArtiDezi orchestrator

This is an [Ansible](https://www.ansible.com/) role which installs the orchestrator of [ArtiDezi](https://github.com/devture/artidezi) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd unit.

The orchestrator is the trusted part of ArtiDezi: an HTTP server with a web UI, which holds the forge credentials and hands out work.
Workers run repository code and a coding agent, so they belong on other, expendable machines. This role doesn't install them.
[Installing](https://github.com/devture/artidezi/blob/main/docs/user-guide/installing.md) in ArtiDezi's documentation describes them, and the forge account the orchestrator needs.

## Prerequisites

- A forge account for the bot, and its token (or GitHub App).
- A reverse proxy with TLS, preserving the `Host` header. Workers refuse plain HTTP, and the web UI's session cookie needs HTTPS.
  The role can set up [Traefik](https://traefik.io/) for it (see below).
- A hostname of its own: ArtiDezi can't be served under a path prefix.

## Adjusting the playbook configuration

To enable the orchestrator with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# artidezi-orchestrator                                                #
#                                                                      #
########################################################################

artidezi_orchestrator_enabled: true

artidezi_orchestrator_hostname: artidezi.example.com

artidezi_orchestrator_secrets:
  admin-token: A_LONG_RANDOM_STRING
  worker-1-token: ANOTHER_LONG_RANDOM_STRING
  forgejo-token: THE_BOT_TOKEN

artidezi_orchestrator_configuration_extension_yaml: |
  instance:
    bot_identity:
      name: ArtiDezi
      email: artidezi@example.com
  model_endpoints:
    openai:
      provider: openai
      auth_mode: openai_api_key
      credential_ref: openai
  worker_pools:
    maintenance:
      harness_sandbox: disabled
      harnesses: [codex]
      workers:
        worker-1: {token_ref: worker-1-token}
  agent_profiles:
    codex:
      harness: codex
      harness_version_constraint: '*'
      model: REPLACE_ME
      model_endpoint: openai
      worker_pool: maintenance
  forges:
    forgejo:
      provider: forgejo
      api_base_url: https://forgejo.example.com
      credential_ref: forgejo-token
      bot_login: artidezi
  projects:
    myapp:
      # See etc/example.yaml in ArtiDezi's repository

########################################################################
#                                                                      #
# /artidezi-orchestrator                                               #
#                                                                      #
########################################################################
```

After adjusting the hostname, make sure that its DNS record points to your server.

### Configuration

The role doesn't model ArtiDezi's configuration: `artidezi_orchestrator_configuration_extension_yaml` holds it, in ArtiDezi's own format.
See [its example](https://github.com/devture/artidezi/blob/main/etc/example.yaml) and [Configuration and operation](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md).

It is merged over a small default from the role (`artidezi_orchestrator_configuration_default_yaml`), with the settings that depend on how the role runs the container: `schema_version`, `instance.name`, `instance.data_dir` and `instance.admin_token_ref`.
Dictionaries are merged recursively; lists are replaced.
As it is an Ansible variable, it can use Jinja, and text in it that looks like Jinja (e.g. in a project's instructions) needs escaping.

To manage all of it yourself, override `artidezi_orchestrator_configuration` with a dictionary.
`instance.data_dir` has to stay `artidezi_orchestrator_container_data_path`.

On every run, the role renders a candidate file and validates it with ArtiDezi's `validate-config`, in the configured image, in a container without network access.
A rejected configuration fails the run before the installed configuration or the secrets change, also when it's a new image that rejects an unchanged configuration.
The error names the problem, but not the configuration lines around it, as those may hold password hashes.
`bin/check-config [FILE]` under `artidezi_orchestrator_base_path` prints all of it on the host, for the installed configuration or for another file.

The orchestrator reads its configuration and secrets only when it starts, so changing them restarts it.

### Secrets

`artidezi_orchestrator_secrets` becomes 1 file per name in `artidezi_orchestrator_secrets_path`, readable by root and the orchestrator's group.
A name you remove loses its file.

It needs the credentials that the orchestrator itself reads:

- `admin-token` (or whatever `instance.admin_token_ref` says);
- each worker's `token_ref`;
- each forge's `credential_ref`.

The role refuses to install when one of them is missing, and when the admin and worker tokens are shorter than 32 characters or share a value, as these would keep the orchestrator from starting.
Generate them with e.g. `openssl rand -hex 32`.
Model endpoints' credentials belong on the workers, not here.

Keep the values in Ansible Vault, or similar.

### GitHub forges (optional)

To have `artidezi github-check` verify a GitHub App's access to its repositories before a configuration gets installed, list the forges:

```yaml
artidezi_orchestrator_github_check_forges:
  - github
```

It needs the secrets, so a failed check leaves the new secrets in place next to the old configuration.
GitHub being unreachable fails the run.

### Web UI (optional)

The web UI is off by default.
To enable it, declare the accounts in the configuration, as [Enabling the web UI](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md#enabling-the-web-ui) describes, and add:

```yaml
artidezi_orchestrator_ui_enabled: true
```

The configuration then holds password hashes. The role keeps them out of its output, and the file is readable only by root and the orchestrator's group.

### Reverse proxy

By default, the role adds Traefik labels to the container, routing the whole of `artidezi_orchestrator_hostname` to it.
If Traefik uses another container network, connect the orchestrator to it and tell Traefik which one to use:

```yaml
artidezi_orchestrator_container_additional_networks_custom:
  - traefik
artidezi_orchestrator_container_labels_traefik_docker_network: traefik
```

To use another reverse proxy, disable the labels and publish the port on the host:

```yaml
artidezi_orchestrator_container_labels_traefik_enabled: false
artidezi_orchestrator_container_http_host_bind_port: "127.0.0.1:8080"
```

Workers upload their artifacts through the reverse proxy, up to 1 GiB each by default, so make sure that its request size limits and timeouts allow for yours.

### Running several instances

An instance is an inventory host. In the MASH playbook, [add another one for the same server](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/running-multiple-instances.md), with its own `mash_playbook_service_identifier_prefix` and `mash_playbook_service_base_directory_name_prefix`.

Names and paths all derive from `artidezi_orchestrator_identifier` and `artidezi_orchestrator_base_path`, so only these need to differ, along with:

- `artidezi_orchestrator_hostname`;
- `artidezi_orchestrator_container_http_host_bind_port`, if you publish the port.

### Hardening

The container runs as `artidezi_orchestrator_uid`, without capabilities, with a read-only root filesystem and `no-new-privileges`.
It can write only to the data directory and to a tmpfs at `/tmp` (`artidezi_orchestrator_container_tmp_size`).
The configuration and the secrets are mounted read-only.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

`https://artidezi.example.com/healthz` answers `ok` when the orchestrator runs. It says nothing about workers or forges.

Point each worker's `orchestrator_url` at `https://artidezi.example.com`, with its token.
Use the same release on the orchestrator and the workers, and upgrade them together.

To create a task from any machine:

```sh
export ARTIDEZI_URL=https://artidezi.example.com
export ARTIDEZI_ADMIN_TOKEN_FILE=/path/to/admin-token
artidezi task create --project myapp --instructions 'Add a small script and test it.'
```

To reset the password of a web UI account, on the host:

```sh
printf '%s\n' 'A_NEW_PASSWORD_OF_16_CHARACTERS' | docker exec -i artidezi-orchestrator \
  artidezi ui-user reset-password --config /etc/artidezi/artidezi.yaml --id alice --password-file /dev/stdin
```

The role restarts the orchestrator without waiting for work in progress, so [finish or hold outstanding work](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md#state-backups-and-upgrades) before upgrading.

### Backups

`artidezi_orchestrator_data_path` holds the database, the artifacts and the Git caches. Back them up together, with the orchestrator stopped.
[State, backups and upgrades](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md#state-backups-and-upgrades) in ArtiDezi's documentation has the details.

>[!WARNING]
> Uninstalling (`artidezi_orchestrator_enabled: false`) deletes `artidezi_orchestrator_base_path`, and with it the data directory.

## Troubleshooting

Log in to the server with SSH and run `journalctl -fu artidezi-orchestrator` (`artidezi_orchestrator_identifier`, or however your playbook named it, e.g. `mash-artidezi-orchestrator`).
To change how much it logs, set `artidezi_orchestrator_log_level` (`RUST_LOG`).

[Investigate stuck work](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md#investigate-stuck-work) in ArtiDezi's documentation lists what to check when a task doesn't progress.
