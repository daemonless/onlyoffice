---
title: "ONLYOFFICE Document Server on FreeBSD: Collaborative Office Suite using Podman & Jails"
description: "Deploy ONLYOFFICE Document Server on FreeBSD natively using Podman. Self-hosted online office suite fully compatible with .docx, .xlsx, .pptx formats."
---

# ONLYOFFICE Document Server

[![Build Status](https://img.shields.io/github/actions/workflow/status/daemonless/onlyoffice/build.yaml?style=flat-square&label=Build&color=green)](https://github.com/daemonless/onlyoffice/actions)
[![Last Commit](https://img.shields.io/github/last-commit/daemonless/onlyoffice?style=flat-square&label=Last+Commit&color=blue)](https://github.com/daemonless/onlyoffice/commits)
[![sysvipc Required](https://img.shields.io/badge/sysvipc-required-orange?style=flat-square&logo=freebsd&logoColor=white)](https://daemonless.io/guides/ocijail-patch/)

Self-hosted online office suite for documents, spreadsheets, and presentations — fully compatible with .docx, .xlsx, and .pptx. Running natively on FreeBSD.



| | |
|---|---|
| **Port** | 8080 |
| **Registry** | `ghcr.io/daemonless/onlyoffice` |
| **Source** | [https://github.com/ONLYOFFICE/DocumentServer](https://github.com/ONLYOFFICE/DocumentServer) |
| **Website** | [https://www.onlyoffice.com/](https://www.onlyoffice.com/) |

## Version Tags

| Tag | Description | Best For |
| :--- | :--- | :--- |
| `pkg` | **FreeBSD Quarterly**. Uses stable, tested packages. | Most users. |
| `pkg-latest` | **FreeBSD Latest**. Rolling package updates. | Newest FreeBSD packages. |

## Prerequisites

Before deploying, ensure your host environment is ready. See the [Quick Start Guide](https://daemonless.io/guides/quick-start) for host setup instructions.

**Requirements:**

- FreeBSD 15.0 with Podman and ocijail
- `podman-compose`
- PostgreSQL — provided by the companion `onlyoffice-postgresql` container below

## Deploy

=== ":material-tune: Podman Compose"

    **1.** Save as `compose.yaml`:

    ```yaml { data-zip-filename="compose.yaml" }
    name: onlyoffice

    services:
      onlyoffice-documentserver:
        image: ghcr.io/daemonless/onlyoffice:pkg
        container_name: onlyoffice-documentserver
        network_mode: host
        restart: unless-stopped
        depends_on:
          - onlyoffice-postgresql
        environment:
          - DB_HOST=127.0.0.1
          - DB_PORT=5432
          - DB_NAME=onlyoffice
          - DB_USER=onlyoffice
          - DB_PWD=onlyoffice
          - HTTP_PORT=8080
          - EXAMPLE_SERVER_URL=your-hostname:8080
          #- JWT_ENABLED=true
          #- JWT_SECRET=your-secret-here
          - TZ=UTC
        volumes:
          - "/path/to/containers/onlyoffice:/config"

      onlyoffice-postgresql:
        image: ghcr.io/daemonless/postgres:latest
        container_name: onlyoffice-postgresql
        network_mode: host
        restart: unless-stopped
        annotations:
          org.freebsd.jail.allow.sysvipc: "true"
        environment:
          - POSTGRES_DB=onlyoffice
          - POSTGRES_USER=onlyoffice
          - POSTGRES_PASSWORD=onlyoffice
          - POSTGRES_HOST_AUTH_METHOD=trust
        volumes:
          - "/path/to/containers/onlyoffice-postgresql:/config"
    ```

    **2.** Deploy:

    ```bash
    mkdir -p /path/to/containers/onlyoffice /path/to/containers/onlyoffice-postgresql
    chown -R 1000:1000 /path/to/containers/onlyoffice /path/to/containers/onlyoffice-postgresql
    podman-compose up -d
    ```

    Access the editor at: **http://your-host:8080/example/**

    ??? tip "Using a named network with cni-dnsname"
        If `cni-dnsname` is installed (`pkg install cni-dnsname`), you can use a named network instead. Remove `network_mode: host`, set `DB_HOST=onlyoffice-postgresql`, remove `HTTP_PORT`, and add ports:

        ```yaml
        services:
          onlyoffice-documentserver:
            networks: [onlyoffice]
            environment:
              - DB_HOST=onlyoffice-postgresql
            ports:
              - "8080:80"
          onlyoffice-postgresql:
            networks: [onlyoffice]
        networks:
          onlyoffice:
        ```

=== ":material-console: Podman CLI"

    ```bash
    podman run -d --name onlyoffice-postgresql \
      --network host \
      --restart unless-stopped \
      --annotation 'org.freebsd.jail.allow.sysvipc=true' \
      -e POSTGRES_DB=onlyoffice \
      -e POSTGRES_USER=onlyoffice \
      -e POSTGRES_PASSWORD=onlyoffice \
      -e POSTGRES_HOST_AUTH_METHOD=trust \
      -v /path/to/containers/onlyoffice-postgresql:/config \
      ghcr.io/daemonless/postgres:latest

    podman run -d --name onlyoffice-documentserver \
      --network host \
      --restart unless-stopped \
      -e DB_HOST=127.0.0.1 \
      -e DB_PORT=5432 \
      -e DB_NAME=onlyoffice \
      -e DB_USER=onlyoffice \
      -e DB_PWD=onlyoffice \
      -e HTTP_PORT=8080 \
      -e EXAMPLE_SERVER_URL=your-hostname:8080 \
      -e TZ=UTC \
      -v /path/to/containers/onlyoffice:/config \
      ghcr.io/daemonless/onlyoffice:pkg
    ```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_HOST` | `onlyoffice-postgresql` | PostgreSQL hostname |
| `DB_PORT` | `5432` | PostgreSQL port |
| `DB_NAME` | `onlyoffice` | Database name |
| `DB_USER` | `onlyoffice` | Database user |
| `DB_PWD` | `onlyoffice` | Database password |
| `HTTP_PORT` | `80` | Port nginx listens on inside the container. Set to `8080` when using `network_mode: host` |
| `EXAMPLE_SERVER_URL` | `your-hostname:8080` | Public hostname:port the browser uses to reach the document server |
| `AMQP_URI` | `amqp://guest:guest@localhost` | RabbitMQ URI (bundled RabbitMQ used by default) |
| `JWT_ENABLED` | `true` | Enable JWT token validation |
| `JWT_SECRET` | *(auto-generated)* | JWT secret — randomly generated on first start if not set |
| `JWT_HEADER` | `Authorization` | JWT header name |
| `WOPI_ENABLED` | `false` | Enable WOPI protocol for Nextcloud/SharePoint integration |
| `TZ` | `UTC` | Timezone |

## Ports

| Port | Description |
|------|-------------|
| `8080` | Document server + example app at `/example/` |

## FreeBSD-Specific Notes

### PostgreSQL Shared Memory

PostgreSQL requires System V IPC for shared memory. Without the sysvipc annotation the container fails with `shmget: Function not implemented`. The annotation is already included in the compose file above.

### JWT Secret

If `JWT_SECRET` is not set, a random secret is generated on first start. Set it explicitly for consistent behavior across restarts.

### WOPI Integration

Set `WOPI_ENABLED=true` to enable WOPI for Nextcloud or SharePoint integration. An RSA keypair is generated automatically on first start and stored in `/config/data/wopi/`.

## Nextcloud Integration

ONLYOFFICE Document Server integrates with Nextcloud via the official ONLYOFFICE connector app, enabling in-browser editing of .docx, .xlsx, and .pptx files stored in Nextcloud.

=== ":material-tune: Podman Compose"

    Add Nextcloud to the same compose file:

    ```yaml { data-zip-filename="compose-nextcloud.yaml" }
    name: onlyoffice

    services:
      onlyoffice-documentserver:
        image: ghcr.io/daemonless/onlyoffice:pkg
        container_name: onlyoffice-documentserver
        network_mode: host
        restart: unless-stopped
        depends_on:
          - onlyoffice-postgresql
        environment:
          - DB_HOST=127.0.0.1
          - DB_PORT=5432
          - DB_NAME=onlyoffice
          - DB_USER=onlyoffice
          - DB_PWD=onlyoffice
          - HTTP_PORT=8080
          - EXAMPLE_SERVER_URL=your-hostname:8080
          - JWT_ENABLED=true
          - JWT_SECRET=your-secret-here
          - TZ=UTC
        volumes:
          - "/path/to/containers/onlyoffice:/config"

      onlyoffice-postgresql:
        image: ghcr.io/daemonless/postgres:latest
        container_name: onlyoffice-postgresql
        network_mode: host
        restart: unless-stopped
        annotations:
          org.freebsd.jail.allow.sysvipc: "true"
        environment:
          - POSTGRES_DB=onlyoffice
          - POSTGRES_USER=onlyoffice
          - POSTGRES_PASSWORD=onlyoffice
          - POSTGRES_HOST_AUTH_METHOD=trust
        volumes:
          - "/path/to/containers/onlyoffice-postgresql:/config"

      nextcloud:
        image: ghcr.io/daemonless/nextcloud:latest
        container_name: nextcloud
        network_mode: host
        restart: unless-stopped
        environment:
          - TZ=UTC
        volumes:
          - "/path/to/containers/nextcloud/config:/config"
          - "/path/to/containers/nextcloud/data:/data"
    ```

=== ":material-console: Podman CLI"

    ```bash
    # PostgreSQL
    podman run -d --name onlyoffice-postgresql \
      --network host \
      --restart unless-stopped \
      --annotation 'org.freebsd.jail.allow.sysvipc=true' \
      -e POSTGRES_DB=onlyoffice \
      -e POSTGRES_USER=onlyoffice \
      -e POSTGRES_PASSWORD=onlyoffice \
      -e POSTGRES_HOST_AUTH_METHOD=trust \
      -v /path/to/containers/onlyoffice-postgresql:/config \
      ghcr.io/daemonless/postgres:latest

    # Document Server
    podman run -d --name onlyoffice-documentserver \
      --network host \
      --restart unless-stopped \
      -e DB_HOST=127.0.0.1 \
      -e HTTP_PORT=8080 \
      -e JWT_ENABLED=true \
      -e JWT_SECRET=your-secret-here \
      -e EXAMPLE_SERVER_URL=your-hostname:8080 \
      -e TZ=UTC \
      -v /path/to/containers/onlyoffice:/config \
      ghcr.io/daemonless/onlyoffice:pkg

    # Nextcloud
    podman run -d --name nextcloud \
      --network host \
      --restart unless-stopped \
      -v /path/to/containers/nextcloud/config:/config \
      -v /path/to/containers/nextcloud/data:/data \
      ghcr.io/daemonless/nextcloud:latest
    ```

### Connector Setup

Once both containers are running:

1. Open Nextcloud at `http://your-host:8082` and complete the setup wizard
2. Go to **Apps** → search for **ONLYOFFICE** → install it
3. Go to **Settings** → **ONLYOFFICE** and set:
    - **Document Editing Service address:** `http://your-host:8080`
    - **Secret key:** `your-secret-here` (must match `JWT_SECRET`)
4. Save — Nextcloud will verify the connection

Documents in Nextcloud will now open directly in ONLYOFFICE.

!!! tip "JWT Required"
    JWT must be enabled (`JWT_ENABLED=true`) when integrating with Nextcloud. Set the same `JWT_SECRET` in both the Document Server environment and the Nextcloud ONLYOFFICE settings.

## Management

### View Logs

```bash
podman logs -f onlyoffice-documentserver
podman logs -f onlyoffice-postgresql
```

### Restart

```bash
podman-compose restart
```

### Update

```bash
podman-compose pull
podman-compose up -d
```


**Architectures:** amd64
**User:** `bsd` (UID/GID 1000:1000)
**Base:** FreeBSD 15.0


---

Need help? Join our [Discord](https://discord.gg/Kb9tkhecZT) community.