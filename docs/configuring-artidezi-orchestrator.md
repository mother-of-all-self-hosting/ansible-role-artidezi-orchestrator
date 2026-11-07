<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up the ArtiDezi orchestrator

This is an [Ansible](https://www.ansible.com/) role which installs the orchestrator of [ArtiDezi](https://github.com/devture/artidezi) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

ArtiDezi is a self-hosted maintenance agent that pushes the boring buttons: it lets coding agents do routine maintenance on your repositories, such as adapting dependency updates, without giving them your forge credentials.

ArtiDezi has 2 parts. The orchestrator is the trusted one: an HTTP server with a web UI, which holds the forge credentials, hands out work and publishes the results. The workers are not trusted: they run repository code and a coding agent on expendable machines. This role installs only the orchestrator. For the workers, see [ansible-role-artidezi-worker](https://github.com/mother-of-all-self-hosting/ansible-role-artidezi-worker).

See the project's [documentation](https://github.com/devture/artidezi/blob/main/docs/README.md) to learn what ArtiDezi does and why it might be useful to you.

## Prerequisites

- A forge account for the bot on Forgejo or GitHub, with an access token or a GitHub App. [Forge account](https://github.com/devture/artidezi/blob/main/docs/user-guide/installing.md#forge-account) in ArtiDezi's documentation describes what it needs.
- A hostname of its own. ArtiDezi cannot be served under a path prefix.
- A reverse proxy with TLS, which preserves the `Host` header. Workers refuse plain HTTP, and the web UI's session cookie needs HTTPS. By default, the role sets up [Traefik](https://traefik.io/) labels for it (see [below](#adjusting-the-reverse-proxy-optional)).
- At least 1 worker, set up separately with [ansible-role-artidezi-worker](https://github.com/mother-of-all-self-hosting/ansible-role-artidezi-worker) or by hand.

## Adjusting the playbook configuration

To enable the ArtiDezi orchestrator with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# artidezi_orchestrator                                                #
#                                                                      #
########################################################################

artidezi_orchestrator_enabled: true

artidezi_orchestrator_hostname: artidezi.example.com

########################################################################
#                                                                      #
# /artidezi_orchestrator                                               #
#                                                                      #
########################################################################
```

After adjusting the hostname, make sure that its DNS record points to your server.

`artidezi_orchestrator_uid` and `artidezi_orchestrator_gid` need values too, if your playbook does not provide them.

The orchestrator also needs its ArtiDezi configuration and the secrets it refers to, as described below.

### Set the ArtiDezi configuration

`artidezi_orchestrator_configuration_extension_yaml` holds ArtiDezi's configuration, in its own format.

Add the following configuration to your `vars.yml` file, and adjust it to your setup:

```yaml
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
  projects: {}
```

Then add your repositories under `projects`. See ArtiDezi's [example configuration](https://github.com/devture/artidezi/blob/main/etc/example.yaml) and [Projects and instances](https://github.com/devture/artidezi/blob/main/docs/user-guide/projects.md).

The value is merged over the role's small default (`artidezi_orchestrator_configuration_default_yaml`). As it is an Ansible variable, Jinja in it gets rendered, so text that only looks like Jinja needs escaping.

On every run, the role validates the configuration with ArtiDezi itself before installing it. To see the full error of a rejected one, run `bin/check-config` under `artidezi_orchestrator_base_path` on the server.

### Set the secrets

`artidezi_orchestrator_secrets` maps credential names to values.

Add the following configuration to your `vars.yml` file, with the names your configuration uses:

```yaml
artidezi_orchestrator_secrets:
  admin-token: ADMIN_TOKEN_HERE
  worker-1-token: WORKER_1_TOKEN_HERE
  forgejo-token: FORGE_BOT_TOKEN_HERE
```

It needs the credentials the orchestrator itself reads: the admin token (`instance.admin_token_ref`, `admin-token` by default), each worker's `token_ref` and each forge's `credential_ref`. Model endpoints' credentials (e.g. the OpenAI API key) belong on the workers, not here.

Tokens need at least 32 characters and must differ from each other. Generate them with e.g. `openssl rand -hex 32`.

### Enabling the web UI (optional)

The web UI is off by default. To enable it, declare its accounts under `ui_access` in the configuration, as [Enabling the web UI](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md#enabling-the-web-ui) in ArtiDezi's documentation describes, and add the following configuration to your `vars.yml` file:

```yaml
artidezi_orchestrator_ui_enabled: true
```

### Checking GitHub App access before installing (optional)

For forges with `provider: github`, the role can run ArtiDezi's `github-check` on the candidate configuration, which verifies the App's access to the configured repositories. To have it run, add the following configuration to your `vars.yml` file, listing the forges to check by their keys under `forges`:

```yaml
artidezi_orchestrator_github_check_forges:
  - github
```

GitHub being unreachable fails the run.

### Adjusting the reverse proxy (optional)

By default, the role adds Traefik labels to the container, which route the whole of `artidezi_orchestrator_hostname` to it.

If Traefik runs in another container network, connect the orchestrator to it and tell Traefik which network to use, by adding the following configuration to your `vars.yml` file:

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

Then route `https://artidezi.example.com` to `http://127.0.0.1:8080`, preserving the `Host` header.

Workers upload artifacts of up to 1 GiB through it by default, so check its request size limits.

### Running several instances (optional)

An instance is an inventory host. In the MASH playbook, [add another one for the same server](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/running-multiple-instances.md), with its own `mash_playbook_service_identifier_prefix` and `mash_playbook_service_base_directory_name_prefix`.

All names and paths derive from `artidezi_orchestrator_identifier` and `artidezi_orchestrator_base_path`, so only these need to differ between instances, along with `artidezi_orchestrator_hostname` and, if you publish the port, `artidezi_orchestrator_container_http_host_bind_port`.

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. Among them:
  - `artidezi_orchestrator_lease_ttl_seconds`: how long a worker's lease stays valid without a heartbeat, and `artidezi_orchestrator_tick_seconds`: how often forge maintenance runs
  - `artidezi_orchestrator_log_level`: the `RUST_LOG` filter
  - `artidezi_orchestrator_process_extra_arguments_custom` for other arguments to `artidezi serve`, and `artidezi_orchestrator_container_extra_arguments_custom` for other arguments to `docker create`

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, the orchestrator becomes available at the URL specified with `artidezi_orchestrator_hostname`. With the configuration above, the service is hosted at `https://artidezi.example.com`. `https://artidezi.example.com/healthz` answers `ok` when the orchestrator runs; this says nothing about the workers or the forges.

Point each worker at that URL (`orchestrator_url` in its configuration, or `artidezi_worker_orchestrator_url` with [ansible-role-artidezi-worker](https://github.com/mother-of-all-self-hosting/ansible-role-artidezi-worker)), with the token you listed for it. Use the same ArtiDezi release on the orchestrator and the workers, and upgrade them together.

To create a task from any machine, with ArtiDezi's CLI:

```sh
export ARTIDEZI_URL=https://artidezi.example.com
export ARTIDEZI_ADMIN_TOKEN_FILE=/path/to/admin-token
artidezi task create --project myapp --instructions 'Add a small script and test it.'
```

[Tasks, assessments and review](https://github.com/devture/artidezi/blob/main/docs/user-guide/tasks.md) in ArtiDezi's documentation describes what happens next.

### Changing the configuration

The role restarts the orchestrator whenever its configuration or secrets change. Running attempts survive a brief restart, as workers keep retrying until their lease (`artidezi_orchestrator_lease_ttl_seconds`) expires. After that, an attempt only reruns if its worker pool sets `lost_attempt_retries`.

Before an upgrade, take a backup: schema migrations run when the new version starts.

### Resetting a web UI password

The web UI has no account administration. To reset a password, run the following command on the server, as root, adjusting the container's name to `artidezi_orchestrator_identifier` (e.g. `mash-artidezi-orchestrator`):

```sh
printf '%s\n' 'A_NEW_PASSWORD_OF_16_CHARACTERS' | docker exec -i artidezi-orchestrator \
  artidezi ui-user reset-password --config /etc/artidezi/artidezi.yaml --id alice --password-file /dev/stdin
```

[Permissions and password recovery](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md#permissions-and-password-recovery) in ArtiDezi's documentation has the details.

### Backing up

`artidezi_orchestrator_data_path` holds the SQLite database, the artifacts and the Git caches, and has to be on a local filesystem. Back them up together, with the orchestrator stopped, along with the configuration and the secrets. [State, backups and upgrades](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md#state-backups-and-upgrades) in ArtiDezi's documentation has the details.

### Uninstalling

>[!WARNING]
> Uninstalling (`artidezi_orchestrator_enabled: false`) deletes `artidezi_orchestrator_base_path`, and with it the data directory and the secrets.

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu artidezi-orchestrator` (or how you/your playbook named the service, e.g. `mash-artidezi-orchestrator`).

To change how much it logs, set `artidezi_orchestrator_log_level`.

### Work does not progress

[Investigate stuck work](https://github.com/devture/artidezi/blob/main/docs/user-guide/operations.md#investigate-stuck-work) in ArtiDezi's documentation lists what to check when a task stays queued or a PR is never evaluated. With the web UI enabled, its Workers page shows each worker's last contact, current task and state.
