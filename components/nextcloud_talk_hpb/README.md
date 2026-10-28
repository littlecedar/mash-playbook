<!--
SPDX-FileCopyrightText: 2026 Travis Wichert

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Nextcloud Talk High-Performance Backend (HPB)

The playbook can install and configure the [Nextcloud Talk High-Performance Backend (HPB)](https://github.com/nextcloud/talk-hpb) for you.

Nextcloud Talk HPB is a high-performance backend for Nextcloud Talk that provides WebRTC signaling, a Janus WebRTC gateway, and a NATS message broker. It enables high-quality real-time communication for Nextcloud Talk.

See the project's [documentation](https://github.com/nextcloud/talk-hpb) to learn what HPB does and why it might be useful to you.

For details about configuring the [Ansible role for Nextcloud Talk HPB](https://forgejo.littlecedar.net/littlecedar/ansible-role-mash-nextcloud-talk-hpb), you can check them via:

- 🌐 [the role's documentation](https://forgejo.littlecedar.net/littlecedar/ansible-role-mash-nextcloud-talk-hpb) online
- 📁 `roles/galaxy/nextcloud_talk_hpb/docs/configuring-nextcloud-talk-hpb.md` locally, if you have [fetched the Ansible roles](../../docs/installing.md)

## Dependencies

This service requires the following other services:

- [Nextcloud](../nextcloud/README.md) — for Talk integration
- [Docker](../docker/README.md) — container runtime
- [Traefik](../traefik/README.md) reverse-proxy server (for exposing the signaling server)
- (optional) [coTURN](https://github.com/coturn/coturn) for WebRTC STUN/TURN relaying

## Architecture

Nextcloud Talk HPB consists of 3 separate systemd services:

1. **signaling** (`nextcloud_talk_hpb_signaling_identifier`): the signaling server, which is the only component exposed to the internet (via Traefik)
2. **janus** (`nextcloud_talk_hpb_janus_identifier`): the Janus WebRTC gateway (MCU), reachable only over the internal container network
3. **nats** (`nextcloud_talk_hpb_nats_identifier`): the NATS message broker used internally by the signaling server

The identifiers are prefixed with `mash-` by default, so the services are named `mash-nextcloud_talk_hpb-signaling.service`, `mash-nextcloud_talk_hpb-janus.service` and `mash-nextcloud_talk_hpb-nats.service`.

## Configuration

To enable this service, add the following configuration to your `vars.yml` file and re-run the [installation](../../docs/installing.md) process:

```yaml
########################################################################
#                                                                      #
# nextcloud_talk_hpb                                                   #
#                                                                      #
########################################################################

# Enable Nextcloud first
nextcloud_enabled: true

nextcloud_talk_hpb_enabled: true

# The hostname at which the signaling server is served (via Traefik)
nextcloud_talk_hpb_signaling_hostname: talk.example.com

# Put a strong secret below, generated with `pwgen -s 64 1` or in another way
nextcloud_talk_hpb_signaling_sessions_hash_key: ""

# The Nextcloud instance(s) allowed to use this signaling server.
# The shared_secret must match the one you enter in the Nextcloud Talk admin settings.
nextcloud_talk_hpb_signaling_backends:
  - id: nextcloud
    url: "https://cloud.example.com"
    # Put a strong secret below, generated with `pwgen -s 64 1` or in another way
    shared_secret: "PUT-A-STRONG-SHARED-SECRET-HERE"

########################################################################
#                                                                      #
# /nextcloud_talk_hpb                                                  #
#                                                                      #
########################################################################
```

### Basic Configuration

The following variables control the basic HPB configuration. Their defaults are set by the playbook in [`wiring.yml`](wiring.yml):

```yaml
# Master toggle for the whole role
nextcloud_talk_hpb_enabled: true

# Base configuration
nextcloud_talk_hpb_identifier: "{{ mash_playbook_service_identifier_prefix }}nextcloud_talk_hpb"
nextcloud_talk_hpb_base_path: "{{ mash_playbook_base_path }}/{{ mash_playbook_service_base_directory_name_prefix }}nextcloud_talk_hpb"
nextcloud_talk_hpb_uid: "{{ mash_playbook_uid }}"
nextcloud_talk_hpb_gid: "{{ mash_playbook_gid }}"

# The internal container network shared by the three containers
nextcloud_talk_hpb_container_network: "{{ nextcloud_talk_hpb_identifier }}"
```

### Signaling Server Configuration

```yaml
nextcloud_talk_hpb_signaling_enabled: "{{ nextcloud_talk_hpb_enabled }}"
nextcloud_talk_hpb_signaling_identifier: "{{ nextcloud_talk_hpb_identifier }}-signaling"
nextcloud_talk_hpb_signaling_base_path: "{{ nextcloud_talk_hpb_base_path }}/signaling"
nextcloud_talk_hpb_signaling_hostname: talk.example.com
nextcloud_talk_hpb_signaling_http_port: 8080
nextcloud_talk_hpb_signaling_sessions_hash_key: ""
# Optional, 16, 24 or 32 bytes; generated with `pwgen -s 32 1`
nextcloud_talk_hpb_signaling_sessions_block_key: ""
```

The signaling container is connected to the reverse-proxy's network by the playbook (see [`wiring.yml`](wiring.yml)), so Traefik can reach it without publishing a host port.

### Janus Configuration

```yaml
nextcloud_talk_hpb_janus_enabled: "{{ nextcloud_talk_hpb_enabled }}"
nextcloud_talk_hpb_janus_identifier: "{{ nextcloud_talk_hpb_identifier }}-janus"
nextcloud_talk_hpb_janus_base_path: "{{ nextcloud_talk_hpb_base_path }}/janus"
nextcloud_talk_hpb_janus_stun_server: ""
nextcloud_talk_hpb_janus_stun_port: 3478
```

The Janus RTP UDP port range (`nextcloud_talk_hpb_janus_rtp_udp_min_port` … `nextcloud_talk_hpb_janus_rtp_udp_max_port`) is not published on the host by default, because media is expected to be relayed via a TURN server. Set `nextcloud_talk_hpb_janus_container_rtp_host_bind_interface` to publish it.

### NATS Configuration

```yaml
nextcloud_talk_hpb_nats_enabled: "{{ nextcloud_talk_hpb_enabled }}"
nextcloud_talk_hpb_nats_identifier: "{{ nextcloud_talk_hpb_identifier }}-nats"
nextcloud_talk_hpb_nats_base_path: "{{ nextcloud_talk_hpb_base_path }}/nats"
nextcloud_talk_hpb_nats_port: 4222
nextcloud_talk_hpb_nats_log_level: warn
```

### TURN Configuration

Calls generally do not work reliably without a reachable TURN server. Point the signaling server at yours:

```yaml
nextcloud_talk_hpb_signaling_turn_secret: "SAME-SECRET-AS-ON-THE-TURN-SERVER"
nextcloud_talk_hpb_signaling_turn_servers: "turn:turn.example.com:3478?transport=udp,turn:turn.example.com:3478?transport=tcp"
nextcloud_talk_hpb_signaling_turn_api_key: "AN-API-KEY-FOR-THE-MCU"
```

The role also has `nextcloud_talk_hpb_signaling_turn_secret_auto`, `nextcloud_talk_hpb_signaling_turn_servers_auto` and `nextcloud_talk_hpb_signaling_turn_api_key_auto` hooks for a coTURN role which fills them in automatically.

### Full Configuration with coTURN

```yaml
nextcloud_enabled: true
traefik_enabled: true

nextcloud_talk_hpb_enabled: true
nextcloud_talk_hpb_signaling_hostname: talk.example.com
nextcloud_talk_hpb_signaling_sessions_hash_key: ""
nextcloud_talk_hpb_signaling_backends:
  - id: nextcloud
    url: "https://cloud.example.com"
    shared_secret: "PUT-A-STRONG-SHARED-SECRET-HERE"

# Janus with STUN/TURN
nextcloud_talk_hpb_janus_stun_server: "turn.example.com"
nextcloud_talk_hpb_janus_stun_port: 5349
```

## Tags Available

The role provides these tags:

- `setup-nextcloud-talk-hpb`
- `install-nextcloud-talk-hpb`
- `setup-all`
- `install-all`

## Installation Commands

```sh
# Install (and start) the whole playbook
just install-all

# Or only this component
just run-tags install-nextcloud-talk-hpb,start

# Without `just`
ansible-playbook -i inventory/hosts setup.yml --tags=install-nextcloud-talk-hpb,start
```

## Validation Steps

After installation, verify:

```sh
# Check all services are running
systemctl status mash-nextcloud_talk_hpb-signaling.service
systemctl status mash-nextcloud_talk_hpb-janus.service
systemctl status mash-nextcloud_talk_hpb-nats.service

# Check Docker containers
docker ps | grep mash-nextcloud_talk_hpb

# Verify Traefik routing
curl -I https://talk.example.com
```

## Nextcloud Configuration

In the Nextcloud Talk admin settings, under **High-performance backend**, add the signaling server with the URL `https://talk.example.com/standalone-signaling/` and the shared secret you set for that backend in `nextcloud_talk_hpb_signaling_backends`. Nextcloud should then report the signaling server as connected.

Also under **Talk → Advanced settings**:

```text
Turn server: turn:talk.example.com:443?transport=udp
Turn server: turn:talk.example.com:443?transport=tcp
STUN server: stun:talk.example.com:443
```

## Security Considerations

1. The signaling server is exposed to the internet — ensure Traefik TLS is configured
2. Use `nextcloud_talk_hpb_signaling_sessions_hash_key` and a strong `shared_secret` per backend for session security
3. Configure coTURN for WebRTC media relay instead of publishing Janus' RTP ports
4. Keep all container images updated
5. Monitor logs for suspicious activity

## Troubleshooting

Common issues:

1. **Services not starting**: check Docker is running, verify the configuration
2. **Traefik not routing**: verify `nextcloud_talk_hpb_signaling_hostname` is DNS-resolvable
3. **Janus RTP issues**: configure the STUN/TURN server properly
4. **Nextcloud can't connect**: verify the backend's `shared_secret` matches the one in the Nextcloud Talk admin settings

## Migration from Standalone HPB

If migrating from a standalone HPB installation:

1. Stop the existing HPB services
2. Back up configuration and data
3. Run the playbook with `nextcloud_talk_hpb_enabled: true`
4. Update the Nextcloud Talk configuration
5. Verify functionality
6. Remove the old standalone installation
