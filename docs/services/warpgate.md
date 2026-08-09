<!--
SPDX-FileCopyrightText: 2026 MASH project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Warpgate

The playbook can install and configure [Warpgate](https://warpgate.tech/), a secure SSH gateway for teams and organizations, for you.

Warpgate is a smart SSH, HTTPS, and MySQL bastion host that provides secure access to your servers and services. It features:

- Docker-based deployment
- Systemd service management
- Support for Postgres, MySQL, and SQLite backends
- Automatic integration with Authentik (OIDC) if present in the playbook

See the project's [documentation](https://warpgate.tech/docs/) to learn what Warpgate does and why it might be useful to you.

## Dependencies

This service requires the following other services:

- [Postgres](postgres.md) database server (for the default configuration)
- [Traefik](traefik.md) reverse-proxy server (for HTTPS access)

## Prerequisites

This service requires the following Ansible collection:

- [community.docker](https://github.com/ansible-collections/community.docker) — for Docker image management

Install it via:

```sh
ansible-galaxy collection install community.docker
```

## Configuration

To enable this service, add the following configuration to your `vars.yml` file and re-run the [installation](../installing.md) process:

```yaml
########################################################################
#                                                                      #
# warpgate                                                             #
#                                                                      #
########################################################################

warpgate_enabled: true

warpgate_hostname: warpgate.example.com

# Generate a strong password for the Warpgate database
warpgate_database_password: ""

# If using Authentik integration, provide your OIDC credentials
# warpgate_sso_client_id: ""
# warpgate_sso_client_secret: ""

########################################################################
#                                                                      #
# /warpgate                                                            #
#                                                                      #
########################################################################
```

### Extending the configuration

There are some additional things you may wish to configure about Warpgate.

Take a look at:

- The [Warpgate role](https://forgejo.littlecedar.net/littlecedar/ansible-role-mash-warpgate/)'s [`defaults/main.yml`](https://forgejo.littlecedar.net/littlecedar/ansible-role-mash-warpgate/src/branch/main/defaults/main.yml) for additional variables that you can customize via your `vars.yml` file.

## Usage

After running the installation command, the Warpgate service becomes available at the URL specified with `warpgate_hostname`. With the configuration above, the service is hosted at `https://warpgate.example.com`.

### Ports

Warpgate exposes the following ports by default:

- SSH port: `2222`
- HTTP port: `8888`
- MySQL proxy: `33306`
- PostgreSQL proxy: `5432`

### Database

By default, Warpgate uses the Postgres database that's already part of the MASH Playbook. If you prefer to use a different database engine (MySQL or SQLite), you can configure it via the `warpgate_database_engine` variable.

### Authentik Integration

If you have Authentik enabled in your MASH Playbook, Warpgate can automatically integrate with it for single sign-on (SSO). To enable this:

1. Create an Application and OIDC Provider in Authentik
2. Provide the Client ID and Secret in your configuration

```yaml
warpgate_authentik_integration_enabled: true
warpgate_sso_client_id: "your-client-id"
warpgate_sso_client_secret: "your-client-secret"
```

### Uninstall

To remove Warpgate, set the following variables and re-run the playbook:

```yaml
warpgate_enabled: false
warpgate_uninstall_remove_data: true  # Set to true to remove data
```

## License

This project is licensed under the AGPL-3.0-or-later License - see the [LICENSE](https://forgejo.littlecedar.net/littlecedar/ansible-role-mash-warpgate/src/branch/main/LICENSE) file for details.
