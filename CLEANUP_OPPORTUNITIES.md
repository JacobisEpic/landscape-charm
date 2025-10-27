# Cleanup Opportunities

This document identifies areas of technical debt, potential refactoring opportunities, and code quality improvements in the Landscape Server charm. These are observations only—**no changes should be made without proper design review and testing**.

## High Priority: Architectural Concerns

### 1. Error Status Handling Anti-Pattern

**Files**: `src/charm.py:358`, `src/charm.py:368`

**Issue**: When configuration errors occur (invalid `redirect_https` or package installation failures), the charm sets `MaintenanceStatus` instead of `BlockedStatus`. This is done intentionally to avoid "permanent blocking," but it creates a misleading user experience.

**Current Code**:
```python
try:
    _get_redirect_https(raw_redirect_https)
except InvalidRedirectHTTPS as e:
    # TODO Should be "blocked" eventually, but this causes the charm to be
    # stuck in the "blocked" state permanently.
    self.unit.status = MaintenanceStatus(str(e))
    return
```

**Why Risky**:
- Users see "maintenance" status when there's actually a configuration error
- The charm appears to be working when it's not
- Violates Juju status semantics (BlockedStatus = needs human intervention)

**Proposed Refactor**:
1. Use `BlockedStatus` for configuration errors
2. Implement config validation in `validate()` method (ops framework pattern)
3. Clear blocked status when config is corrected
4. Add tests for status transitions

**Effort**: Medium (requires understanding ops validation patterns)

---

### 2. Leader Settings Changed Event (Deprecated)

**File**: `src/charm.py:1142-1163`

**Issue**: The charm still uses the deprecated `leader-settings-changed` event from Juju 2.x. The code even has a comment acknowledging this.

**Current Code**:
```python
def _leader_settings_changed(self, event: LeaderSettingsChangedEvent) -> None:
    """
    Applies changes on non-leader units after a new leader is elected
    Deprecated call from Juju 3.x
    It is better to handler non-leader specific configuration by using
    the peer relation replicas_relation_changed contents
    """
```

**Why Risky**:
- `leader-settings-changed` is deprecated and may be removed in future Juju versions
- Duplicate logic exists in `_on_replicas_relation_changed`
- Potential race conditions between the two event handlers

**Proposed Refactor**:
1. Remove `leader-settings-changed` event handler
2. Consolidate all logic into `_on_replicas_relation_changed`
3. Ensure leader-elected triggers peer relation data update
4. Test thoroughly with multi-unit deployments

**Effort**: Medium (requires integration testing with leader failover scenarios)

---

### 3. Hardcoded Service Configuration

**File**: `src/charm.py:139-155`

**Issue**: Service ports and metrics configurations are hardcoded in Python. The TODO comment indicates this should be configurable.

**Current Code**:
```python
METRIC_INSTRUMENTED_SERVICE_PORTS = [
    ("appserver", 8080),
    ("pingserver", 8070),
    ("message-server", 8090),
    ("api", 9080),
    ("package-upload", 9100),
    ("package-search", 9099),
]
"""
Currently this var is only used for metrics configuration, so it only includes the
applicable services.

TODO all service configuration should be configurable through Juju and passed to the
Landscape server configuration file.
"""
```

**Why Risky**:
- Cannot change ports without modifying charm code
- HAProxy configuration tightly coupled to these constants
- Difficult to support custom Landscape configurations

**Proposed Refactor**:
1. Add Juju config options for service ports (with current values as defaults)
2. Pass port configuration to HAProxy dynamically
3. Update service.conf generation to use configured ports
4. Validate port ranges and conflicts

**Effort**: High (impacts multiple systems: charm, HAProxy, service.conf)

---

## Medium Priority: Code Quality Issues

### 4. Large Monolithic Charm Class

**File**: `src/charm.py` (1556 lines, 49 methods)

**Issue**: The `LandscapeServerCharm` class handles too many responsibilities:
- All lifecycle events
- All relation handlers
- All action handlers
- Service management
- Configuration validation
- Status management

**Why Confusing**:
- Difficult to test individual components
- Hard to understand the flow for specific features
- Violates Single Responsibility Principle

**Proposed Refactor**:
1. Extract relation handlers into separate classes:
   - `DatabaseRelationHandler`
   - `AMQPRelationHandler`
   - `HAProxyRelationHandler`
   - `NRPERelationHandler`
2. Extract action handlers into `LandscapeActions` class
3. Extract service management into `LandscapeServiceManager` class
4. Use composition pattern in main charm class

**Example**:
```python
class LandscapeServerCharm(CharmBase):
    def __init__(self, *args):
        super().__init__(*args)
        self.database = DatabaseRelationHandler(self)
        self.amqp = AMQPRelationHandler(self)
        self.haproxy = HAProxyRelationHandler(self)
        self.services = LandscapeServiceManager(self)
```

**Effort**: High (major refactor, requires extensive testing)

---

### 5. Duplicated SSL Certificate Handling

**Files**: `src/charm.py:178-196`, `src/charm.py:912-920`, `src/settings_files.py:195-201`

**Issue**: SSL certificate decoding and validation logic appears in multiple places.

**Why Confusing**:
- `_get_ssl_cert()` in charm.py does base64 decode + validation
- `write_ssl_cert()` in settings_files.py also does base64 decode
- `_website_relation_changed()` has special handling for HAProxy cert format

**Proposed Refactor**:
1. Create `SSLCertificateManager` class in settings_files.py
2. Centralize all cert validation, decoding, and writing
3. Handle different cert sources (config, HAProxy, combined cert+key) in one place

**Effort**: Medium

---

### 6. Environment Variable Manipulation Pattern

**File**: `src/helpers.py:11-20`

**Issue**: The charm modifies `PYTHONPATH` for subprocess calls, but this pattern is scattered throughout the codebase.

**Current Code**:
```python
def get_modified_env_vars():
    """
    Because the python path gets munged by the juju env, this grabs the current
    env vars and returns a copy with the juju env removed from the python paths
    """
    env_vars = os.environ.copy()
    logger.info("Fixing python paths")
    new_paths = [path for path in sys.path if "juju" not in path]
    env_vars["PYTHONPATH"] = ":".join(new_paths) + ":/opt/canonical/landscape"
    return env_vars
```

**Why Confusing**:
- Every subprocess call to Landscape scripts needs to remember to use this
- Easy to forget and cause subtle bugs
- String matching "juju" is fragile (what about a path like `/home/juju-user/`?)

**Proposed Refactor**:
1. Create wrapper function: `run_landscape_script(script_path, args, **kwargs)`
2. Automatically apply environment modifications
3. Add logging and error handling
4. Replace all `subprocess.run()` calls with wrapper

**Effort**: Low-Medium

---

### 7. Inconsistent Configuration File Updates

**Files**: `src/settings_files.py` (multiple functions)

**Issue**: Different patterns for updating config files:
- `update_service_conf()` - Uses ConfigParser
- `update_default_settings()` - Manual line-by-line replacement
- `prepend_default_settings()` - Prepends to file
- `merge_service_conf()` - Merges ConfigParser data

**Why Confusing**:
- Hard to predict how a config change will be applied
- Potential for race conditions (file read/write not atomic)
- No rollback mechanism if update fails mid-way

**Proposed Refactor**:
1. Use atomic file writes (write to temp, then rename)
2. Standardize on ConfigParser for all INI files
3. Add validation before writing
4. Add backup mechanism

**Effort**: Medium

---

## Low Priority: Minor Improvements

### 8. Magic Strings and Constants

**File**: `src/charm.py` (throughout)

**Issue**: Many hardcoded paths and values scattered in code:
```python
LSCTL = "/usr/bin/lsctl"
NRPE_D_DIR = "/etc/nagios/nrpe.d"
POSTFIX_CF = "/etc/postfix/main.cf"
SCHEMA_SCRIPT = "/usr/bin/landscape-schema"
# ... 10+ more
```

**Why Risky**:
- If paths change in Landscape package, must update multiple places
- Hard to override for testing

**Proposed Refactor**:
1. Group all paths into `LandscapePaths` dataclass or config object
2. Allow environment variable overrides for testing
3. Single source of truth

**Effort**: Low

---

### 9. Inconsistent Return Types

**File**: `src/charm.py` (various methods)

**Issue**: Some methods return `bool` for success/failure, others return `None`, others raise exceptions.

**Examples**:
- `_start_services()` returns `bool`
- `_migrate_schema_bootstrap()` returns `bool | None`
- `_update_wsl_distributions()` returns `bool | None`
- `_bootstrap_account()` returns `None` (logs errors)

**Why Confusing**:
- Inconsistent error handling patterns
- Callers must check different ways

**Proposed Refactor**:
1. Standardize on one pattern: raise exceptions for errors
2. Or: Always return result object with status + message
3. Document pattern in CONTRIBUTING.md

**Effort**: Medium (affects many callsites)

---

### 10. Test Skips Without Clear Reason

**File**: `tests/unit/test_charm.py:669`, `tests/unit/test_charm.py:1909`

**Issue**:
```python
@unittest.skipIf(IS_CI, "Fails in CI for unknown reason. TODO FIXME.")
```

```python
# TODO fix from broken commit.
```

**Why Risky**:
- Tests that are skipped don't catch regressions
- "Unknown reason" suggests the test itself might be flaky or incorrect
- Broken commits should be fixed immediately

**Proposed Refactor**:
1. Investigate why test fails in CI
2. Fix the test or the code
3. If test is fundamentally flaky, remove it or redesign
4. Never skip tests "for unknown reason"

**Effort**: Medium (requires debugging)

---

### 11. Long Parameter Lists

**File**: `src/haproxy.py` (various functions)

**Issue**: Functions like `create_http_service()` take 9+ parameters.

**Current Signature**:
```python
def create_http_service(
    http_service: dict,
    server_ip: str,
    unit_name: str,
    worker_counts: int,
    is_leader: bool,
    error_files: list[HAProxyErrorFile],
    service_ports: HAProxyServicePorts,
    server_options: HAProxyServerOptions,
    redirect_https: RedirectHTTPS,
) -> dict:
```

**Why Confusing**:
- Hard to remember parameter order
- Easy to pass wrong value to wrong parameter
- Function signature changes break many callsites

**Proposed Refactor**:
1. Create `HAProxyServiceConfig` dataclass
2. Pass single config object instead of many parameters
3. Use keyword-only arguments

**Effort**: Low-Medium

---

### 12. Proxy Configuration Complexity

**Files**: `src/charm.py:560-586`, `src/charm.py:779-796`

**Issue**: Proxy configuration logic is duplicated and complex:
- Charm config overrides Juju model config
- Different proxy env vars for different tools
- Special handling for `add-apt-repository`

**Why Confusing**:
- Multiple sources of truth for proxy settings
- Easy to miss a place that needs proxy config

**Proposed Refactor**:
1. Create `ProxyConfiguration` class
2. Centralize logic for resolving proxy settings
3. Provide method: `get_proxy_env()` that returns complete environment

**Effort**: Medium

---

### 13. Service Port Constants Duplicated

**Files**: `src/charm.py:139-146`, `src/haproxy.py:190-198`

**Issue**: Service ports defined in two places (though values are the same).

**Why Risky**:
- Could diverge if one is updated
- DRY principle violation

**Proposed Refactor**:
1. Define in one place (probably haproxy.py since it's data, not logic)
2. Import where needed

**Effort**: Low

---

## Testing Gaps

### 14. Integration Test Coverage

**File**: `tests/integration/test_bundle.py`

**Issue**: Only one integration test file, and it primarily tests deployment success. Missing:
- Leader failover scenarios
- Scale-up/scale-down testing
- Upgrade testing
- Relation lifecycle (remove/re-add relations)
- Config change propagation
- Action testing

**Proposed Additions**:
1. `test_leader_failover.py` - Test leader re-election
2. `test_scaling.py` - Test adding/removing units
3. `test_upgrade.py` - Test charm/Landscape package upgrades
4. `test_actions.py` - Test all Juju actions

**Effort**: High (integration tests are slow)

---

### 15. Unit Test for Relation Data Validation

**Issue**: Unit tests don't thoroughly validate malformed relation data handling.

**Scenarios to Test**:
- PostgreSQL sends incomplete data
- RabbitMQ sends malformed password
- HAProxy sends invalid SSL cert
- Peer relation data is corrupted

**Proposed Additions**:
Add test cases for each relation's error handling.

**Effort**: Medium

---

## Documentation Gaps

### 16. Inline Comments vs Docstrings

**File**: `src/charm.py` (various methods)

**Issue**: Many public methods lack docstrings. Some have inline comments instead.

**Example**:
```python
def _start_services(self) -> bool:
    """
    Starts all Landscape Server systemd services. Returns True if
    successful, False otherwise.
    """
    # Good! Has docstring
```

vs

```python
def _configure_smtp(self, relay_host: str) -> None:
    # Rewrite postfix config.  <-- Should be docstring
```

**Proposed Refactor**:
1. Add docstrings to all public/protected methods
2. Follow Google or NumPy docstring style
3. Run docstring linter (pydocstyle)

**Effort**: Medium (tedious but straightforward)

---

### 17. Configuration Option Documentation

**File**: `config.yaml`

**Issue**: Some config options have minimal descriptions. Operators may not understand implications.

**Example**:
```yaml
deployment_mode:
  type: string
  default: standalone
  description: |
    Landscape Server tenancy mode - do not modify unless you are able
    to provide the additional configuration required to run
    Landscape in SaaS mode.
```

This doesn't explain what "SaaS mode" is or what additional configuration is needed.

**Proposed Improvement**:
Expand descriptions with:
- What the option does
- When to use it
- Default behavior
- Examples
- Warnings/caveats

**Effort**: Low

---

## Summary

The Landscape Server charm is generally well-structured, but has accumulated technical debt typical of mature projects:

**Critical Issues** (should be addressed soon):
1. Error status handling (MaintenanceStatus vs BlockedStatus)
2. Deprecated leader-settings-changed event

**Refactoring Opportunities** (improve maintainability):
3. Hardcoded service configuration
4. Large monolithic charm class
5. Duplicated SSL certificate handling
6. Environment variable manipulation

**Code Quality** (clean up when touching nearby code):
7-13. Various minor issues (constants, return types, parameter lists, etc.)

**Testing** (ongoing improvement):
14-15. Integration and unit test gaps

**Documentation** (easy wins):
16-17. Docstrings and config descriptions

**Estimated Total Effort**: 3-6 months of engineering time to address all items systematically.

**Recommended Approach**:
1. Fix critical issues first (1-2)
2. Add missing tests (14-15) to prevent regressions
3. Refactor incrementally (3-6) when working in those areas
4. Clean up minor issues opportunistically (7-13)
5. Improve docs continuously (16-17)
