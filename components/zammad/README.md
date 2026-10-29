<!--
SPDX-FileCopyrightText: 2026 Travis Wichert

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Zammad

The playbook can install and configure [Zammad](https://zammad.org/) for you.

Zammad is a web-based, open source helpdesk and customer support (ticketing) system.

See the project's [documentation](https://docs.zammad.org/) to learn what Zammad does and why it might be useful to you.

For details about configuring the [Ansible role for Zammad](https://forgejo.littlecedar.net/littlecedar/ansible-role-mash-zammad), you can check them via:

- 🌐 [the role's documentation](https://forgejo.littlecedar.net/littlecedar/ansible-role-mash-zammad/src/branch/main/docs/configuring-zammad.md) online
- 📁 `roles/galaxy/zammad/docs/configuring-zammad.md` locally, if you have [fetched the Ansible roles](../../docs/installing.md)

## Dependencies

This service requires the following other services:

- [Postgres](../postgres/README.md) database
- [Valkey](../valkey/README.md) data-store; see [below](#configure-valkey) for details about installation
- [Traefik](../traefik/README.md) reverse-proxy server

## Configuration

To enable this service, add the following configuration to your `vars.yml` file and re-run the [installation](../../docs/installing.md) process:

```yaml
########################################################################
#                                                                      #
# zammad                                                               #
#                                                                      #
########################################################################

zammad_enabled: true

zammad_hostname: zammad.example.com

########################################################################
#                                                                      #
# /zammad                                                              #
#                                                                      #
########################################################################
```

**Note**: hosting Zammad under a subpath (by configuring the `zammad_path_prefix` variable) is not possible due to Zammad's limitations; it must be `/`.

With the [Postgres](../postgres/README.md) service enabled, the playbook creates the database and user and wires the connection for you. Valkey is not wired automatically; see [below](#configure-valkey).

### Configure Valkey

Zammad requires a Valkey data-store to work. This playbook supports it, and you can set up a Valkey instance by enabling it on `vars.yml`.

If Zammad is the sole service which requires Valkey on your server, it is fine to set up just a single Valkey instance. However, **it is not recommended if there are other services which require it, because sharing the Valkey instance has security concerns and possibly causes data conflicts**, as described on the [documentation for configuring Valkey](../valkey/README.md). In this case, you should install a dedicated Valkey instance for each of them.

If you are unsure whether you will install other services along with Zammad or you have already set up services which need Valkey (such as [Nextcloud](../nextcloud/README.md), [PeerTube](../peertube/README.md), and [Funkwhale](../funkwhale/README.md)), it is recommended to install a Valkey instance dedicated to Zammad.

*See [below](#setting-up-a-shared-valkey-instance) for an instruction to install a shared instance.*

#### Setting up a dedicated Valkey instance

To create a dedicated instance for Zammad, you can follow the steps below:

1. Adjust the `hosts` file
2. Create a new `vars.yml` file for the dedicated instance
3. Edit the existing `vars.yml` file for the main host

*Refer to [this page](../../docs/running-multiple-instances.md) for details about configuring multiple instances of Valkey on the same server.*

##### Adjust `hosts`

At first, you need to adjust `inventory/hosts` file to add a supplementary host for Zammad.

The content should be something like below. Make sure to replace `mash.example.com` with your hostname and `YOUR_SERVER_IP_ADDRESS_HERE` with the IP address of the host, respectively. The same IP address should be set to both, unless the Valkey instance will be served from a different machine.

```ini
[mash_servers]
[mash_servers:children]
mash_example_com

[mash_example_com]
mash.example.com ansible_host=YOUR_SERVER_IP_ADDRESS_HERE
mash.example.com-zammad-deps ansible_host=YOUR_SERVER_IP_ADDRESS_HERE
…
```

`mash_example_com` can be any string and does not have to match with the hostname.

You can just add an entry for the supplementary host to `[mash_example_com]` if there are other entries there already.

##### Create `vars.yml` for the dedicated instance

Then, create a new directory where `vars.yml` for the supplementary host is stored. If `mash.example.com` is your main host, name the directory as `mash.example.com-zammad-deps`. Its path therefore will be `inventory/host_vars/mash.example.com-zammad-deps`.

After creating the directory, add a new `vars.yml` file inside it with a content below. It will have running the playbook create a `mash-zammad-valkey` instance on the new host, setting `/mash/zammad-valkey` to the base directory of the dedicated Valkey instance.

```yaml
# This is vars.yml for the supplementary host of Zammad.

---

########################################################################
#                                                                      #
# Playbook                                                             #
#                                                                      #
########################################################################

# Put a strong secret below, generated with `pwgen -s 64 1` or in another way
mash_playbook_generic_secret_key: ''

# Override service names and directory path prefixes
mash_playbook_service_identifier_prefix: 'mash-zammad-'
mash_playbook_service_base_directory_name_prefix: 'zammad-'

########################################################################
#                                                                      #
# /Playbook                                                            #
#                                                                      #
########################################################################

########################################################################
#                                                                      #
# valkey                                                               #
#                                                                      #
########################################################################

valkey_enabled: true

########################################################################
#                                                                      #
# /valkey                                                              #
#                                                                      #
########################################################################
```

##### Edit the main `vars.yml` file

Having configured `vars.yml` for the dedicated instance, add the following configuration to `vars.yml` for the main host, whose path should be `inventory/host_vars/mash.example.com/vars.yml` (replace `mash.example.com` with yours).

```yaml
########################################################################
#                                                                      #
# zammad                                                               #
#                                                                      #
########################################################################

# Add the base configuration as specified above

# Point Zammad to its dedicated Valkey instance
zammad_redis_hostname: mash-zammad-valkey

# Make sure the Zammad containers are connected to the container network of the dedicated Valkey service (mash-zammad-valkey)
zammad_container_additional_networks_custom:
  - "mash-zammad-valkey"

# Make sure the Zammad init and Rails services start after the dedicated Valkey service (mash-zammad-valkey.service)
zammad_init_systemd_required_services_list_custom:
  - "mash-zammad-valkey.service"
zammad_railsserver_systemd_required_services_list_custom:
  - "mash-zammad-valkey.service"

########################################################################
#                                                                      #
# /zammad                                                              #
#                                                                      #
########################################################################
```

Running the installation command will create the dedicated Valkey instance named `mash-zammad-valkey`.

#### Setting up a shared Valkey instance

If you host only Zammad on this server, it is fine to set up a single shared Valkey instance.

To install the single instance and hook Zammad to it, add the following configuration to `inventory/host_vars/mash.example.com/vars.yml`:

```yaml
########################################################################
#                                                                      #
# valkey                                                               #
#                                                                      #
########################################################################

valkey_enabled: true

########################################################################
#                                                                      #
# /valkey                                                              #
#                                                                      #
########################################################################

########################################################################
#                                                                      #
# zammad                                                               #
#                                                                      #
########################################################################

# Add the base configuration as specified above

# Point Zammad to the shared Valkey instance
zammad_redis_hostname: "{{ valkey_identifier }}"

# No zammad_container_additional_networks_custom entry is needed for the shared instance:
# the playbook connects the containers to its network automatically (the entry above triggers it).

# Make sure the Zammad init and Rails services start after the shared Valkey service (mash-valkey.service)
zammad_init_systemd_required_services_list_custom:
  - "{{ valkey_identifier }}.service"
zammad_railsserver_systemd_required_services_list_custom:
  - "{{ valkey_identifier }}.service"

########################################################################
#                                                                      #
# /zammad                                                              #
#                                                                      #
########################################################################
```

Running the installation command will create the shared Valkey instance named `mash-valkey`.

If your Valkey instance is protected with a password, override `zammad_redis_url` (e.g. `redis://:PASSWORD@mash-valkey:6379`).

### Elasticsearch

Zammad runs its own Elasticsearch container by default. Elasticsearch requires the host's `vm.max_map_count` kernel parameter to be at least `262144`:

```sh
sysctl -w vm.max_map_count=262144
```

Persist the value in a file such as `/etc/sysctl.d/99-zammad.conf` so that it survives a reboot; the role does not set it.

To run Zammad without the bundled Elasticsearch (with reduced search quality), add the following configuration to your `vars.yml` file:

```yaml
zammad_elasticsearch_enabled: false
```

### Memcached

Zammad runs its own Memcached container by default; it can be disabled with `zammad_memcached_enabled: false`.

## Installation

If you have decided to install the dedicated Valkey instance for Zammad, make sure to run the [installing](../../docs/installing.md) command for the supplementary host (`mash.example.com-zammad-deps`) first, before running it for the main host (`mash.example.com`).

Note that running the `just` commands for installation (`just install-all` or `just setup-all`) automatically takes care of the order. See [here](../../docs/running-multiple-instances.md#1-adjust-hosts) for more details about it.

To install only this service, run `just install-service zammad`.

The first start runs the `zammad-init` service, which sets up the database schema and can take a while to finish.

## Usage

After running the command for installation, the Zammad instance becomes available at the URL specified with `zammad_hostname`. With the configuration above, the service is hosted at `https://zammad.example.com`.

To get started, open the URL with a web browser and complete the setup wizard to create the first administrator account.

## Troubleshooting

The Zammad containers run as several systemd services. Inspect the main ones with:

```sh
journalctl -fu mash-zammad-railsserver.service
systemctl status 'mash-zammad-*'
```

Refer to [this section](https://forgejo.littlecedar.net/littlecedar/ansible-role-mash-zammad/src/branch/main/docs/configuring-zammad.md#troubleshooting) on the role's documentation for details about the other services.