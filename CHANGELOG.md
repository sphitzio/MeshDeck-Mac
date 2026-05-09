# Changelog

All notable changes to this project will be documented in this file.

---

## [1.0.3-beta] - UI Stall Workaround / Config Tab Interface Binding

### Fixed

- Prevented a UI stall that could occur immediately after the initial NodeDB load.
- MeshDeck could connect successfully, load nodes, insert the local node into the table, and then become unresponsive.
- The last visible log before the stall was typically:

```text
Local node inserted in table: !xxxxxxxx
```

- The status bar could still show the network as ready, confirming that the issue happened after connection and NodeDB loading, not during the initial connection itself.

### Cause

The stall was isolated to the direct signal connection in `main.py`:

```python
self.worker.interface_ready.connect(self.config_tab.set_interface)
```

After the worker completed the initial connection and NodeDB setup, it emitted `interface_ready` with the live Meshtastic interface object.

That signal immediately called:

```python
config_tab.set_interface(...)
```

This appears to trigger blocking interface/config work on the Qt UI thread, causing the main UI to freeze after the NodeDB load completes.

### Workaround Applied

The direct Config tab interface binding was disabled:

```python
# self.worker.interface_ready.connect(self.config_tab.set_interface)
```

### Result

- MeshDeck no longer stalls after the local node is inserted into the table.
- NodeDB loads successfully.
- The node list remains responsive.
- The main UI becomes usable after connection.
- This confirms that the freeze is related to Config tab initialization rather than the node table, watchdog, or NodeDB batch loading.

### Known Side Effect

The Config tab may no longer automatically receive the live Meshtastic interface after connection.

This may affect:

- reading node configuration,
- editing node configuration,
- saving configuration changes,
- reboot-related config workflows.

The rest of the app appears to remain usable with this workaround.

### Recommended Proper Fix

This workaround should not be considered the final fix.

The proper fix should preserve Config tab functionality while avoiding blocking calls on the Qt UI thread.

Recommended approaches:

- Load the Config tab interface only when the Config tab is opened.
- Defer `config_tab.set_interface(...)` until after the initial NodeDB/UI load has completed.
- Move blocking config reads/writes into a worker thread.
- Update the Config tab UI via Qt signals after config data is available.
- Avoid direct synchronous reads from the live Meshtastic interface during initial connection.

### Suggested Future Implementation

Instead of this direct connection:

```python
self.worker.interface_ready.connect(self.config_tab.set_interface)
```

Use a deferred or lazy-loading flow, for example:

```python
self.worker.interface_ready.connect(self._on_interface_ready)
```

Then:

```python
def _on_interface_ready(self, iface):
    self._pending_config_interface = iface

    # Only initialize Config tab if it is currently visible,
    # or defer until the user opens the Config tab.
    if self.tab_widget.currentIndex() == self.TAB_CONFIG:
        QTimer.singleShot(500, lambda: self.config_tab.set_interface(iface))
```

And when the Config tab is opened:

```python
def _on_tab_changed(self, index):
    if index == self.TAB_CONFIG and getattr(self, "_pending_config_interface", None):
        self.config_tab.set_interface(self._pending_config_interface)
```

If `set_interface()` performs heavy reads, it should still be moved into a worker thread instead of running directly on the UI thread.

### Files Involved

- `main.py`
- `tabs/tab_config.py`
- `worker.py`

### Notes

This issue is separate from the connection watchdog.

The watchdog only handles failed reconnect attempts. In this case, the connection is already established and the NodeDB has already loaded successfully.

The stall happens after the worker emits local node/interface readiness signals.
