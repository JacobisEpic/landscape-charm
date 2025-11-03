# Copilot Instructions for Landscape Server Charm

## Project Overview
This is a Juju charm for deploying Canonical's Landscape Server, a systems management tool for monitoring and managing Ubuntu infrastructure. The charm is written in Python using the Operator Framework.

## Project Structure

### Key Directories
- `src/` - Main charm implementation
- `tests/unit/` - Unit tests using pytest and ops.testing
- `tests/integration/` - Integration tests for full deployment scenarios
- `lib/` - External charm libraries and dependencies
- `bundle-examples/` - Example Juju bundle configurations

### Core Files
- `src/charm.py` - Main charm class (`LandscapeServerCharm`)
- `src/config.py` - Configuration validation using Pydantic
- `src/database.py` - Database relation handling
- `src/haproxy.py` - HAProxy reverse proxy integration
- `src/settings_files.py` - Landscape configuration file management
- `metadata.yaml` - Charm metadata and relation definitions
- `config.yaml` - Charm configuration options
- `charmcraft.yaml` - Charm build configuration

## Development Environment

### Building and Testing
- Use `tox` for all testing: `tox -e unit`, `tox -e integration`, `tox -e lint`
- Specific test patterns: `tox -e unit -- tests/unit/test_charm.py::TestClass::test_method`
- Build locally: `ccc pack` (charmcraftcache)
- Deploy: `make deploy` or `juju deploy ./bundle-examples/postgres16.bundle.yaml`

### Dependencies
- Python 3.8+ with ops framework
- Juju 3.x for deployment
- PostgreSQL for database backend
- HAProxy for reverse proxy

## Code Conventions

### Python Style
- Follow PEP 8
- Use type hints for function signatures
- Comprehensive docstrings for public methods
- Import order: standard library, third-party, local imports

### Testing Patterns
- Use `ops.testing.Context` for unit tests
- Mock external dependencies (subprocess, file operations)
- Test both success and failure scenarios
- Parameterized tests for multiple input scenarios

### Charm Patterns
- Event handlers prefixed with `_on_` or `_` for internal methods
- Use `StoredState` for persistent charm data
- Relations use modern `data_interfaces` library where possible
- Configuration validation using Pydantic models

## Key Components

### Main Charm Class
- `LandscapeServerCharm` inherits from `ops.charm.CharmBase`
- Handles lifecycle events: install, config-changed, start, update-status
- Manages relations: database, website (HAProxy), AMQP
- Supports actions: pause, resume, upgrade, migrate-schema

### Configuration Management
- `LandscapeCharmConfiguration` Pydantic model in `config.py`
- Validates configuration on charm events
- Supports SSL certificates, database settings, deployment modes

### Service Management
- Multiple Landscape services: api, appserver, msgserver, etc.
- Leader/follower pattern for some services
- Service configuration via `/etc/landscape/service.conf`

## Relations

### Database
- Modern: `database` relation using `data_platform_libs`
- Legacy: `db` relation (deprecated but supported)
- Requires PostgreSQL with SUPERUSER role

### Website
- HAProxy reverse proxy integration
- SSL certificate management
- Multiple service backends (HTTP, HTTPS, gRPC)

### AMQP
- Inbound/outbound message queuing
- RabbitMQ integration

## Common Patterns

### Error Handling
- Use charm status to communicate state: `ActiveStatus`, `BlockedStatus`, `WaitingStatus`
- Log errors before setting blocked status
- Graceful degradation when possible

### File Operations
- Use absolute paths consistently
- Set proper file permissions and ownership
- Backup configuration files before modification

### Secret Management
- Use `get_args_with_secrets_removed()` for logging command arguments
- Base64 encoding for SSL certificates
- Juju secrets for sensitive data

## Testing Guidelines

### Unit Tests
- Test each method in isolation
- Mock external dependencies (subprocess, file I/O, network)
- Use `Context` for charm lifecycle testing
- Verify both success and error paths

### Integration Tests
- Test full charm deployment scenarios
- Verify relation data exchange
- Test configuration changes
- Use real Juju models when possible

## Debugging Tips

### Common Issues
- License file configuration required for deployment
- Database relation must be established before services start
- SSL certificate format must be base64 encoded
- Worker counts affect memory usage

### Useful Commands
- `juju debug-log --include=landscape-server` - View charm logs
- `juju show-unit landscape-server/0` - Check unit status and relations
- `juju config landscape-server` - View current configuration
- `juju ssh landscape-server/0` - SSH to unit for debugging

## Architecture Notes

### Deployment Modes
- `standalone` - Single unit deployment
- `dense` - Multiple services on fewer units
- `scalable` - Distributed across multiple units

### Service Topology
- Frontend: HAProxy (SSL termination, load balancing)
- Application: Landscape server components
- Database: PostgreSQL (separate charm)
- Optional: RabbitMQ for messaging

### Scaling Considerations
- Database is typically the bottleneck
- Multiple landscape-server units for high availability
- HAProxy distributes load across application servers

## Security Considerations
- SSL/TLS required for production
- Database connections use encrypted channels
- Secret data handled via Juju secrets
- File permissions properly managed for service user

This file helps GitHub Copilot understand the project structure, conventions, and common patterns when providing assistance with code generation, debugging, and refactoring.