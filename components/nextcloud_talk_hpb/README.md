<!--
SPDX-FileCopyrightText: 2026 Travis Wichert

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Nextcloud Talk high-performance backend (HPB)

The playbook can install and configure the [Nextcloud Talk high-performance backend](https://github.com/strukturag/nextcloud-spreed-signaling) for you.

The backend consists of the signaling server and the Janus MCU. Clients reach the signaling server through the reverse proxy; Janus carries the media over the host network. The message bus is [NATS](../nats/README.md), installed by its own component.

See the project's [documentation](https://github.com/strukturag/nextcloud-spreed-signaling) to learn what the high-performance backend does and why it might be useful to you.

For details about configuring the [Ansible role for Nextcloud Talk HPB](https://github.com/littlecedar/ansible-role-mash-nextcloud-talk-hpb), you can check them via:
- 🌐 [the role's documentation](https://github.com/littlecedar/ansible-role-mash-nextcloud-talk-hpb/blob/main/docs/configuring-nextcloud-talk-hpb.md) online
- 📁 `roles/galaxy/nextcloud_talk_hpb/docs/configuring-nextcloud-talk-hpb.md` locally, if you have [fetched the Ansible roles](../../docs/installing.md)

## Topology

It is recommended to run the backend on its own server, separate from Nextcloud. Janus
carries the media and needs host networking plus the `20000-40000/udp` range, so isolating it
keeps the media path off the Nextcloud server's CPU and bandwidth. The signaling server
reaches Nextcloud over HTTPS; the two servers only need a network path between them.

Two dependencies are local to the HPB server, because Docker networks do not cross servers:

- The `nats` component (`nats_enabled: true`), enabled on the HPB server. The signaling
  container joins the bus over the NATS Docker network, which exists only on that host.
- The reverse proxy that serves the signaling hostname. The playbook-managed
  [Traefik](../traefik/README.md) on the HPB server provides its network; a proxy on
  another server cannot join it. Point the signaling hostname's DNS at the HPB server.

Set `nextcloud_talk_hpb_nextcloud_url` explicitly on the HPB server, as in the example
below. The automatic derivation from the Nextcloud component only applies when Nextcloud is
enabled on the same host, because host variables do not cross hosts.

## Dependencies

This service requires the following services:

- [NATS](../nats/README.md) message bus on the HPB server, enabled with `nats_enabled: true` in that server's `vars.yml`. The role validates that the NATS container network exists on the target host. This service reads `nats_token`, `nats_identifier` and `nats_container_network` from it, joins its network, and orders the signaling unit after the NATS unit. The token is not repeated here.
- A reverse proxy and its external network on the HPB server. The playbook-managed [Traefik](../traefik/README.md) provides it by default; without it, set `nextcloud_talk_hpb_proxy_network` to the network your proxy uses.
- An external TURN server (coturn) configured for TURN REST credentials, sharing `nextcloud_talk_hpb_turn_static_auth_secret`. This service never deploys it.
- A Nextcloud instance with the Talk app, on its own server. Set `nextcloud_talk_hpb_nextcloud_url` to its base URL; the automatic derivation only applies when Nextcloud is enabled on the same host.

## Configuration

To enable the service, add the following configuration to the HPB server's
`inventory/host_vars/<host>/vars.yml` (not the Nextcloud server's) and re-run the
[installation](../../docs/installing.md) process:

```yaml
########################################################################
#                                                                      #
# nextcloud_talk_hpb                                                   #
#                                                                      #
########################################################################

nextcloud_talk_hpb_enabled: true

# The NATS message bus used by the backend. The nats component must be in use too.
nats_enabled: true

# The public hostname of the signaling server, as entered in Nextcloud Talk's HPB settings.
nextcloud_talk_hpb_hostname: talk.example.com

# The HPB server's public IP address. Janus announces it as its 1:1 NAT mapping.
nextcloud_talk_hpb_public_ip: 203.0.113.10

# The Nextcloud base URL. Set it explicitly on the HPB server, with the Nextcloud hostname.
nextcloud_talk_hpb_nextcloud_url: https://cloud.example.com

# Put a strong secret below, generated with `openssl rand -hex 32` or in another way.
# Register the same value with `occ talk:signaling:add` (see below).
nextcloud_talk_hpb_signaling_secret: ''

# Put the TURN REST shared secret below. It must match the TURN server's secret
# (coturn `--use-auth-secret`).
nextcloud_talk_hpb_turn_static_auth_secret: ''

########################################################################
#                                                                      #
# /nextcloud_talk_hpb                                                  #
#                                                                      #
########################################################################
```

`nextcloud_talk_hpb_signaling_hash_key`, `..._signaling_block_key` and
`..._signaling_internal_shared_secret_key` need no inventory value. They are derived from
`mash_playbook_generic_secret_key`, so they stay stable across runs; set a strong
playbook-wide value for that key, because the session keys inherit its strength.
`nextcloud_talk_hpb_nextcloud_url` is derived only when Nextcloud is enabled on the same
host with a hostname. On the recommended separate-server deployment, set it explicitly to
the Nextcloud base URL, as in the example above.

Janus gathers TURN relay candidates from `nextcloud_talk_hpb_janus_turn_server_host` and
`..._turn_server_port` (default `127.0.0.1:3478`). Set both when coturn runs elsewhere. If
`172.29.0.0/24` is already in use on the host, set `nextcloud_talk_hpb_egress_subnet` and
`nextcloud_talk_hpb_egress_gateway` together.

While the Traefik component is in use, the signaling container gets Traefik labels for
`https://<nextcloud_talk_hpb_hostname>/`, using the playbook's entrypoints and certificate
resolver. Without a playbook-managed Traefik, set `nextcloud_talk_hpb_proxy_network` and make
your proxy send `X-Real-IP` and `X-Forwarded-For`.

## Register the signaling server with Nextcloud

After a deployment, run this on the Nextcloud server to register the signaling server with its container:

```sh
docker exec -u www-data <nextcloud_identifier>-server php occ talk:signaling:add \
  "https://<nextcloud_talk_hpb_hostname>" "<nextcloud_talk_hpb_signaling_secret>" --verify
docker exec -u www-data <nextcloud_identifier>-server php occ talk:signaling:list
```

Then add the external TURN and STUN servers in the Talk admin settings, using the same TURN
secret. The role does not modify Nextcloud.

## Networks and ports

| Port | Scope | Service |
| --- | --- | --- |
| `443` | public | reverse proxy (WebSocket upgrade for `/spreed`) |
| `20000-40000/udp` | public | Janus RTP/RTCP; Janus runs with host networking |
| `3478/tcp+udp` and the relay range | public | operator's coturn |
| `127.0.0.1:8081` | loopback | signaling HTTP; also the Janus event handler target |
| `127.0.0.1:8088` | loopback | Janus HTTP API |
| `172.29.0.1:8188` | egress subnet | Janus WebSocket transport |
| `4222` | NATS network only (no host publish) | NATS |

The egress network `nextcloud-talk-hpb-egress` (`172.29.0.0/24`) connects the signaling
container to the host-networking Janus container. The host firewall must drop `8081`, `8088`
and `8188` on public interfaces.

## Operating

```sh
journalctl -u nextcloud-talk-hpb-signaling -f
journalctl -u nextcloud-talk-hpb-janus -f
```

Both containers log through systemd only. Setting `nextcloud_talk_hpb_enabled: false` and
re-running the playbook stops and removes the two units, the role's egress network and the
base path. The reverse proxy network, the NATS unit and the NATS network are removed by their
own components.

## Troubleshooting

Refer to the **Operating** and **Troubleshooting** sections of the role's documentation
[online](https://github.com/littlecedar/ansible-role-mash-nextcloud-talk-hpb/blob/main/docs/configuring-nextcloud-talk-hpb.md)
or in `roles/galaxy/nextcloud_talk_hpb/docs/configuring-nextcloud-talk-hpb.md`.

## Related services

- [NATS](../nats/README.md) — A lightweight publish/subscribe message broker
- [Nextcloud](../nextcloud/README.md) — Popular collaboration solution
- [Traefik](../traefik/README.md) — A container-aware reverse-proxy server