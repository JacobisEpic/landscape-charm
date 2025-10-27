# Runtime Flow Documentation

This document traces the critical execution paths in the Landscape Server charm, showing step-by-step what happens during key operations.

## Flow 1: Initial Installation and Deployment

This flow describes what happens when you deploy the charm for the first time.

### Step-by-Step Trace

#### 1. Install Event (`_on_install`)
**File**: `src/charm.py:537-635`

1. **Set status**: "Installing apt packages"

2. **Import PPA GPG key** (if provided):
   - Config: `landscape_ppa_key`
   - Function: `charms.operator_libs_linux.v0.apt.import_key()`
   - Stores key for signature verification

3. **Remove needrestart package**:
   - Command: `apt.remove_package(["needrestart"])`
   - Reason: Prevents hanging installs

4. **Set up proxy environment** for `add-apt-repository`:
   - Checks `http_proxy`, `https_proxy`, `no_proxy` from config
   - Falls back to `JUJU_CHARM_HTTP_PROXY`, etc. from environment
   - File: `src/charm.py:560-586`

5. **Add Landscape PPA**:
   - Command: `add-apt-repository -y <landscape_ppa>`
   - Default: `ppa:landscape/self-hosted-beta`

6. **Install packages**:
   - If `min_install=True`:
     - `apt install landscape-server --no-install-recommends -y`
   - Otherwise:
     - `apt.add_package(["landscape-server", "landscape-hashids"])`
     - `apt-mark hold landscape-hashids`
   - Always: `apt-mark hold landscape-server`

7. **Install SSL certificate** (if provided):
   - Config: `ssl_cert` (base64-encoded)
   - Function: `settings_files.write_ssl_cert()`
   - File: `/etc/ssl/certs/landscape_server_ca.crt`

8. **Install license file** (if provided):
   - Config: `license_file` (base64 or URL)
   - Function: `settings_files.write_license_file()`
   - File: `/etc/landscape/license.txt`
   - Permissions: `0640`, owned by `landscape:root`

9. **Mark deployment source**:
   - Function: `settings_files.prepend_default_settings({"DEPLOYED_FROM": "charm"})`
   - File: `/etc/default/landscape-server`

10. **Update status**: "Unit is ready"

11. **Check readiness**:
    - Function: `_update_ready_status()`
    - Waits for relations: db, inbound-amqp, outbound-amqp, haproxy

**Result**: Packages installed, basic config in place, waiting for relations.

---

#### 2. Database Relation Joined (`_db_relation_changed`)
**File**: `src/charm.py:702-777`

1. **Receive relation data**:
   - `master` (connection string)
   - `port` (database port)
   - `user` (admin user)
   - `password` (admin password)
   - `allowed-units` (space-separated list)

2. **Validate this unit is allowed**:
   - Check if `self.unit.name` in `allowed-units`
   - Return early if not allowed yet

3. **Parse master connection string**:
   - Format: `host=<ip> dbname=<db> port=<port>`
   - Extract host, port, user, password

4. **Override with manual config** (if provided):
   - `db_host`, `db_port`, `db_schema_user`, `db_landscape_password`, `db_schema_password`

5. **Update database config**:
   - Function: `settings_files.update_db_conf()`
   - Updates `/etc/landscape/service.conf`:
     - `[stores]` section: `host`, `password`
     - `[schema]` section: `store_user`, `store_password`

6. **Run schema bootstrap**:
   - Function: `_migrate_schema_bootstrap()`
   - Command: `/usr/bin/landscape-schema --bootstrap [proxy-settings]`
   - Creates databases, tables, landscape user
   - Sets up proxy settings in the database

7. **Bootstrap admin account** (if configured):
   - Function: `_bootstrap_account()`
   - Config needed: `admin_email`, `admin_name`, `admin_password`, `root_url`
   - Script: `/opt/canonical/landscape/bootstrap-account`
   - Creates initial administrator account

8. **Set autoregistration** (if configured):
   - Function: `_set_autoregistration()`
   - Config: `autoregistration` (boolean)
   - Script: `src/autoregistration.py`
   - Allows computers to self-register

9. **Update WSL distributions**:
   - Function: `_update_wsl_distributions()`
   - Script: `/opt/canonical/landscape/update-wsl-distributions`
   - Pre-populates Windows Subsystem for Linux distribution data

10. **Mark database ready**:
    - `self._stored.ready["db"] = True`

11. **Trigger service restart**:
    - Function: `_update_ready_status(restart_services=True)`

**Result**: Database configured, schemas migrated, admin account created.

---

#### 3. RabbitMQ Relations Joined (`_amqp_relation_joined`, `_amqp_relation_changed`)
**File**: `src/charm.py:841-888`

Two separate relations: `inbound-amqp` and `outbound-amqp`

**Join Phase** (`_amqp_relation_joined`):
1. **Send requirements** to RabbitMQ:
   - `username`: "landscape"
   - `vhost`:
     - `inbound-amqp` → "landscape"
     - `outbound-amqp` → "landscape-hostagent"

**Changed Phase** (`_amqp_relation_changed`):
1. **Wait for password**:
   - RabbitMQ sends `password` and `hostname` back

2. **Wait for both relations**:
   - Need both `inbound-amqp` and `outbound-amqp` ready

3. **Update service config**:
   - Function: `settings_files.update_service_conf()`
   - Updates `/etc/landscape/service.conf`:
     ```ini
     [broker]
     host = <hostname>
     password = <password>
     ```

4. **Mark both AMQP relations ready**:
   - `self._stored.ready["inbound-amqp"] = True`
   - `self._stored.ready["outbound-amqp"] = True`

**Result**: Message queue configured for asynchronous jobs.

---

#### 4. HAProxy Relation Joined (`_website_relation_joined`, `_update_haproxy_connection`)
**File**: `src/charm.py:890-991`

1. **Validate SSL configuration**:
   - Function: `_get_ssl_cert()`
   - If `ssl_cert != "DEFAULT"` and `ssl_key == ""`: ERROR
   - If both provided: combine cert + key, base64-encode

2. **Validate redirect_https config**:
   - Function: `_get_redirect_https()`
   - Valid values: "default", "all", "none"

3. **Get error files**:
   - Function: `haproxy.get_haproxy_error_files()`
   - Location: `/opt/canonical/landscape/canonical/landscape/offline/`
   - Files: 403, 500, 502, 503, 504 HTML pages
   - Base64-encode each file

4. **Create HTTP service** configuration:
   - Function: `haproxy.create_http_service()`
   - Service: `landscape-http` on port 80
   - ACLs for routing:
     - `/ping` → ping backend
     - `/repository` → repository backend  
     - `/message-system` → message backend
     - `/attachment` → message backend
     - `/api` → API backend
     - `/hash-id-databases` → hashids backend
     - `/upload` → package-upload backend
   - HTTPS redirect (unless `/ping` or `/repository`)
   - Metrics endpoints blocked

5. **Create HTTPS service** configuration:
   - Function: `haproxy.create_https_service()`
   - Service: `landscape-https` on port 443
   - Same ACLs as HTTP
   - SSL certificate attached

6. **Create gRPC service** (if `enable_hostagent_messenger`):
   - Function: `haproxy.create_grpc_service()`
   - Service: `landscape-grpc` on port 6554
   - HTTP/2 protocol

7. **Create Ubuntu Installer Attach service** (if enabled):
   - Function: `haproxy.create_ubuntu_installer_attach_service()`
   - Service: `landscape-ubuntu-installer-attach` on port 50051
   - HTTP/2 protocol

8. **Send services to HAProxy**:
   - Relation data: `services` (YAML-encoded list)
   - HAProxy reads this and configures backends

9. **Set default root_url** (if not manually configured):
   - Uses HAProxy's public-address
   - Format: `https://<haproxy-ip>/`
   - Updates service.conf

10. **Mark HAProxy ready**:
    - `self._stored.ready["haproxy"] = True`

**Result**: HAProxy configured with all routing rules and backends.

---

#### 5. All Relations Ready → Start Services (`_start_services`)
**File**: `src/charm.py:663-700`

Triggered when all four relations are ready: db, inbound-amqp, outbound-amqp, haproxy.

1. **Update service environment**:
   - Function: `settings_files.update_default_settings()`
   - File: `/etc/default/landscape-server`
   - Settings:
     ```bash
     RUN_ALL=no
     RUN_APISERVER=2          # (or worker_counts value)
     RUN_APPSERVER=2
     RUN_MSGSERVER=2
     RUN_PINGSERVER=2
     RUN_ASYNC_FRONTEND=yes
     RUN_JOBHANDLER=yes
     RUN_CRON=yes             # if leader
     RUN_PACKAGESEARCH=yes    # if leader
     RUN_PACKAGEUPLOADSERVER=yes  # if leader and standalone mode
     RUN_PPPA_PROXY=no
     ```

2. **Restart all services**:
   - Command: `/usr/bin/lsctl restart`
   - Environment: Modified PYTHONPATH (removes Juju paths)
   - Starts/restarts all enabled services

3. **Update status**: "Unit is ready"

4. **Mark running**:
   - `self._stored.running = True`

**Result**: Landscape Server fully operational!

---

## Flow 2: Configuration Changes

This flow describes what happens when you run `juju config landscape-server <option>=<value>`.

### Step-by-Step Trace

#### Config Changed Event (`_on_config_changed`)
**File**: `src/charm.py:347-499`

1. **Validate `redirect_https` config**:
   - Function: `_get_redirect_https()`
   - If invalid: Set status to "MaintenanceStatus" with error message
   - Return early

2. **Handle Ubuntu Installer Attach toggle**:
   - Function: `_configure_ubuntu_installer_attach()`
   - If enabling: `apt.add_package("landscape-ubuntu-installer-attach")`
   - If disabling: `apt.remove_package("landscape-ubuntu-installer-attach")`
   - Updates `self._stored.enable_ubuntu_installer_attach`

3. **Update deployment mode**:
   - Config: `deployment_mode` ("standalone" or "saas")
   - Function: `settings_files.configure_for_deployment_mode()`
   - Creates symlink: `/opt/canonical/landscape/configs/<mode>` → `standalone`

4. **Update service.conf**:
   - Section: `[global]`
   - Key: `deployment-mode`

5. **Merge additional config**:
   - Config: `additional_service_config`
   - Function: `settings_files.merge_service_conf()`
   - Allows custom service.conf sections

6. **Install SSL certificate** (if changed):
   - Config: `ssl_cert`
   - If not "DEFAULT": Function: `settings_files.write_ssl_cert()`

7. **Install license file** (if changed):
   - Config: `license_file`
   - Function: `settings_files.write_license_file()`

8. **Configure SMTP relay** (if changed):
   - Config: `smtp_relay_host`
   - Function: `_configure_smtp()`
   - Updates `/etc/postfix/main.cf`
   - Reloads postfix service

9. **Update HAProxy relations** (if any exist):
   - Function: `_update_haproxy_connection()`
   - Regenerates service configurations
   - Sends updated YAML to HAProxy

10. **Validate OpenID vs OIDC**:
    - Both cannot be configured simultaneously
    - If both: Set status to "BlockedStatus"

11. **Configure OpenID** (if changed):
    - Function: `_configure_openid()`
    - Configs: `openid_provider_url`, `openid_logout_url`
    - Both required
    - Updates service.conf

12. **Configure OIDC** (if changed):
    - Function: `_configure_oidc()`
    - Configs: `oidc_issuer`, `oidc_client_id`, `oidc_client_secret`, `oidc_logout_url`
    - First three required, logout_url optional
    - Updates service.conf

13. **Update worker counts and root_url**:
    - Config: `worker_counts`, `root_url`
    - Updates service.conf sections: `[landscape]`, `[api]`, `[message-server]`, `[pingserver]`, `[package-upload]`

14. **Override database config** (if changed):
    - Configs: `db_host`, `db_port`, `db_schema_user`, `db_landscape_password`, `db_schema_password`
    - Function: `settings_files.update_db_conf()`
    - Re-runs schema migration if needed

15. **Bootstrap account** (if not done yet):
    - Function: `_bootstrap_account()`

16. **Set autoregistration** (if changed):
    - Function: `_set_autoregistration()`

17. **Generate/update secret tokens**:
    - Config: `secret_token` (or auto-generated by leader)
    - Config: `cookie_encryption_key` (or auto-generated by leader)
    - Leader stores in peer relation data
    - All units read from peer relation
    - Functions: `_get_secret_token()`, `_write_secret_token()`

18. **Update status and restart services**:
    - Function: `_update_ready_status(restart_services=True)`
    - Restarts services if config changed

**Result**: Configuration changes applied, services restarted.

---

## Flow 3: Leader Election and Failover

This flow describes what happens when the leader changes (e.g., due to unit removal or failure).

### Step-by-Step Trace

#### Leader Elected Event (`_leader_elected`)
**File**: `src/charm.py:1121-1140`

**On the newly elected leader:**

1. **Double-check leadership**:
   - `if self.unit.is_leader():`
   - Juju events can be delayed

2. **Update peer relation**:
   - Relation: `replicas`
   - Data: `leader-ip` = <this unit's bind address>
   - All units will receive this change

3. **Configure package-search to localhost**:
   - Updates service.conf:
     ```ini
     [package-search]
     host = localhost
     ```
   - Leader runs package-search service locally

4. **Call generic leader change handler**:
   - Function: `_leader_changed()`

---

#### Leader Settings Changed Event (`_leader_settings_changed`)
**File**: `src/charm.py:1142-1163`

**On non-leader units:**

1. **Read leader IP from peer relation**:
   - Relation: `replicas`
   - Data: `leader-ip`

2. **Configure package-search to leader IP**:
   - Updates service.conf:
     ```ini
     [package-search]
     host = <leader-ip>
     ```
   - Non-leaders proxy package-search requests to leader

3. **Call generic leader change handler**:
   - Function: `_leader_changed()`

---

#### Generic Leader Change Handler (`_leader_changed`)
**File**: `src/charm.py:1165-1197`

**On all units:**

1. **Update NRPE checks**:
   - Function: `_update_nrpe_checks()`
   - Leader gets checks for all services
   - Non-leaders get checks only for non-leader services
   - Creates/removes files in `/etc/nagios/nrpe.d/`

2. **If this is the leader:**
   - **Update HAProxy connections**:
     - Function: `_update_haproxy_connection()`
     - Regenerates service configurations
   
   - **Resume leader-only services**:
     - Services: `landscape-package-search`, `landscape-package-upload`
     - Command: `systemctl start <service>` (via `charms.operator_libs_linux.v1.systemd.service_resume()`)

3. **If this is a non-leader:**
   - **Pause leader-only services**:
     - Services: `landscape-package-search`, `landscape-package-upload`
     - Command: `systemctl stop <service>` (via `charms.operator_libs_linux.v1.systemd.service_pause()`)

4. **Restart all services**:
   - Function: `_update_ready_status(restart_services=True)`

**Result**: Leader-only services running on exactly one unit, non-leaders properly configured to proxy requests.

---

## Key Observations

### State Transitions

The charm tracks readiness using `StoredState`:
```python
self._stored.ready = {
    "db": False,
    "inbound-amqp": False,
    "outbound-amqp": False,
    "haproxy": False,
}
```

Each relation must be ready before services start. The charm uses `_update_ready_status()` to check this state after each relation change.

### Service Control Pattern

All Landscape services are controlled via `/usr/bin/lsctl`:
- `lsctl restart` - Restart all enabled services
- `lsctl start` - Start all enabled services
- `lsctl stop` - Stop all enabled services
- `lsctl status` - Check status

Which services run is controlled by `/etc/default/landscape-server` environment variables (`RUN_APISERVER=2`, etc.).

### Error Handling

The charm uses Juju status messages to communicate state:
- `WaitingStatus` - Waiting for relations or configuration
- `MaintenanceStatus` - Performing maintenance operations
- `BlockedStatus` - User intervention required (invalid config, failed operations)
- `ActiveStatus` - Everything is working

Critical failures (package installation, schema migration) will set `BlockedStatus` and log detailed errors.

### Configuration Persistence

The charm migrates service.conf after every update:
- Function: `helpers.migrate_service_conf()`
- Script: `/opt/canonical/landscape/migrate-service-conf`
- Ensures compatibility with newer Landscape versions

### Proxy Handling

Proxy settings are complex:
1. Can be set via charm config (`http_proxy`, `https_proxy`, `no_proxy`)
2. Can come from Juju model config (`JUJU_CHARM_HTTP_PROXY`, etc.)
3. Charm config overrides Juju model config
4. Used for:
   - `add-apt-repository` (during install)
   - `landscape-schema --bootstrap` (database setup)
   - Stored in Landscape database for client use
