# Harness to State-Transition Migration Guide

This guide provides step-by-step instructions for migrating your `test_charm.py` file from Harness to state-transition tests.
https://documentation.ubuntu.com/ops/latest/howto/legacy/migrate-unit-tests-from-harness/#harness-migration

## Why Migrate?

1. **Harness is deprecated** - Will not be supported in future Ops releases
2. **Better isolation** - Each test starts with a clean slate
3. **More realistic** - Tests match actual charm deployment scenarios
4. **Automatic status collection** - Framework handles collect_status events
5. **Future-proof** - State-transition is the recommended approach

## Migration Strategy

### Option 1: Incremental Migration (Recommended)

1. **Keep existing test class structure** - No need to change to functions
2. **Migrate tests one by one** within the same TestCharm class
3. **Use helper methods** for common patches and setup
4. **Run tests frequently** to ensure no regressions

### Option 2: Create New Test File

Create a separate file and migrate gradually, keeping both during transition.

## Key Migration Findings

### 1. Class Structure Can Be Preserved

**✅ KEEP** the existing `unittest.TestCase` class structure:
```python
class TestCharm(unittest.TestCase):
    def setUp(self):
        # Remove harness setup, add helper for patches
        self.common_patches = {
            "user_exists": patch("charm.user_exists"),
            "group_exists": patch("charm.group_exists"),
            # ... other common patches
        }
    
    def _setup_common_mocks(self, additional_patches=None):
        """Helper method to set up common patches for state-transition tests."""
        # Implementation here
```

### 2. Import Changes

**Remove:**
```python
from ops.testing import Harness
```

**Keep/Add:**
```python
from ops.testing import (
    Context,
    MaintenanceStatus,
    PeerRelation,
    Relation,
    State,
    StoredState,
)

# Add for error handling
try:
    from scenario.errors import UncaughtCharmError
except ImportError:
    class UncaughtCharmError(Exception):
        pass
```

### 3. Error Handling Pattern (CRITICAL)

State-transition tests wrap charm exceptions in `UncaughtCharmError`:

**Before:**
```python
with patches as mocks:
    mocks["apt"].add_package.side_effect = PackageNotFoundError
    self.assertRaises(PackageNotFoundError, harness.begin_with_initial_hooks)
```

**After:**
```python
with patches as mocks:
    mocks["apt"].add_package.side_effect = PackageNotFoundError
    try:
        ctx.run(ctx.on.install(), state_in)
        assert False, "Expected UncaughtCharmError to be raised"
    except UncaughtCharmError as e:
        # Verify that the underlying exception is PackageNotFoundError
        assert "PackageNotFoundError" in str(e)
```
### 4. Helper Method Pattern (PROVEN EFFECTIVE)

Create a common helper to manage patches across tests:

```python
def _setup_common_mocks(self, additional_patches=None):
    """Helper method to set up common patches for state-transition tests."""
    patches = {**self.common_patches}
    if additional_patches:
        patches.update(additional_patches)
    
    started_mocks = {}
    for name, patcher in patches.items():
        started_mocks[name] = patcher.start()
    
    # Set up standard return values
    started_mocks["user_exists"].return_value = Mock(spec_set=struct_passwd, pw_uid=1000)
    started_mocks["group_exists"].return_value = Mock(spec_set=struct_group, gr_gid=1000)
    
    self.addCleanup(lambda: [p.stop() for p in patches.values()])
    return started_mocks
```

### 5. Installation Test Pattern

**Before:**
```python
def test_install(self):
    harness = Harness(LandscapeServerCharm)
    relation_id = harness.add_relation("replicas", "landscape-server")
    harness.update_relation_data(relation_id, "landscape-server", {"leader-ip": "test"})
    
    with patches as mocks:
        harness.begin_with_initial_hooks()
    # assertions
```

**After:**
```python
def test_install(self):
    """Test successful charm installation with peer relation - migrated to state-transition."""
    ctx = Context(LandscapeServerCharm)
    
    # Set up peer relation as in original test
    peer_relation = PeerRelation(
        endpoint="replicas",
        peers_data={1: {"leader-ip": "test"}}
    )
    state_in = State(relations={peer_relation})

    # Set up patches using helper
    additional_patches = {
        "check_call": patch("charm.check_call"),
        "apt": patch("charm.apt"),
        "prepend_default_settings": patch("charm.prepend_default_settings"),
        "update_service_conf": patch("charm.update_service_conf"),
    }
    
    mocks = self._setup_common_mocks(additional_patches)
    
    # Add the additional mock objects
    for name, patcher in additional_patches.items():
        mocks[name] = patcher.start()
    
    state_out = ctx.run(ctx.on.install(), state_in)
    
    # Verify calls and status
    mocks["apt"].add_package.assert_called_once_with(
        ["landscape-server", "landscape-hashids"],
        update_cache=True,
    )
    assert isinstance(state_out.unit_status, WaitingStatus)
```

### 6. Configuration With Relations

**Before:**
```python
harness.update_config({"ssl_cert": "CERT_DATA"})
harness.add_relation("replicas", "landscape-server")
harness.begin_with_initial_hooks()
```

**After:**
```python
peer_relation = PeerRelation(
    endpoint="replicas",
    peers_data={1: {"leader-ip": "test"}}
)

state_in = State(
    relations={peer_relation},
    config={"ssl_cert": "CERT_DATA"}
)
state_out = ctx.run(ctx.on.install(), state_in)
```

### 7. Database Relation Tests

**Before:**
```python
mock_event = Mock()
mock_event.relation.data = {
    mock_event.unit: {
        "allowed-units": self.harness.charm.unit.name,
        "master": "host=1.2.3.4 password=testpass",
        "host": "1.2.3.4",
        "port": "5678",
        "user": "testuser",
        "password": "testpass",
    },
}
self.harness.charm._db_relation_changed(mock_event)
```

**After:**
```python
db_relation = Relation(
    endpoint="db",
    remote_app_name="postgresql", 
    remote_app_data={},
    remote_units_data={0: {
        "allowed-units": "landscape-server/0",  # Use typical unit name
        "master": "host=1.2.3.4 password=testpass",
        "host": "1.2.3.4",
        "port": "5678", 
        "user": "testuser",
        "password": "testpass",
    }}
)

state_in = State(relations={db_relation})
state_out = ctx.run(ctx.on.relation_changed(db_relation), state_in)
```

### 8. Status Checks

**Before:**
```python
status = self.harness.charm.unit.status
self.assertIsInstance(status, WaitingStatus)
```

**After:**
```python
assert isinstance(state_out.unit_status, WaitingStatus)
assert state_out.unit_status.message == "Expected message"
```

## Migration Process - Real World

### Successful Migration Pattern

Based on the actual migration of TestCharm class in landscape-charm:

1. **Start with setUp method** - Replace harness initialization with helper patterns
2. **Create _setup_common_mocks helper** - Manages common patches consistently
3. **Migrate individual test methods** - One at a time, keeping existing test names
4. **Test frequently** - Use `tox -e unit -- tests/unit/test_charm.py::TestCharm::test_methodname -s`
5. **Verify no warnings** - Harness deprecation warnings should disappear

### Tests Successfully Migrated

✅ `test_init` - Basic initialization  
✅ `test_install` - Installation with peer relations  
✅ `test_install_package_not_found_error` - Error handling  
✅ `test_install_package_error` - PackageError handling  
✅ `test_install_called_process_error` - CalledProcessError handling  
✅ `test_install_add_apt_repository_with_proxy` - Proxy configuration  
✅ `test_install_ssl_cert` - SSL certificate configuration  
✅ `test_install_license_file` - License file handling  
✅ `test_install_license_file_b64` - Base64 license handling  
✅ `test_db_relation_changed` - Database relation integration  
✅ `test_db_manual_configs_used` - Manual DB configuration override  

### Testing Commands

```bash
# Test individual migrated method
tox -e unit -- tests/unit/test_charm.py::TestCharm::test_install -s

# Test multiple migrated methods
tox -e unit -- tests/unit/test_charm.py::TestCharm::test_init tests/unit/test_charm.py::TestCharm::test_install -s

# Run all tests (mixed harness and state-transition)
tox -e unit -- tests/unit/test_charm.py
```

## Key Findings

### 1. No Harness Warnings ✅
Migrated tests produce NO deprecation warnings when run individually or together.

### 2. Preserved Test Structure ✅
Existing `unittest.TestCase` class structure works perfectly with state-transition tests.

### 3. Error Handling Pattern Works ✅
The `UncaughtCharmError` wrapper pattern correctly handles charm exceptions.

### 4. Database Relations Work ✅
Complex relation data can be properly modeled with `Relation` objects.

### 5. Configuration Integration ✅
Config overrides and relation data integration works as expected.

### 6. Mock Management ✅
The helper method approach scales well for managing many patches across tests.

## Remaining Work

After this migration, approximately **45+ test methods** still need migration from the TestCharm class. The patterns established above should handle most scenarios:

- Action tests
- Complex multi-relation scenarios  
- Leader/follower tests
- Service management tests
- Configuration validation tests
- Bootstrap account tests

Use the same patterns and helper methods for consistent migration.

## Common Issues and Solutions

### 1. UncaughtCharmError Handling ⚠️
- **Issue**: Charm exceptions are wrapped in `UncaughtCharmError`
- **Solution**: Always catch `UncaughtCharmError` and check the underlying exception in `str(e)`

### 2. Stored State Access 🔧
- **Issue**: `self.harness.charm._stored.ready` doesn't work
- **Solution**: Use `state_out.get_stored_state("_stored", owner_path="CharmClassName")` (if available)
- **Alternative**: Test behavior through status changes and mock calls

### 3. Unit Names in Relations 📝
- **Issue**: `self.harness.charm.unit.name` in relation data
- **Solution**: Use realistic unit names like `"landscape-server/0"`

### 4. Multiple Patch Management 🛠️
- **Issue**: Managing many patches across tests
- **Solution**: Use the `_setup_common_mocks()` helper pattern with additional_patches

### 5. Error vs Success Flows ⚡
- **Issue**: Same test trying to test both success and error
- **Solution**: Separate into different test methods for clarity

### 6. Import Issues 📦
- **Issue**: `ScalarStateSnapshot` not available in some ops versions
- **Solution**: Avoid advanced stored state features, test through observable behavior

### 7. Proxy Environment Variables 🌐
- **Issue**: Environment variable testing
- **Solution**: Use `@patch.dict(os.environ, {...})` decorator as before

## Final Migration Tips

### 1. Start Simple
Begin with tests that have minimal dependencies and build up complexity.

### 2. Test One Method at a Time  
Run individual tests frequently: `tox -e unit -- tests/unit/test_charm.py::TestCharm::test_methodname -s`

### 3. Preserve Test Semantics
Keep the same test logic and assertions, just change the setup mechanism.

### 4. Use Descriptive Comments
Add migration comments: `"""Test description - migrated to state-transition."""`

### 5. Verify No Warnings
Ensure migrated tests produce no Harness deprecation warnings.

### 6. Keep Existing Names
Preserve original test method names for consistency and easier review.

## Benefits After Migration

✅ **No deprecation warnings** - Tests run cleanly  
✅ **Isolated tests** - Each test starts fresh  
✅ **Realistic scenarios** - Match actual charm behavior  
✅ **Future-proof** - Compatible with latest Ops  
✅ **Better debugging** - Clearer state transitions  
✅ **Maintainable structure** - Helper methods scale well  
✅ **Preserved semantics** - Same test coverage and logic
