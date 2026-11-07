<!--
SPDX-FileCopyrightText: 2018-2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (12/Bookworm or newer)
  - **Ubuntu** (22.04 or newer)
- `root` access on the computer which Molecule runs against
- [Ansible](https://www.ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default; others need it installed (something like `apt-get install python3`)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## Scenarios

Currently there is 1 testing scenario available.

### `default`

Tests 2 instances of the orchestrator on 1 machine, the second as another inventory host, the way the MASH playbook runs several instances.
Only their identifiers, hostnames and ports differ.

The scenario checks that, for each of them:

- the orchestrator runs the image version that `defaults/main.yml` pins, read-only, without capabilities and as the configured user, on its own network and the additional one;
- `/healthz` and the web UI's `/login` answer;
- the admin API accepts the token from `artidezi_orchestrator_secrets` and refuses a wrong one;
- the configuration and the secrets have the expected owners and modes, and the data ends up in the instance's data directory;
- a configuration which ArtiDezi rejects fails the run, without changing the installed configuration or the secrets, and leaves no candidate file behind;
- a secret which is gone from `artidezi_orchestrator_secrets` loses its file, and asks for a restart.

Then, that neither orchestrator restarts over 45 seconds, and that uninstalling the second one leaves the first one running.

As that uninstalls an instance, run `molecule converge` before running `molecule verify` again.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Ubuntu 24.04
MOLECULE_DISTRO=ubuntu2404 molecule test --scenario-name default

# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default

# Debian 12
MOLECULE_DISTRO=debian12 molecule test --scenario-name default
```
