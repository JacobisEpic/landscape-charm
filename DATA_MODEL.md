# Data Model Documentation

This document describes the main data structures, configuration objects, and settings that the Landscape Server charm uses.

## Configuration Options (Juju Config)

The charm exposes 57 configuration options via `config.yaml`. These control every aspect of Landscape deployment.

### Core Installation Settings

#### `landscape_ppa` (string, default: "ppa:landscape/self-hosted-beta")
- **Purpose**: APT repository for Landscape Server packages
- **Consumed by**: `_on_install()` during package installation
- **Example**: `ppa:landscape/self-hosted-24.04`

#### `landscape_ppa_key` (string, default: "")
- **Purpose**: ASCII-armored GPG public key for the PPA
- **Consumed by**: `_on_install()` for signature verification
- **Format**: Full PGP public key block

#### `license_file` (string, required)
- **Purpose**: Landscape license from Canonical
- **Consumed by**: `write_license_file()` in settings_files.py
- **Format**: base64-encoded license data OR URL (file://, http://, https://)
- **Location**: `/etc/landscape/license.txt`
- **Permissions**: 0640, landscape:root

#### `min_install` (boolean, default: False)
- **Purpose**: Skip recommended packages (like hashids) for faster installation
- **Consumed by**: `_on_install()`

### SSL/TLS Settings

#### `ssl_cert` (string, default: "DEFAULT")
- **Purpose**: SSL certificate for HTTPS
- **Consumed by**: `_get_ssl_cert()`, `write_ssl_cert()`
- **Format**: Base64-encoded certificate (combined cert + key for HAProxy)
- **Special value**: "DEFAULT" means use HAProxy-generated cert
- **Location**: `/etc/ssl/certs/landscape_server_ca.crt`

#### `ssl_key` (string, default: "")
- **Purpose**: Private key for SSL certificate
- **Consumed by**: `_get_ssl_cert()` (combined with ssl_cert)
- **Format**: Base64-encoded private key
- **Validation**: Required if ssl_cert is set (not DEFAULT)

### Service Configuration

#### `worker_counts` (int, default: 2)
- **Purpose**: Number of worker processes for each service
- **Consumed by**: `_start_services()`, `_on_config_changed()`
- **Applies to**:
  - landscape-api (2 processes)
  - landscape-appserver (2 processes)
  - landscape-msgserver (2 processes)
  - landscape-pingserver (2 processes)
- **Updates**: `/etc/default/landscape-server` (RUN_APISERVER=2, etc.)

#### `deployment_mode` (string, default: "standalone")
- **Purpose**: Tenancy mode for Landscape
- **Consumed by**: `configure_for_deployment_mode()`
- **Values**:
  - `standalone` - Single-tenant (normal use case)
  - `saas` - Multi-tenant (requires additional config)
- **Effect**: Creates symlink in `/opt/canonical/landscape/configs/`

#### `root_url` (string, optional)
- **Purpose**: Base URL for Landscape deployment
- **Consumed by**: Many parts of the charm, service.conf
- **Format**: `https://landscape.example.com/`
- **Default behavior**: If not set, uses HAProxy's public-address
- **Updates**: service.conf sections: [global], [api], [package-upload]

#### `site_name` (string, default: "juju")
- **Purpose**: Unique identifier for this Landscape deployment
- **Consumed by**: NRPE monitoring, application dashboard
- **Example**: "production-eu-west"

### Database Settings

All database settings can override values from the PostgreSQL relation.

#### `db_host` (string, optional)
- **Purpose**: PostgreSQL host (overrides relation-provided value)
- **Consumed by**: `update_db_conf()`
- **Format**: IP address or hostname

#### `db_port` (string, optional)
- **Purpose**: PostgreSQL port (overrides relation-provided value)
- **Default**: "5432" (standard PostgreSQL port)
- **Consumed by**: `update_db_conf()`

#### `db_schema_user` (string, optional)
- **Purpose**: Admin user for schema migrations
- **Consumed by**: `update_db_conf()`, landscape-schema script
- **Default**: Uses value from PostgreSQL relation

#### `db_schema_password` (string, optional)
- **Purpose**: Password for schema admin user
- **Consumed by**: `update_db_conf()`
- **Default**: Falls back to db_landscape_password, then relation value

#### `db_landscape_password` (string, optional)
- **Purpose**: Password for normal landscape database user
- **Consumed by**: `update_db_conf()`
- **Default**: Uses value from PostgreSQL relation

### Authentication Settings (Mutually Exclusive)

#### OpenID (Legacy)

**`openid_provider_url`** (string, optional)
- **Purpose**: OpenID provider endpoint
- **Consumed by**: `_configure_openid()`
- **Requires**: openid_logout_url also set

**`openid_logout_url`** (string, optional)
- **Purpose**: OpenID logout endpoint
- **Consumed by**: `_configure_openid()`
- **Requires**: openid_provider_url also set

#### OIDC (Modern)

**`oidc_issuer`** (string, optional)
- **Purpose**: OIDC provider issuer URL
- **Consumed by**: `_configure_oidc()`
- **Example**: `https://accounts.google.com`

**`oidc_client_id`** (string, optional)
- **Purpose**: OIDC client identifier
- **Consumed by**: `_configure_oidc()`

**`oidc_client_secret`** (string, optional)
- **Purpose**: OIDC client secret
- **Consumed by**: `_configure_oidc()`

**`oidc_logout_url`** (string, optional)
- **Purpose**: OIDC logout endpoint
- **Consumed by**: `_configure_oidc()`

### Account Bootstrap Settings (One-time use)

These settings can only be used once during initial deployment.

#### `admin_email` (string, optional)
- **Purpose**: Email address for initial admin account
- **Consumed by**: `_bootstrap_account()`
- **Requires**: admin_name, admin_password, root_url
- **Script**: `/opt/canonical/landscape/bootstrap-account`

#### `admin_name` (string, optional)
- **Purpose**: Full name of initial admin
- **Consumed by**: `_bootstrap_account()`

#### `admin_password` (string, optional)
- **Purpose**: Password for initial admin
- **Consumed by**: `_bootstrap_account()`
- **Security**: Redacted in logs

#### `registration_key` (string, optional)
- **Purpose**: Initial account registration key for client enrollment
- **Consumed by**: `_bootstrap_account()`
- **Security**: Redacted in logs

#### `system_email` (string, optional)
- **Purpose**: "From" address for Landscape emails
- **Consumed by**: `_bootstrap_account()`

#### `autoregistration` (boolean, default: false)
- **Purpose**: Allow clients to register without admin approval
- **Consumed by**: `_set_autoregistration()`, `autoregistration.py`
- **When**: Only after account is bootstrapped

### Network Settings

#### `http_proxy` (string, optional)
- **Purpose**: HTTP proxy for outbound connections
- **Consumed by**: `_proxy_settings`, `_on_install()`
- **Fallback**: JUJU_CHARM_HTTP_PROXY environment variable

#### `https_proxy` (string, optional)
- **Purpose**: HTTPS proxy for outbound connections
- **Consumed by**: `_proxy_settings`, `_on_install()`

#### `no_proxy` (string, optional)
- **Purpose**: Comma-separated list of hosts that bypass proxy
- **Consumed by**: `_proxy_settings`, `_on_install()`

#### `smtp_relay_host` (string, optional)
- **Purpose**: SMTP server for outgoing email
- **Consumed by**: `_configure_smtp()`
- **Updates**: `/etc/postfix/main.cf`

### HAProxy Routing Settings

#### `redirect_https` (string, default: "default")
- **Purpose**: Controls HTTP to HTTPS redirection
- **Consumed by**: `_update_haproxy_connection()`, `create_http_service()`
- **Values**:
  - `default` - Redirect all except /ping and /repository
  - `all` - Redirect everything
  - `none` - No redirection
- **Validation**: Invalid values cause MaintenanceStatus error

### Optional Features

#### `enable_hostagent_messenger` (boolean, default: false)
- **Purpose**: Enable hostagent services for advanced features
- **Consumed by**: `_start_services()`, `_update_haproxy_connection()`
- **Effect**: 
  - Runs landscape-hostagent-messenger service
  - Creates gRPC frontend on HAProxy (port 6554)

#### `enable_ubuntu_installer_attach` (boolean, default: false)
- **Purpose**: Enable Ubuntu installer integration
- **Consumed by**: `_configure_ubuntu_installer_attach()`
- **Effect**:
  - Installs landscape-ubuntu-installer-attach package
  - Creates gRPC frontend on HAProxy (port 50051)

### Security Settings

#### `secret_token` (string, optional)
- **Purpose**: Secret for signing Landscape sessions/tokens
- **Consumed by**: `_get_secret_token()`, `_write_secret_token()`
- **Generation**: Auto-generated by leader if not provided (172 random alphanumeric chars)
- **Storage**: Peer relation data (shared across all units)
- **Updates**: service.conf [landscape] section

#### `cookie_encryption_key` (string, optional)
- **Purpose**: Fernet-style key for API cookie encryption
- **Consumed by**: `_get_cookie_encryption_key()`, `_write_cookie_encryption_key()`
- **Format**: 32 URL-safe base64-encoded bytes
- **Generation**: Auto-generated by leader if not provided
- **Storage**: Peer relation data
- **Updates**: service.conf [api] section

### Monitoring Settings

#### `nagios_context` (string, default: "juju")
- **Purpose**: Prefix for NRPE check names in Nagios
- **Consumed by**: `_update_nrpe_checks()`
- **Example**: "juju-landscape-server-0"

#### `nagios_servicegroups` (string, optional)
- **Purpose**: Comma-separated list of Nagios servicegroups
- **Consumed by**: `_update_nrpe_checks()`
- **Default**: Uses nagios_context if not set

#### `prometheus_scrape_interval` (string, default: "1m")
- **Purpose**: How often Prometheus scrapes metrics
- **Consumed by**: `_generate_scrape_configs()` for COS integration
- **Format**: Prometheus duration (e.g., "30s", "5m")

### Advanced Settings

#### `additional_service_config` (string, optional)
- **Purpose**: Custom service.conf settings to merge
- **Consumed by**: `merge_service_conf()`
- **Format**: ConfigParser INI format
- **Use case**: Override/add settings not exposed as charm config

---

## Relation Data Structures

### Database Relation (`pgsql` interface)

**Relation name**: `db`  
**Required from PostgreSQL**:

```python
{
    "master": "host=10.0.0.10 dbname=postgres port=5432",
    "allowed-units": "landscape-server/0 landscape-server/1",
    "port": "5432",
    "user": "landscape_admin",
    "password": "random-password-123",
}
```

**Consumed by**: `_db_relation_changed()`

---

### RabbitMQ Relations (`rabbitmq` interface)

**Relation names**: `inbound-amqp`, `outbound-amqp`

**Sent to RabbitMQ**:
```python
{
    "username": "landscape",
    "vhost": "landscape",  # or "landscape-hostagent" for outbound
}
```

**Received from RabbitMQ**:
```python
{
    "hostname": "10.0.0.20",
    "password": "random-password-456",
}
```

**Consumed by**: `_amqp_relation_joined()`, `_amqp_relation_changed()`

---

### HAProxy Relation (`http` interface)

**Relation name**: `website`

**Sent to HAProxy**:
```python
{
    "services": """
    - service_name: landscape-http
      service_host: 0.0.0.0
      service_port: 80
      service_options:
        - mode http
        - timeout client 300000
        - ...
      servers:
        - [landscape-server-0, 10.0.0.30, 8080, 'check inter 5000 rise 2 fall 5']
        - ...
    - service_name: landscape-https
      ...
    """
}
```

**Received from HAProxy**:
```python
{
    "public-address": "203.0.113.10",
    "ssl_cert": "base64-encoded-cert",  # if DEFAULT cert
}
```

**Consumed by**: `_update_haproxy_connection()`, `_website_relation_changed()`

---

### Peer Relation (`landscape-replica` interface)

**Relation name**: `replicas`

**Application-level data** (set by leader):
```python
{
    "leader-ip": "10.0.0.30",
    "secret-token": "random-172-char-string",
    "cookie-encryption-key": "base64-encoded-32-bytes",
}
```

**Unit-level data**:
```python
{
    "unit-data": "landscape-server/0",
}
```

**Consumed by**: `_on_replicas_relation_changed()`, `_leader_elected()`

---

## File Structures

### `/etc/landscape/service.conf` (ConfigParser INI format)

Main configuration file for Landscape Server.

**Example structure**:
```ini
[global]
deployment-mode = standalone
root-url = https://landscape.example.com/

[landscape]
secret-token = AbCdEf123456...
oidc-issuer = https://accounts.google.com
oidc-client-id = my-client-id
oidc-client-secret = my-secret

[api]
workers = 2
root-url = https://landscape.example.com/
cookie-encryption-key = Zm9vYmFy...

[message-server]
workers = 2

[pingserver]
workers = 2

[broker]
host = 10.0.0.20
password = rabbitmq-password

[stores]
host = 10.0.0.10:5432
password = landscape-db-password

[schema]
store_user = landscape_admin
store_password = schema-password

[package-search]
host = 10.0.0.30  # or localhost if leader

[package-upload]
root-url = https://landscape.example.com/
```

**Managed by**: `settings_files.update_service_conf()`, `merge_service_conf()`

---

### `/etc/default/landscape-server` (Shell environment)

Controls which systemd services should run.

**Example**:
```bash
DEPLOYED_FROM="charm"
RUN_ALL=no
RUN_APISERVER=2
RUN_APPSERVER=2
RUN_MSGSERVER=2
RUN_PINGSERVER=2
RUN_ASYNC_FRONTEND=yes
RUN_JOBHANDLER=yes
RUN_CRON=yes
RUN_PACKAGESEARCH=yes
RUN_PACKAGEUPLOADSERVER=yes
RUN_PPPA_PROXY=no
```

**Managed by**: `settings_files.update_default_settings()`, `prepend_default_settings()`

---

## Charm State (StoredState)

The charm uses ops.framework.StoredState to persist data across hook invocations.

```python
self._stored.ready = {
    "db": False,
    "inbound-amqp": False,
    "outbound-amqp": False,
    "haproxy": False,
}
```
**Purpose**: Track which relations are ready to start services.

```python
self._stored.leader_ip = ""
```
**Purpose**: Cache leader's IP address for package-search routing.

```python
self._stored.running = False
```
**Purpose**: Whether services are currently running.

```python
self._stored.paused = False
```
**Purpose**: Whether services have been manually paused (via action).

```python
self._stored.default_root_url = ""
```
**Purpose**: Auto-detected root URL from HAProxy (if not manually configured).

```python
self._stored.account_bootstrapped = False
```
**Purpose**: Whether initial admin account has been created (one-time operation).

```python
self._stored.secret_token = None
```
**Purpose**: Cached secret token (for comparison to detect changes).

```python
self._stored.cookie_encryption_key = None
```
**Purpose**: Cached cookie encryption key.

```python
self._stored.enable_ubuntu_installer_attach = False
```
**Purpose**: Whether Ubuntu installer attach is enabled.

---

## HAProxy Service Definitions (Data Classes)

### `Service` dataclass
**File**: `src/haproxy.py:66-76`

```python
@dataclass(frozen=True)
class Service:
    service_name: str        # e.g., "landscape-http"
    service_host: str        # e.g., "0.0.0.0"
    service_port: int        # e.g., 80
    server_options: list[str]  # e.g., ["check", "inter 5000"]
    service_options: list[str] # e.g., ["mode http", "balance leastconn"]
```

**Pre-defined constants**:
- `HTTP_SERVICE` (port 80)
- `HTTPS_SERVICE` (port 443)
- `GRPC_SERVICE` (port 6554)
- `UBUNTU_INSTALLER_ATTACH_SERVICE` (port 50051)

### `HAProxyErrorFile` dataclass
**File**: `src/haproxy.py:210-219`

```python
@dataclass
class HAProxyErrorFile:
    http_status: int       # e.g., 503
    content: bytes         # base64-encoded HTML
```

Used for custom error pages (403, 500, 502, 503, 504).

---

## Constants and Enums

### ACL Enum (HAProxy Access Control Lists)
**File**: `src/haproxy.py:12-27`

```python
class ACL(str, Enum):
    API = "api"
    ATTACHMENT = "attachment"
    HASHIDS = "hashids"
    MESSAGE = "message"
    PACKAGE_UPLOAD = "package-upload"
    PING = "ping"
    REPOSITORY = "repository"
```

Used to route URLs to backends:
- `/api` → API backend
- `/message-system` → MESSAGE backend
- `/attachment` → MESSAGE backend (same as message)
- `/ping` → PING backend
- `/hash-id-databases` → HASHIDS backend
- `/upload` → PACKAGE_UPLOAD backend

### RedirectHTTPS Enum
**File**: `src/haproxy.py:55-63`

```python
class RedirectHTTPS(str, Enum):
    ALL = "all"
    NONE = "none"
    DEFAULT = "default"
```

### Service Ports Mapping
**File**: `src/haproxy.py:190-198`

```python
PORTS = {
    "appserver": 8080,
    "pingserver": 8070,
    "message-server": 8090,
    "api": 9080,
    "package-upload": 9100,
    "hostagent-messenger": 50052,
    "ubuntu-installer-attach": 53354,
}
```

### Default Services Lists
**File**: `src/charm.py:107-120`

```python
DEFAULT_SERVICES = (
    "landscape-api",
    "landscape-appserver",
    "landscape-async-frontend",
    "landscape-job-handler",
    "landscape-msgserver",
    "landscape-pingserver",
    "landscape-hostagent-messenger",
    "landscape-hostagent-consumer",
)

LEADER_SERVICES = (
    "landscape-package-search",
    "landscape-package-upload",
)
```

### Metric-Instrumented Services
**File**: `src/charm.py:139-146`

```python
METRIC_INSTRUMENTED_SERVICE_PORTS = [
    ("appserver", 8080),
    ("pingserver", 8070),
    ("message-server", 8090),
    ("api", 9080),
    ("package-upload", 9100),
    ("package-search", 9099),
]
```

These services expose `/metrics` endpoints for Prometheus scraping.

---

## Environment Variables

### Juju-Provided (from model or charm config)

- `JUJU_CHARM_HTTP_PROXY` - HTTP proxy from Juju model
- `JUJU_CHARM_HTTPS_PROXY` - HTTPS proxy from Juju model
- `JUJU_CHARM_NO_PROXY` - No-proxy list from Juju model

**Consumed by**: `_proxy_settings`, `_on_install()`

### Charm-Modified (for subprocess calls)

The charm modifies `PYTHONPATH` when calling Landscape scripts:
- Removes Juju paths
- Adds `/opt/canonical/landscape`

**Function**: `helpers.get_modified_env_vars()`  
**Used by**: All subprocess calls to Landscape scripts

---

## Summary

The charm's data model is primarily configuration-driven:
1. **57 Juju config options** control all aspects of deployment
2. **4 relation interfaces** provide external service integration
3. **2 primary config files** (`service.conf`, `default/landscape-server`)
4. **StoredState** tracks operational state between hooks
5. **Peer relation** shares secrets and leader information
6. **HAProxy services** configured dynamically via YAML

The charm acts as a translation layer: it takes high-level Juju config/relations and translates them into low-level Landscape configuration files and systemd service states.
