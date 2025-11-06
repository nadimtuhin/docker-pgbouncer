# PgBouncer Docker Image

[![Docker Hub](https://img.shields.io/docker/pulls/edoburu/pgbouncer.svg)](https://hub.docker.com/r/edoburu/pgbouncer)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Docker Image Size](https://img.shields.io/docker/image-size/edoburu/pgbouncer/latest)](https://hub.docker.com/r/edoburu/pgbouncer)

A lightweight, production-ready PgBouncer Docker image based on Alpine Linux. PgBouncer is a lightweight connection pooler for PostgreSQL that significantly reduces memory usage and improves application performance.

## Table of Contents

- [Features](#features)
- [Why Use PgBouncer](#why-use-pgbouncer)
- [Quick Start](#quick-start)
- [Available Tags](#available-tags)
- [Configuration](#configuration)
  - [Environment Variables](#environment-variables)
  - [Using DATABASE_URL](#using-database_url)
  - [Using Individual Variables](#using-individual-variables)
- [Advanced Usage](#advanced-usage)
  - [Custom Configuration Files](#custom-configuration-files)
  - [Multi-User Setup](#multi-user-setup)
  - [Admin Console Access](#admin-console-access)
- [Integration Examples](#integration-examples)
  - [Docker Compose](#docker-compose)
  - [Kubernetes](#kubernetes)
- [Environment Variables Reference](#environment-variables-reference)
- [PostgreSQL Configuration](#postgresql-configuration)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)
- [Credits](#credits)

## Features

- **Minimal Size**: Only ~15MB compressed image
- **Alpine Linux Based**: Secure, lightweight base image with regular security updates
- **Environment Variable Configuration**: Configure via env vars without custom config files
- **Standard PostgreSQL Port**: Uses port 5432 for transparent integration
- **PostgreSQL Client Tools**: Includes `psql`, `pg_isready`, and other client utilities
- **Multiple Authentication Methods**: Supports MD5, SCRAM-SHA-256, and plain text authentication
- **Auto-Configuration**: Automatically generates `/etc/pgbouncer/pgbouncer.ini` and `/etc/pgbouncer/userlist.txt` if they don't exist
- **Production Ready**: Battle-tested in production environments

## Why Use PgBouncer

PostgreSQL connections are resource-intensive:
- Each connection consumes approximately [10MB of memory](http://hans.io/blog/2014/02/19/postgresql_connection)
- Connection establishment has significant overhead, especially with TLS
- Web applications benefit from persistent connections

**PgBouncer solves these issues by:**
- Acting as a connection pooler between your application and PostgreSQL
- Allowing applications to maintain persistent connections to PgBouncer
- Reusing a small pool of actual PostgreSQL connections
- Dramatically reducing memory usage and improving performance

## Quick Start

### Using DATABASE_URL

```bash
docker run --rm \
    -e DATABASE_URL="postgres://user:pass@postgres-host/database" \
    -p 5432:5432 \
    edoburu/pgbouncer
```

### Using Individual Variables

```bash
docker run --rm \
    -e DB_USER=user \
    -e DB_PASSWORD=pass \
    -e DB_HOST=postgres-host \
    -e DB_NAME=database \
    -p 5432:5432 \
    edoburu/pgbouncer
```

### Connecting to PgBouncer

```bash
psql 'postgresql://user:pass@localhost/database'
```

## Available Tags

| Tag | PgBouncer Version | Status |
|-----|-------------------|--------|
| `latest` | 1.21.0 | Latest stable |
| `1.21.0` | 1.21.0 | Latest |
| `1.20.1` | 1.20.1 | Stable |
| `1.19.1` | 1.19.1 | Stable |
| `1.18.0` | 1.18.0 | Stable |
| `1.17.0` | 1.17.0 | Stable |
| `1.15.0` | 1.15.0 | Stable |
| `1.14.0` | 1.14.0 | Stable |
| `1.12.0` | 1.12.0 | Stable |

**Note:** Images are automatically rebuilt when Alpine Linux security updates are released.

View all tags on [Docker Hub](https://hub.docker.com/r/edoburu/pgbouncer/tags).

## Configuration

### Using DATABASE_URL

The `DATABASE_URL` environment variable accepts standard PostgreSQL connection strings:

```bash
DATABASE_URL="postgres://username:password@hostname:port/database"
```

This format is compatible with Django's `dj-database-url` and other frameworks.

### Using Individual Variables

You can also configure using individual environment variables:

```bash
docker run --rm \
    -e DB_USER=myuser \
    -e DB_PASSWORD=mypassword \
    -e DB_HOST=postgres.example.com \
    -e DB_PORT=5432 \
    -e DB_NAME=mydb \
    -e POOL_MODE=transaction \
    -e MAX_CLIENT_CONN=100 \
    -e DEFAULT_POOL_SIZE=20 \
    -p 5432:5432 \
    edoburu/pgbouncer
```

## Advanced Usage

### Custom Configuration Files

For advanced scenarios requiring custom configuration, mount your own `pgbouncer.ini` file:

```bash
docker run --rm \
    -v $(pwd)/pgbouncer.ini:/etc/pgbouncer/pgbouncer.ini:ro \
    -v $(pwd)/userlist.txt:/etc/pgbouncer/userlist.txt:ro \
    -p 5432:5432 \
    edoburu/pgbouncer
```

Or extend the Dockerfile:

```dockerfile
FROM edoburu/pgbouncer:1.21.0
COPY pgbouncer.ini userlist.txt /etc/pgbouncer/
```

**Note:** When `pgbouncer.ini` exists, the entrypoint script will not override it. However, credentials from `DATABASE_URL`, `DB_USER`, and `DB_PASSWORD` will still be added to `userlist.txt`.

### Multi-User Setup

The `userlist.txt` file format supports multiple users:

```txt
"username1" "plaintext-password"
"username2" "md5<md5 hash of password+username>"
```

#### Generating MD5 Passwords

Use the included script to generate properly formatted entries:

```bash
./examples/generate-userlist >> userlist.txt
```

#### Using auth_user for Database Password Retrieval

You can configure a single authentication user and retrieve database passwords dynamically by setting `AUTH_USER`. See [CYBERTEC's guide](https://www.cybertec-postgresql.com/en/pgbouncer-authentication-made-easy/) for details.

### Admin Console Access

When an admin user is configured with a password in `userlist.txt`, you can access the PgBouncer admin console:

```bash
# From outside the container
psql postgres://postgres:password@hostname/pgbouncer

# From inside the container
psql postgres://postgres:password@127.0.0.1/pgbouncer
```

#### Available Admin Commands

```sql
SHOW STATS;           -- Show connection statistics
SHOW SERVERS;         -- Show server connections
SHOW CLIENTS;         -- Show client connections
SHOW POOLS;           -- Show pool information
SHOW DATABASES;       -- Show configured databases
PAUSE;                -- Pause all connections
RESUME;               -- Resume connections
RELOAD;               -- Reload configuration
```

See the [PgBouncer admin console documentation](https://pgbouncer.github.io/usage.html#admin-console) for all commands.

## Integration Examples

### Docker Compose

See the [examples/docker-compose](examples/docker-compose) folder for a complete example:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb

  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DATABASE_URL: "postgres://myuser:mypassword@postgres/mydb"
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 100
    ports:
      - "5432:5432"
    depends_on:
      - postgres

  app:
    image: myapp:latest
    environment:
      DATABASE_URL: "postgres://myuser:mypassword@pgbouncer/mydb"
    depends_on:
      - pgbouncer
```

### Kubernetes

See the [examples/kubernetes](examples/kubernetes) folder for complete examples:

- **Single User Setup**: [examples/kubernetes/singleuser](examples/kubernetes/singleuser)
- **Multi-User Setup**: [examples/kubernetes/multiuser](examples/kubernetes/multiuser)

## Environment Variables Reference

Almost all [pgbouncer.ini settings](https://pgbouncer.github.io/config.html) can be configured via environment variables. Here are the most commonly used:

### Database Connection

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | - | Full PostgreSQL connection string |
| `DB_HOST` | **required** | PostgreSQL server hostname |
| `DB_PORT` | `5432` | PostgreSQL server port |
| `DB_NAME` | `*` | Database name (use `*` for all databases) |
| `DB_USER` | `postgres` | Database username |
| `DB_PASSWORD` | - | Database password |

### Pool Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `POOL_MODE` | - | Pooling mode: `session`, `transaction`, or `statement` |
| `MAX_CLIENT_CONN` | `100` | Maximum number of client connections |
| `DEFAULT_POOL_SIZE` | `20` | Default pool size per user/database pair |
| `MIN_POOL_SIZE` | `0` | Minimum pool size |
| `RESERVE_POOL_SIZE` | `0` | Reserve pool size |
| `MAX_DB_CONNECTIONS` | unlimited | Maximum connections per database |
| `MAX_USER_CONNECTIONS` | unlimited | Maximum connections per user |

### Authentication

| Variable | Default | Description |
|----------|---------|-------------|
| `AUTH_TYPE` | `md5` | Authentication type: `md5`, `scram-sha-256`, `plain`, `trust`, `any`, `hba` |
| `AUTH_FILE` | `/etc/pgbouncer/userlist.txt` | Path to user authentication file |
| `AUTH_HBA_FILE` | - | Path to HBA-style authentication file |
| `AUTH_USER` | - | User to use for querying authentication |
| `AUTH_QUERY` | - | Query to load user's password |

### Timeouts

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_IDLE_TIMEOUT` | `600` | Server idle timeout in seconds |
| `SERVER_LIFETIME` | `3600` | Maximum server connection lifetime |
| `SERVER_CONNECT_TIMEOUT` | `15` | Timeout for connecting to server |
| `QUERY_TIMEOUT` | `0` | Query timeout (0 = disabled) |
| `QUERY_WAIT_TIMEOUT` | `120` | Maximum wait time for a query |
| `CLIENT_IDLE_TIMEOUT` | `0` | Client idle timeout |
| `IDLE_TRANSACTION_TIMEOUT` | `0` | Timeout for idle transactions |

### Logging

| Variable | Default | Description |
|----------|---------|-------------|
| `LOG_CONNECTIONS` | `1` | Log connections |
| `LOG_DISCONNECTIONS` | `1` | Log disconnections |
| `LOG_POOLER_ERRORS` | `1` | Log pooler errors |
| `VERBOSE` | `0` | Increase verbosity (0-2) |
| `LOG_STATS` | `1` | Log statistics |
| `STATS_PERIOD` | `60` | Statistics logging period |

### Connection Sanity

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_RESET_QUERY` | `DISCARD ALL` | Query to run on release |
| `SERVER_RESET_QUERY_ALWAYS` | `0` | Run reset query always |
| `SERVER_CHECK_QUERY` | `SELECT 1` | Health check query |
| `SERVER_CHECK_DELAY` | `30` | Health check delay in seconds |

### TLS/SSL Configuration

| Variable | Description |
|----------|-------------|
| `CLIENT_TLS_SSLMODE` | Client TLS mode: `disable`, `allow`, `prefer`, `require`, `verify-ca`, `verify-full` |
| `CLIENT_TLS_KEY_FILE` | Path to client TLS key file |
| `CLIENT_TLS_CERT_FILE` | Path to client TLS certificate file |
| `CLIENT_TLS_CA_FILE` | Path to client TLS CA file |
| `SERVER_TLS_SSLMODE` | Server TLS mode |
| `SERVER_TLS_KEY_FILE` | Path to server TLS key file |
| `SERVER_TLS_CERT_FILE` | Path to server TLS certificate file |
| `SERVER_TLS_CA_FILE` | Path to server TLS CA file |

### Advanced Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `MAX_PREPARED_STATEMENTS` | `0` | Maximum prepared statements (0 = disabled) |
| `IGNORE_STARTUP_PARAMETERS` | `extra_float_digits` | Startup parameters to ignore |
| `DISABLE_PQEXEC` | `0` | Disable simple query protocol |
| `APPLICATION_NAME_ADD_HOST` | `0` | Add hostname to application_name |
| `SERVER_ROUND_ROBIN` | `0` | Use round-robin server selection |

See the [entrypoint.sh](entrypoint.sh) script for the complete list of supported variables.

## PostgreSQL Configuration

### Allowing Remote Connections

Ensure PostgreSQL accepts connections from the PgBouncer container:

1. **Update `postgresql.conf`**:
   ```conf
   listen_addresses = '*'  # or specific IP range
   ```

2. **Update `pg_hba.conf`**:
   ```conf
   # TYPE  DATABASE        USER            ADDRESS                 METHOD
   host    all             all             10.0.0.0/8              md5
   host    all             all             172.16.0.0/12           md5
   ```

3. **Restart PostgreSQL** after making changes.

### Security Best Practices

- Use strong passwords
- Limit `pg_hba.conf` to specific IP ranges
- Consider using SCRAM-SHA-256 authentication (PostgreSQL 10+)
- Use TLS/SSL for connections in production
- Regularly update both PostgreSQL and PgBouncer images

## Troubleshooting

### Connection Refused

**Problem**: Cannot connect to PgBouncer

**Solutions**:
- Verify the container is running: `docker ps`
- Check logs: `docker logs <container-name>`
- Ensure port 5432 is mapped: `docker run -p 5432:5432 ...`
- Verify firewall rules allow connections

### Authentication Failed

**Problem**: Password authentication failed

**Solutions**:
- Verify credentials in `DATABASE_URL` or `DB_USER`/`DB_PASSWORD`
- Check `userlist.txt` has the correct user entry
- Ensure `AUTH_TYPE` matches PostgreSQL's authentication method
- For SCRAM-SHA-256, set `AUTH_TYPE=scram-sha-256`

### PostgreSQL Connection Refused

**Problem**: PgBouncer cannot connect to PostgreSQL

**Solutions**:
- Verify `DB_HOST` is correct and reachable from container
- Check PostgreSQL is running and accepting connections
- Verify `pg_hba.conf` allows connections from PgBouncer's IP
- Test connection: `docker exec -it <pgbouncer-container> psql -h $DB_HOST -U $DB_USER`

### Pool Size Issues

**Problem**: "no more connections allowed" or "database connection limit reached"

**Solutions**:
- Increase `MAX_CLIENT_CONN` for more client connections
- Increase `DEFAULT_POOL_SIZE` for more server connections
- Adjust `POOL_MODE` (transaction mode is most efficient)
- Check PostgreSQL's `max_connections` setting

### Prepared Statement Errors

**Problem**: "prepared statement X already exists"

**Solutions**:
- Set `MAX_PREPARED_STATEMENTS=100` (or higher)
- Use transaction pooling mode: `POOL_MODE=transaction`
- Set `SERVER_RESET_QUERY=DISCARD ALL` to clean prepared statements

### Viewing Configuration

To see the active configuration:

```bash
docker exec -it <container-name> cat /etc/pgbouncer/pgbouncer.ini
```

To check if custom config is being used:

```bash
docker exec -it <container-name> ls -la /etc/pgbouncer/
```

### Debugging with Admin Console

Connect to the admin console to check status:

```bash
psql postgres://postgres@localhost/pgbouncer -c "SHOW POOLS;"
psql postgres://postgres@localhost/pgbouncer -c "SHOW CLIENTS;"
psql postgres://postgres@localhost/pgbouncer -c "SHOW SERVERS;"
```

## Contributing

Contributions are welcome! Here's how you can help:

### Reporting Issues

- Use the [GitHub issue tracker](https://github.com/edoburu/docker-pgbouncer/issues)
- Check if the issue already exists before creating a new one
- Include version information (PgBouncer version, Docker version, OS)
- Provide reproduction steps and relevant logs

### Pull Requests

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Make your changes and test thoroughly
4. Commit with clear messages: `git commit -m "Add feature X"`
5. Push to your fork: `git push origin feature/my-feature`
6. Open a Pull Request with a clear description

### Testing Changes

Build and test locally:

```bash
# Build the image
docker build -t pgbouncer-test .

# Run tests
docker run --rm -e DB_HOST=localhost pgbouncer-test pgbouncer --version
```

### Code Style

- Follow existing shell script conventions
- Keep the image minimal and secure
- Document all environment variables
- Update README.md for user-facing changes

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Copyright (c) 2019 Diederik van der Boor

## Credits

- **Maintainer**: [Diederik van der Boor](https://github.com/edoburu)
- **PgBouncer**: [pgbouncer.org](https://pgbouncer.org/)
- **Inspiration**: [gmr/alpine-pgbouncer](https://github.com/gmr/alpine-pgbouncer)

### Contributors

Thank you to all contributors who have helped improve this project!

See the [GitHub contributors page](https://github.com/edoburu/docker-pgbouncer/graphs/contributors) for a full list.

---

**Star this repository** if you find it useful! ⭐

**Found a bug?** [Open an issue](https://github.com/edoburu/docker-pgbouncer/issues)

**Want to discuss?** Start a [discussion](https://github.com/edoburu/docker-pgbouncer/discussions)
