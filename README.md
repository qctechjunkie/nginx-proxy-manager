# Nginx Proxy Manager

A Docker Compose setup for [Nginx Proxy Manager](https://nginxproxymanager.com/) with MariaDB backend.

## Overview

Nginx Proxy Manager is a simple, powerful, and secure way to manage your proxy hosts with a nice web interface. This setup includes:

- Nginx Proxy Manager (latest)
- MariaDB with Aria storage engine
- Secure environment variable configuration
- Persistent data storage for configurations and SSL certificates

## Prerequisites

- Docker Engine 20.10+
- Docker Compose V2+
- External Docker network named `scoobydoo` (or modify the network name in `.env`)

## Quick Start

### 1. Create the external network (if it doesn't exist)

```bash
docker network create scoobydoo
```

### 2. Set up environment variables

Copy the example environment file and configure it:

```bash
cp .env.example .env
```

Edit `.env` and update the following variables:

```bash
# IMPORTANT: Change these default passwords!
MYSQL_ROOT_PASSWORD=your_secure_root_password_here
MYSQL_PASSWORD=your_secure_password_here
```

### 3. Start the services

```bash
docker compose up -d
```

### 4. Access the Admin UI

Navigate to `http://your-server-ip:81` (or the port specified in `ADMIN_PORT`)

**Default Admin User:**
- Email: `admin@example.com`
- Password: `changeme`

**IMPORTANT:** Change the default admin credentials immediately after first login!

## Configuration

### Environment Variables

All configuration is managed through the `.env` file:

| Variable | Description | Default |
|----------|-------------|---------|
| `MYSQL_ROOT_PASSWORD` | MariaDB root password | *Required* |
| `MYSQL_PASSWORD` | Database user password | *Required* |
| `MYSQL_DATABASE` | Database name | `npm` |
| `MYSQL_USER` | Database username | `npm` |
| `DB_MYSQL_PORT` | Database port | `3306` |
| `HTTP_PORT` | HTTP port mapping | `80` |
| `HTTPS_PORT` | HTTPS port mapping | `443` |
| `ADMIN_PORT` | Admin UI port | `81` |
| `CUSTOM_PORT_1` | Custom port 1 | `6690` |
| `CUSTOM_PORT_2` | Custom port 2 | `5000` |
| `CUSTOM_PORT_3` | Custom port 3 | `5001` |
| `NETWORK_NAME` | Docker network name | `scoobydoo` |

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
- `./mysql` - Database files

These directories are created automatically and are excluded from version control.

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

# Backup all data
tar -czf backups/npm-backup-$(date +%Y%m%d-%H%M%S).tar.gz data/ letsencrypt/ mysql/

# Start services again
docker compose up -d
```

### Restore from backup

```bash
# Stop services
docker compose down

# Remove existing data
rm -rf data/ letsencrypt/ mysql/

# Extract backup
tar -xzf backups/npm-backup-YYYYMMDD-HHMMSS.tar.gz

# Start services
docker compose up -d
```

### Rotating Database Credentials on a Live Deployment

If you need to change `MYSQL_USER`, `MYSQL_PASSWORD`, or `MYSQL_ROOT_PASSWORD` after the stack is already running, you **cannot** just update `.env` and restart. MariaDB only reads those variables on first initialization — the credentials are stored inside the database volume itself. Changing `.env` alone will cause a connection failure.

Follow these steps to rotate credentials without losing data:

**Step 1 — Update credentials inside MariaDB while it is still running**

```bash
docker compose exec db mariadb -u root -p
# Enter your current MYSQL_ROOT_PASSWORD when prompted
```

Run the following SQL (replace `newuser`, `newpassword`, and `newrootpassword` with your new values):

```sql
-- Rename the application user (skip if keeping the same username)
RENAME USER 'npm'@'%' TO 'newuser'@'%';

-- Set a new password for the application user
ALTER USER 'newuser'@'%' IDENTIFIED BY 'newpassword';

-- Rotate the root password — root can exist under multiple host entries, update both
ALTER USER 'root'@'%' IDENTIFIED BY 'newrootpassword';
ALTER USER 'root'@'localhost' IDENTIFIED BY 'newrootpassword';

-- Apply all changes
FLUSH PRIVILEGES;

-- Verify users and hosts
SELECT User, Host FROM mysql.user;

EXIT;
```

> **Note:** The database name (`MYSQL_DATABASE`, default `npm`) cannot be renamed without a full dump-and-restore. It is not security-sensitive and does not need to be changed.

**Step 2 — Update `.env`** with the new values to match what you set in SQL.

**Step 3 — Recreate the containers** to pick up the new environment variables:

```bash
docker compose up -d --force-recreate
```

**Step 4 — Verify** the app reconnected successfully:

```bash
docker compose logs app --tail 50
```

## Security Considerations

1. **Change default passwords** - Update `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD` in `.env`
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

Generate a separate password for each variable — do not reuse the same value for `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD`.

## Troubleshooting

### Services won't start

Check if the external network exists:
```bash
docker network ls | grep scoobydoo
```

If not, create it:
```bash
docker network create scoobydoo
```

### Can't access admin interface

1. Check if services are running: `docker compose ps`
2. Check logs: `docker compose logs app`
3. Verify port 81 is not in use: `netstat -tlnp | grep 81`

### Database connection errors

1. Check database logs: `docker compose logs db`
2. Verify credentials in `.env` match between services
3. Ensure `DB_MYSQL_HOST` is set to `db` in compose.yaml

## Additional Resources

- [Nginx Proxy Manager Documentation](https://nginxproxymanager.com/guide/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [MariaDB Documentation](https://mariadb.org/documentation/)

## License

This configuration is provided as-is for personal and commercial use.
