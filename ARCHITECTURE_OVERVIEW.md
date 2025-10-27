# Architecture Overview

## Overview

The **landscape-charm** is a Juju charm that deploys and manages Self-Hosted Landscape Server, which is Canonical's systems management tool for monitoring, managing, and updating Ubuntu infrastructure. This charm orchestrates the deployment of Landscape Server as a multi-server distributed application using Juju's operator framework.

## High-Level Architecture

This is a **machine charm** (not a Kubernetes charm) that deploys Landscape Server on Ubuntu 22.04 or 24.04 (both amd64 and arm64 architectures). The charm manages:

1. **Landscape Server services** - Multiple backend services for API, messaging, package management, etc.
2. **Service coordination** - Through relations with PostgreSQL, RabbitMQ, and HAProxy
3. **Multi-unit deployment** - Leader/follower patterns for scalability
4. **Configuration management** - Service configuration files, SSL certificates, licenses, and secrets

### Deployment Topology

```
┌──────────────┐
│   HAProxy    │  (Handles HTTP/HTTPS/gRPC traffic, SSL termination)
└──────┬───────┘
       │
┌──────┴───────────────────────────────────────────┐
│                                                   │
│  Landscape Server Units (1+)                     │
│  ┌──────────────────────────────────────────┐   │
│  │ Leader Unit                               │   │
│  │  - API Server (worker_counts processes)  │   │
│  │  - App Server (worker_counts processes)  │   │
│  │  - Message Server (worker_counts)        │   │
│  │  - Ping Server (worker_counts)           │   │
│  │  - Async Frontend                        │   │
│  │  - Job Handler                           │   │
│  │  + Package Upload (leader-only)          │   │
│  │  + Package Search (leader-only)          │   │
│  │  + Cron (leader-only)                    │   │
│  └──────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────┐   │
│  │ Follower Units                           │   │
│  │  (Same services minus leader-only ones)  │   │
│  └──────────────────────────────────────────┘   │
│                                                   │
└──────┬────────────────────┬─────────────────────┘
       │                    │
┌──────┴─────┐      ┌──────┴──────┐
│ PostgreSQL │      │ RabbitMQ    │
│ (Database) │      │ (Messaging) │
└────────────┘      └─────────────┘
```

## Directory Structure

### `/src` - Main Charm Code
**Purpose**: Contains the Python source code for the charm logic.

- **`charm.py`** (1556 lines) - Main charm class (`LandscapeServerCharm`) that handles:
  - Lifecycle events (install, config-changed, start, update-status)
  - Relations (database, AMQP, HAProxy, NRPE, COS observability)
  - Leadership changes and peer coordination
  - Actions (pause, resume, upgrade, migrate-schema, etc.)
  
- **`haproxy.py`** (492 lines) - HAProxy integration logic:
  - Service configuration builders for HTTP/HTTPS/gRPC/Ubuntu Installer Attach
  - ACL definitions for routing (API, message, ping, package-upload, etc.)
  - SSL certificate handling
  - Error file management
  
- **`settings_files.py`** (223 lines) - Configuration file management:
  - Updates `/etc/landscape/service.conf` (main Landscape config)
  - Updates `/etc/default/landscape-server` (systemd service settings)
  - License file handling
  - SSL certificate installation
  - Database configuration
  - Secret token and cookie encryption key generation
  
- **`helpers.py`** (39 lines) - Utility functions:
  - Python path cleanup for subprocess calls
  - Service configuration migration helpers
  
- **`autoregistration.py`** (64 lines) - Standalone script:
  - Toggles auto-registration of client computers
  - Imported Landscape libraries to interact with the database directly

- **`prometheus_alert_rules/`** - Prometheus alerting rules for COS integration

### `/tests` - Test Suite
**Purpose**: Unit and integration tests.

- **`unit/`** - Unit tests using ops testing harness (pytest)
  - `test_charm.py` - Main charm behavior tests
  - `test_haproxy_relation.py` - HAProxy relation tests
  - `test_settings_files.py` - Configuration file tests
  
- **`integration/`** - Integration tests
  - `test_bundle.py` - Full bundle deployment tests
  - `conftest.py` - pytest fixtures for Juju model setup

### `/lib` - Charm Libraries
**Purpose**: Third-party charm libraries (via charmcraft).

- **`charms/grafana_agent/v0/cos_agent.py`** - Canonical Observability Stack (COS) integration
- **`charms/operator_libs_linux/v0/`** - Linux system utilities (apt, passwd)
- **`charms/operator_libs_linux/v1/systemd.py`** - systemd service management

### `/bundle-examples` - Deployment Examples
**Purpose**: Example Juju bundle for deploying the full Landscape stack.

- **`bundle.yaml`** - Defines a complete deployment with PostgreSQL, RabbitMQ, HAProxy, and Landscape Server

### `/terraform` - Infrastructure as Code
**Purpose**: Terraform module for deploying this charm.

- **`main.tf`** - Juju application definition
- **`variables.tf`** - Configurable inputs
- **`outputs.tf`** - Integration endpoints
- **`versions.tf`** - Provider version constraints

### Root Configuration Files

- **`metadata.yaml`** - Charm metadata (name, relations, provides/requires interfaces)
- **`config.yaml`** - Juju configuration options (57 config values including PPA, SSL, OIDC, database settings)
- **`actions.yaml`** - Juju actions (pause, resume, upgrade, migrate-schema, etc.)
- **`charmcraft.yaml`** - Charm build configuration
- **`tox.ini`** - Test orchestration (unit, integration, lint, fmt)
- **`requirements.txt`** - Runtime Python dependencies (`ops>=1.4.0`)
- **`requirements-dev.txt`** - Development dependencies (pytest, ops testing harness, etc.)

## External Dependencies and Integrations

### Required Relations

1. **PostgreSQL** (interface: `pgsql`)
   - Purpose: Stores all Landscape data (computers, accounts, packages, alerts, etc.)
   - Required plugins: `plpython3u`, `ltree`, `intarray`, `debversion`, `pg_trgm`
   - Relation: `db` (via `db-admin` endpoint on PostgreSQL charm)

2. **RabbitMQ** (interface: `rabbitmq`)
   - Purpose: Message queue for asynchronous job processing and inter-service communication
   - Two separate vhosts:
     - `inbound-amqp` → vhost: `landscape` (client messages)
     - `outbound-amqp` → vhost: `landscape-hostagent` (hostagent messages)
   
3. **HAProxy** (interface: `http`)
   - Purpose: Load balancing, SSL termination, HTTP(S) routing
   - Relation: `website`
   - Dynamically configures backends based on ACLs:
     - `/api` → API backend (port 9080)
     - `/message-system` → Message Server (port 8090)
     - `/ping` → Ping Server (port 8070)
     - `/upload` → Package Upload (port 9100)
     - `/hash-id-databases` → Hash ID databases (port 8080)

### Optional Relations

4. **NRPE** (interface: `nrpe-external-master`)
   - Purpose: Nagios monitoring integration
   - Automatically configures service checks for all Landscape services

5. **COS Agent** (interface: `cos_agent`)
   - Purpose: Grafana Agent for Prometheus metrics collection
   - Scrapes metrics from all Landscape services (API, app-server, message-server, etc.)
   - Includes Prometheus alert rules

6. **Application Dashboard** (interface: `register-application`)
   - Purpose: Registers Landscape in a centralized dashboard

### External Services & Infrastructure

- **Landscape PPA**: APT repository for Landscape Server packages
  - Default: `ppa:landscape/self-hosted-beta`
  - Configurable via `landscape_ppa` config option
  
- **License Server**: Landscape requires a license file from Canonical
  - Downloaded from https://landscape.canonical.com
  - Configured via `license_file` option (base64-encoded or URL)

- **SMTP Relay** (optional): For email notifications
  - Configured via `smtp_relay_host`

- **OpenID/OIDC Provider** (optional): For SSO authentication
  - Supports both OpenID and OIDC (mutually exclusive)

## Runtime Environment

### Platform
- **Host OS**: Ubuntu 22.04 or 24.04 (LTS releases)
- **Architectures**: amd64, arm64
- **Deployment**: Juju-managed machines (not containers/Kubernetes)

### Landscape Server Packages Installed

1. **`landscape-server`** (main package)
2. **`landscape-client`** (for self-monitoring)
3. **`landscape-common`** (shared libraries)
4. **`landscape-hashids`** (optional, for faster package reporting)
5. **`landscape-ubuntu-installer-attach`** (optional, for Ubuntu installer integration)

### Systemd Services Managed

**Always Running (on all units):**
- `landscape-api` (worker_counts processes)
- `landscape-appserver` (worker_counts processes)
- `landscape-msgserver` (worker_counts processes)
- `landscape-pingserver` (worker_counts processes)
- `landscape-async-frontend`
- `landscape-job-handler`
- `landscape-hostagent-messenger` (if `enable_hostagent_messenger` is true)
- `landscape-hostagent-consumer` (if hostagent is enabled)

**Leader-Only Services:**
- `landscape-package-search`
- `landscape-package-upload`

**Control**: All services are managed via `/usr/bin/lsctl` wrapper script.

### Key File Locations

- **Service Config**: `/etc/landscape/service.conf` (ConfigParser format)
- **Default Settings**: `/etc/default/landscape-server` (systemd environment)
- **License File**: `/etc/landscape/license.txt`
- **SSL Certificate**: `/etc/ssl/certs/landscape_server_ca.crt`
- **Deployment Configs**: `/opt/canonical/landscape/configs/`
- **Scripts**:
  - `/usr/bin/landscape-schema` (database migrations)
  - `/opt/canonical/landscape/bootstrap-account` (admin account creation)
  - `/opt/canonical/landscape/hash-id-databases-ignore-maintenance` (package hash regeneration)
  - `/opt/canonical/landscape/update-wsl-distributions` (WSL distribution updates)
  - `/opt/canonical/landscape/migrate-service-conf` (config migration)

### Configuration State Management

The charm uses Juju's `StoredState` to track:
- Readiness of relations (`db`, `inbound-amqp`, `outbound-amqp`, `haproxy`)
- Service running state
- Pause state
- Leader IP address
- Account bootstrap status
- Secret tokens and encryption keys
- Ubuntu installer attach enabled state

### Deployment Modes

- **`standalone`** (default): Single-tenant, self-hosted deployment
- **`saas`**: Multi-tenant mode (requires additional configuration)

Configured via `deployment_mode` option.

## Scalability Patterns

### Horizontal Scaling
- Deploy multiple Landscape Server units for load distribution
- HAProxy automatically load-balances across all units
- Leader-only services run on exactly one unit
- Non-leader units proxy package-search requests to the leader

### Vertical Scaling
- `worker_counts` config option (default: 2) controls process count for:
  - API Server
  - App Server
  - Message Server
  - Ping Server

### High Availability
- Multiple units provide redundancy
- Leader election handles failover for leader-only services
- PostgreSQL can be configured with replication (external to this charm)
- RabbitMQ can be clustered (external to this charm)

## Security Considerations

- **SSL/TLS**: All traffic encrypted via HAProxy
- **Secret Management**: Secret tokens and cookie encryption keys generated securely or via Juju config
- **Database Credentials**: Provided via relation or manual config
- **License File**: Restricted permissions (0640, landscape:root)
- **Metrics Endpoints**: Blocked from external access via HAProxy ACLs
- **Proxy Support**: HTTP(S)_PROXY and NO_PROXY configuration for restricted network environments
