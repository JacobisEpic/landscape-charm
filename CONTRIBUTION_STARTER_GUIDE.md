# Contribution Starter Guide

This guide will help you set up a development environment, run tests, and understand the workflow for contributing to the Landscape Server charm.

## Prerequisites

- **Ubuntu 22.04 or 24.04** (or similar Debian-based system)
- **Python 3.10+** (Python 3.12 recommended)
- **Juju 3.x** (for integration testing)
- **Git**

## Setting Up a Development Environment

### 1. Clone the Repository

```bash
git clone https://github.com/canonical/landscape-charm
cd landscape-charm
```

### 2. Create a Python Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Install Development Dependencies

```bash
pip install -r requirements-dev.txt
```

This installs:
- **pytest** - Test framework
- **ops** - Juju operator framework
- **ops-scenario** - Testing harness for charms
- **flake8, black, isort, ruff** - Code linters and formatters
- **coverage** - Code coverage tool
- **cosl** - COS library dependencies
- **pydantic** - Data validation

### 4. Install Charmcraft (for building the charm)

```bash
sudo snap install charmcraft --classic
```

### 5. Install Juju (for integration testing)

```bash
sudo snap install juju --channel=3.6/stable
```

## Running Tests

The project uses `tox` for test orchestration, but you can also run tests directly with `pytest`.

### Unit Tests

**With tox** (recommended):
```bash
tox -e unit
```

**With pytest directly**:
```bash
pytest tests/unit
```

**Run specific test file**:
```bash
tox -e unit -- tests/unit/test_charm.py
```

**Run specific test**:
```bash
tox -e unit -- tests/unit/test_charm.py::TestCharm::test_install
```

**Test structure**:
- `tests/unit/test_charm.py` - Main charm logic tests
- `tests/unit/test_haproxy_relation.py` - HAProxy relation tests
- `tests/unit/test_settings_files.py` - Configuration file tests

**Unit test duration**: ~30-60 seconds

### Integration Tests

Integration tests deploy a full Landscape bundle with PostgreSQL, RabbitMQ, HAProxy, and Landscape Server.

**Prerequisites**:
1. A Juju controller bootstrapped
2. Access to cloud credentials (LXD, AWS, etc.)

**Run integration tests**:
```bash
tox -e integration
```

**Using existing deployment** (faster iteration):
```bash
# If you already have a Landscape deployment running
export LANDSCAPE_CHARM_USE_HOST_JUJU_MODEL=1
tox -e integration
```

**Integration test duration**: ~20-30 minutes (first run), ~5-10 minutes (with existing model)

**Test structure**:
- `tests/integration/test_bundle.py` - Full bundle deployment tests
- `tests/integration/conftest.py` - Juju model fixtures

### Code Coverage

```bash
tox -e coverage
```

This runs unit tests with coverage tracking and generates a report.

### Linting

**Check code style**:
```bash
tox -e lint
```

This runs:
- `flake8` - PEP 8 compliance
- `isort` - Import sorting
- `black` - Code formatting
- `ruff` - Fast linting

**Auto-format code**:
```bash
tox -e fmt
```

This automatically fixes:
- Import order (isort)
- Code formatting (black)
- Fixable lint issues (ruff)

## Building the Charm

### Local Build

```bash
charmcraft pack
```

This creates a `.charm` file (e.g., `landscape-server_ubuntu@22.04-amd64.charm`).

### Multi-Architecture Build

Charmcraft automatically builds for all platforms defined in `charmcraft.yaml`:
- ubuntu@22.04:amd64
- ubuntu@22.04:arm64
- ubuntu@24.04:amd64
- ubuntu@24.04:arm64

## Testing Your Changes Locally

### Quick Smoke Test (Recommended)

The fastest way to validate changes is to use the example bundle with a local charm build.

**1. Build the charm**:
```bash
make build
```

This runs `charmcraft pack` and creates a Juju model named `landscape-charm-build`.

**2. The Makefile will**:
- Build the charm
- Create a new Juju model
- Deploy the bundle from `bundle-examples/bundle.yaml`

**3. Monitor deployment**:
```bash
juju status --watch 1s
```

Wait for all units to reach "active/idle" status (~15-20 minutes).

**4. Verify Landscape is accessible**:
```bash
# Get HAProxy's IP address
juju status haproxy --format=json | jq -r '.applications.haproxy.units."haproxy/0"."public-address"'

# Open in browser (you'll get a certificate warning for self-signed cert)
https://<haproxy-ip>/
```

**5. Clean up**:
```bash
make clean
```

This destroys the test model.

### Manual Deployment Testing

If you want more control:

**1. Bootstrap a Juju controller** (if not already done):
```bash
# LXD example
juju bootstrap localhost lxd-controller

# AWS example
juju bootstrap aws aws-controller
```

**2. Create a model**:
```bash
juju add-model landscape-test
```

**3. Build the charm**:
```bash
charmcraft pack
```

**4. Deploy the bundle** (edit `bundle-examples/bundle.yaml` to point to your local charm):
```yaml
applications:
  landscape-server:
    charm: ./landscape-server_ubuntu@22.04-amd64.charm  # Use local charm
    num_units: 1
    options:
      landscape_ppa: ppa:landscape/self-hosted-beta
      min_install: True
```

**5. Deploy**:
```bash
juju deploy ./bundle-examples/bundle.yaml
```

**6. Configure license** (required):
```bash
# Option 1: From file
juju config landscape-server license_file="$(cat ~/landscape-license.txt | base64 -w0)"

# Option 2: From URL
juju config landscape-server license_file="https://your-server.com/license.txt"
```

**7. Watch status**:
```bash
watch -c juju status --color
```

**8. Check logs if issues occur**:
```bash
juju debug-log --replay --no-tail
```

### Health Check After Deployment

**1. Check all services are running** on Landscape unit:
```bash
juju ssh landscape-server/0 'sudo lsctl status'
```

Expected output: All services should show "running" (e.g., `landscape-api (1/2 workers running)`).

**2. Check relation status**:
```bash
juju status --relations
```

All relations should be established:
- landscape-server:db → postgresql:db-admin
- landscape-server:inbound-amqp → rabbitmq-server
- landscape-server:outbound-amqp → rabbitmq-server
- landscape-server → haproxy

**3. Check HAProxy routing**:
```bash
juju ssh haproxy/0 'sudo cat /var/run/haproxy.cfg | grep landscape'
```

Should show configured backends and frontends.

**4. Test web interface**:
```bash
# Get HAProxy IP
HAPROXY_IP=$(juju status haproxy --format=json | jq -r '.applications.haproxy.units."haproxy/0"."public-address"')

# Test HTTP (should redirect to HTTPS)
curl -I http://$HAPROXY_IP/

# Test HTTPS (ignore self-signed cert)
curl -k https://$HAPROXY_IP/
```

**5. Test API endpoint**:
```bash
curl -k https://$HAPROXY_IP/api/v2/
```

Should return a JSON response.

## Development Workflow

### 1. Create a Feature Branch

```bash
git checkout -b feature/my-improvement
```

### 2. Make Changes

Edit files in `src/`, update tests in `tests/`.

### 3. Run Tests Iteratively

```bash
# Quick unit test
tox -e unit

# Format code
tox -e fmt

# Lint
tox -e lint
```

### 4. Test Changes with Local Deployment

```bash
make build
# Wait for deployment
juju ssh landscape-server/0 'sudo lsctl status'
# Verify your changes work
make clean
```

### 5. Write/Update Tests

- Add unit tests for new functions in `tests/unit/`
- Update integration tests if needed in `tests/integration/`

### 6. Commit Changes

```bash
git add .
git commit -m "feat: add support for XYZ feature"
```

Follow conventional commit format:
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation changes
- `test:` - Test additions/changes
- `refactor:` - Code refactoring
- `chore:` - Build/tooling changes

### 7. Push and Create PR

```bash
git push origin feature/my-improvement
```

Then create a pull request on GitHub.

## Common Development Tasks

### Add a New Configuration Option

1. **Add to `config.yaml`**:
```yaml
options:
  my_new_option:
    type: string
    default: "default-value"
    description: |
      Description of what this does.
```

2. **Handle in charm code** (`src/charm.py`):
```python
def _on_config_changed(self, _):
    my_value = self.model.config.get("my_new_option")
    # Do something with my_value
```

3. **Write test** (`tests/unit/test_charm.py`):
```python
def test_my_new_option():
    state = State(config={"my_new_option": "test-value"})
    # ... test logic
```

### Add a New Juju Action

1. **Define in `actions.yaml`**:
```yaml
my-action:
  description: |
    Does something useful.
  params:
    my-param:
      type: string
      description: A parameter.
```

2. **Implement in charm** (`src/charm.py`):
```python
def __init__(self, *args):
    super().__init__(*args)
    self.framework.observe(self.on.my_action, self._my_action)

def _my_action(self, event: ActionEvent) -> None:
    param_value = event.params.get("my-param")
    # Do something
    event.set_results({"result": "success"})
```

3. **Test**:
```bash
juju run landscape-server/0 my-action my-param="test"
```

### Debug Integration Test Failures

1. **Keep the model alive**:
```bash
# In tests/integration/conftest.py, comment out model cleanup
```

2. **SSH into units**:
```bash
juju ssh landscape-server/0
```

3. **Check logs**:
```bash
juju debug-log --include=landscape-server/0 --replay
```

4. **Check service status**:
```bash
juju ssh landscape-server/0 'sudo lsctl status'
juju ssh landscape-server/0 'sudo journalctl -u landscape-api -n 100'
```

## Known Sharp Edges & Gotchas

### 1. Network Timeouts During Install

**Issue**: Installing dependencies can timeout in slow networks.

**Workaround**: Pre-cache packages or increase timeout values in tests.

### 2. License File Required

**Issue**: Landscape won't start without a valid license file.

**Solution**: You must obtain a license from https://landscape.canonical.com (requires Ubuntu Pro or trial).

### 3. Integration Tests are Slow

**Issue**: Full bundle deployment takes 20-30 minutes.

**Workaround**: Use `LANDSCAPE_CHARM_USE_HOST_JUJU_MODEL=1` to reuse existing model.

### 4. PostgreSQL Plugin Requirements

**Issue**: Landscape requires specific PostgreSQL plugins.

**Solution**: The bundle configures these automatically:
```yaml
postgresql:
  options:
    plugin_plpython3u_enable: true
    plugin_ltree_enable: true
    plugin_intarray_enable: true
    plugin_debversion_enable: true
    plugin_pg_trgm_enable: true
```

Don't forget these if deploying manually.

### 5. HAProxy Self-Signed Certificates

**Issue**: Browser warnings about self-signed certs.

**Solution**: For testing, ignore warnings. For production, configure proper SSL:
```bash
juju config landscape-server ssl_cert="$(cat cert.pem | base64 -w0)"
juju config landscape-server ssl_key="$(cat key.pem | base64 -w0)"
```

### 6. Secret Token Generation

**Issue**: If multiple units deploy simultaneously, they might generate different secret tokens.

**Solution**: The charm handles this via peer relation - leader generates and shares. But if you see authentication issues, check:
```bash
juju ssh landscape-server/0 'sudo grep secret-token /etc/landscape/service.conf'
juju ssh landscape-server/1 'sudo grep secret-token /etc/landscape/service.conf'
```

They should match. If not, trigger config-changed:
```bash
juju config landscape-server worker_counts=2
```

### 7. Linters Can Be Strict

**Issue**: Code must pass flake8, black, isort, and ruff.

**Solution**: Run `tox -e fmt` before committing to auto-fix most issues.

### 8. Makefile Creates Unique Model Names

**Issue**: `make build` creates a model named `<directory>-build`.

**Solution**: If you have multiple clones, rename directory or manually specify model:
```bash
juju deploy --model my-test-model ./bundle-examples/bundle.yaml
```

## Useful Commands Reference

### Juju Commands

```bash
# Show status
juju status

# Show detailed unit info
juju show-unit landscape-server/0

# SSH into unit
juju ssh landscape-server/0

# Run command on unit
juju exec --unit landscape-server/0 -- 'sudo lsctl status'

# View logs
juju debug-log --replay --include=landscape-server

# Run action
juju run landscape-server/0 pause
juju run landscape-server/0 resume

# Get config
juju config landscape-server

# Set config
juju config landscape-server worker_counts=4

# Scale units
juju add-unit landscape-server
juju remove-unit landscape-server/1
```

### Landscape Service Commands (on unit)

```bash
# Via SSH: juju ssh landscape-server/0

# Check all services
sudo lsctl status

# Restart services
sudo lsctl restart

# Stop services
sudo lsctl stop

# Start services
sudo lsctl start

# Check individual service
sudo systemctl status landscape-api

# View service logs
sudo journalctl -u landscape-api -f
```

### Testing Commands

```bash
# All unit tests
tox -e unit

# Specific test file
tox -e unit -- tests/unit/test_charm.py

# Specific test
tox -e unit -- tests/unit/test_charm.py::TestCharm::test_install -v

# Integration tests
tox -e integration

# Coverage
tox -e coverage

# Lint
tox -e lint

# Format
tox -e fmt
```

## Getting Help

- **Juju SDK Docs**: https://juju.is/docs/sdk
- **Landscape Docs**: https://ubuntu.com/landscape/docs
- **Charm Repository**: https://github.com/canonical/landscape-charm
- **Charmhub**: https://charmhub.io/landscape-server
- **Ubuntu Discourse**: https://discourse.ubuntu.com/c/landscape

## Next Steps

Once you're comfortable with the development workflow:

1. **Read the codebase docs**:
   - `ARCHITECTURE_OVERVIEW.md` - Understand the system design
   - `RUNTIME_FLOW.md` - See how things work at runtime
   - `DATA_MODEL.md` - Learn about data structures and config

2. **Pick an issue**: Check GitHub issues for "good first issue" labels

3. **Join the community**: Introduce yourself on Ubuntu Discourse

Happy hacking! 🚀
