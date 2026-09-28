# Nginx Proxy Manager

A Docker Compose setup for [Nginx Proxy Manager](https://nginxproxymanager.com/) with a PostgreSQL backend.

## Overview

Nginx Proxy Manager is a simple, powerful, and secure way to manage your proxy hosts with a nice web interface. This setup includes:

- Nginx Proxy Manager (latest)
- PostgreSQL database, provided by the shared `global-postgres` service
- Secure environment variable configuration
- Persistent data storage for configurations and SSL certificates

## Prerequisites

- Docker Engine 20.10+
- Docker Compose V2+
- The shared `global-postgres` stack already running, with its Docker network (`global-postgres-network` by default) present on the host and a `npm` database/user already provisioned in it

This stack does not run its own database container. It connects to the shared `global-postgres` Postgres instance over the external `global-postgres-network` Docker network, using the `npm` database that is pre-provisioned there.

## Quick Start

### 1. Set up environment variables

Copy the example environment file and configure it:

```bash
cp .env.example .env
```

Edit `.env` and set the following to match the `npm` database credentials provisioned in `global-postgres`:

```bash
# IMPORTANT: Use the actual password provisioned for the npm database in global-postgres
DB_POSTGRES_PASSWORD=your_secure_password_here
```

### 2. Start the services

```bash
docker compose up -d
```

### 3. Access the Admin UI

Navigate to `http://your-server-ip:81` (or the port specified in `ADMIN_PORT`)

**Default Admin User (first-time setup only):**
- Email: `admin@example.com`
- Password: `changeme`

**IMPORTANT:** Change the default admin credentials immediately after first login!

## Configuration

### Environment Variables

All configuration is managed through the `.env` file:

| Variable | Description | Default |
|----------|-------------|---------|
| `DB_POSTGRES_HOST` | Postgres host/container name | `global-postgres` |
| `DB_POSTGRES_PORT` | Postgres port | `5432` |
| `DB_POSTGRES_USER` | Database username | `npm` |
| `DB_POSTGRES_PASSWORD` | Database user password | *Required* |
| `DB_POSTGRES_NAME` | Database name | `npm` |
| `HTTP_PORT` | HTTP port mapping | `80` |
| `HTTPS_PORT` | HTTPS port mapping | `443` |
| `ADMIN_PORT` | Admin UI port | `81` |
| `CUSTOM_PORT_1` | Custom port 1 | `6690` |
| `CUSTOM_PORT_2` | Custom port 2 | `5000` |
| `CUSTOM_PORT_3` | Custom port 3 | `5001` |

### Port Mappings

The default ports exposed are:
- `80` - HTTP traffic
- `443` - HTTPS traffic
- `81` - Admin Web Interface
- `6690`, `5000`, `5001` - Custom application ports

You can modify these in your `.env` file.

### Data Persistence

Data is stored in the following local directories:
- `./data` - Nginx Proxy Manager configuration
- `./letsencrypt` - SSL certificates

These directories are created automatically and are excluded from version control.

Database data lives in the shared `global-postgres` stack's own volume, not in this repo — see that stack's documentation for its backup/restore procedures.

## Management

### View logs

```bash
docker compose logs -f
```

### Stop services

```bash
docker compose down
```

### Update to latest version

```bash
docker compose pull
docker compose up -d
```

### Backup

To backup your configuration:

```bash
# Stop the services
docker compose down

# Create backup directory
mkdir -p backups

# Backup app data (SSL certs and NPM config)
tar -czf backups/npm-backup-$(date +%Y%m%d-%H%M%S).tar.gz data/ letsencrypt/

# Start services again
docker compose up -d
```

Database data is not included in this backup — it is backed up as part of the shared `global-postgres` stack (for example via `pg_dump`). See that stack's README for details.

### Restore from backup

```bash
# Stop services
docker compose down

# Remove existing data
rm -rf data/ letsencrypt/

# Extract backup
tar -xzf backups/npm-backup-YYYYMMDD-HHMMSS.tar.gz

# Start services
docker compose up -d
```

To restore the database itself, restore the `npm` database inside `global-postgres` from its own backup (e.g. `pg_restore`/`psql`), independent of this repo.

### Rotating Database Credentials on a Live Deployment

Because the database lives in the shared `global-postgres` stack, credential rotation is done there, not in this repo:

**Step 1 — Update the password inside Postgres**

```bash
docker exec -it global-postgres psql -U postgres
```

```sql
ALTER USER npm WITH PASSWORD 'newpassword';
```

**Step 2 — Update `.env`** in this repo with the new `DB_POSTGRES_PASSWORD` value.

**Step 3 — Recreate the container** to pick up the new environment variable:

```bash
docker compose up -d --force-recreate
```

**Step 4 — Verify** the app reconnected successfully:

```bash
docker compose logs app --tail 50
```

## Security Considerations

1. **Change default passwords** - Use a strong, unique `DB_POSTGRES_PASSWORD` in `.env`, matching what is provisioned in `global-postgres`
2. **Change admin credentials** - Change the default Nginx Proxy Manager admin login immediately
3. **Keep .env secure** - Never commit `.env` to version control (already in `.gitignore`)
4. **Regular updates** - Keep Docker images updated with `docker compose pull`
5. **Firewall rules** - Consider restricting access to port 81 (admin interface) to trusted IPs
6. **SSL certificates** - Use Let's Encrypt through the NPM interface for SSL certificates

### Generating Secure Passwords

Use one of these commands to generate a strong, random password before deploying:

```bash
# Base64-encoded (alphanumeric + symbols, good for most cases)
openssl rand -base64 32

# Hex-encoded (alphanumeric only, safe in all contexts)
openssl rand -hex 32

# Alphanumeric only without openssl (Linux/macOS)
< /dev/urandom tr -dc A-Za-z0-9 | head -c 32; echo
```

## Troubleshooting

### Services won't start

Check if the external network exists:
```bash
docker network ls | grep global-postgres-network
```

If not, make sure the `global-postgres` stack is deployed and running first — this stack depends on its network and database.

### Can't access admin interface

1. Check if services are running: `docker compose ps`
2. Check logs: `docker compose logs app`
3. Verify port 81 is not in use: `netstat -tlnp | grep 81`

### Database connection errors

1. Confirm `global-postgres` is healthy: `docker ps` and `docker logs global-postgres`
2. Verify `DB_POSTGRES_PASSWORD` in `.env` matches the `npm` user's password in `global-postgres`
3. Ensure `DB_POSTGRES_HOST` resolves on the `global-postgres-network` (defaults to `global-postgres`)
4. Check the app logs for `Using Postgres configuration` on startup — if you instead see `Using MySQL configuration` or `Using configuration from file`, the environment variables are not being picked up correctly

## Additional Resources

- [Nginx Proxy Manager Documentation](https://nginxproxymanager.com/guide/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

## License

This configuration is provided as-is for personal and commercial use.
