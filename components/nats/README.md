<!--
SPDX-FileCopyrightText: 2026 Travis Wichert

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# NATS

The playbook can install and configure [NATS](https://nats.io/) for you.

NATS is a lightweight publish/subscribe message broker. This component runs the standalone
server: a single container on an internal network, no JetStream persistence, no clustering,
and one bearer token that clients present.

See the project's [documentation](https://docs.nats.io/) to learn what NATS does and why it
might be useful to you.

For details about configuring the [Ansible role for NATS](https://github.com/littlecedar/ansible-role-mash-nats), you can check them via:

- 🌐 [the role's documentation](https://github.com/littlecedar/ansible-role-mash-nats/blob/main/docs/configuring-nats.md) online
- 📁 `roles/galaxy/nats/docs/configuring-nats.md` locally, if you have [fetched the Ansible roles](../../docs/installing.md)

## Dependencies

This service requires no other services.

## Configuration

To enable the service, add the following configuration to `inventory/host_vars/<host>/vars.yml`
and re-run the [installation](../../docs/installing.md) process:

```yaml
########################################################################
#                                                                      #
# nats                                                                 #
#                                                                      #
########################################################################

nats_enabled: true

########################################################################
#                                                                      #
# /nats                                                                #
#                                                                      #
########################################################################
```

The role enables itself by default, so the playbook keeps the service disabled: a component
is only installed when the inventory enables it in a host's `host_vars`.

`nats_token` needs no inventory value. It is derived from `mash_playbook_generic_secret_key`,
so it is stable across runs and differs per playbook deployment.

## Usage

The server listens on port `4222` inside its own internal Docker network and publishes no host
port, so it is reachable only from containers attached to that network. There is no web
interface and no metrics endpoint.

A service that consumes NATS attaches the network with its own
`<service>_container_additional_networks_custom` list, connects to
`<nats identifier>:4222` and presents `nats_token`. Its systemd unit should also order itself
after this one through `<service>_systemd_wanted_services_list_custom`.

## Troubleshooting

Refer to the **Operating** and **Troubleshooting** sections of the role's documentation
[online](https://github.com/littlecedar/ansible-role-mash-nats/blob/main/docs/configuring-nats.md)
or in `roles/galaxy/nats/docs/configuring-nats.md`.

## Related services

- [Redis](../redis/README.md) — In-memory data store used by millions of developers as a database, cache, streaming engine, and message broker
- [Valkey](../valkey/README.md) — A flexible distributed key-value datastore that is optimized for caching and other realtime workloads
