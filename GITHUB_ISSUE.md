# UI stalls after NodeDB load when Config tab receives live interface

## Environment

- MeshDeck 1.0.3-beta
- macOS
- PyQt5 / PyQtWebEngine
- Serial connection

## Problem

MeshDeck connects successfully and loads the initial NodeDB, but the UI can stall immediately after the local node is inserted into the table.

The last visible log before the stall is usually:

```text
Local node inserted in table: !xxxxxxxx
```

The status bar may show that the network is ready and nodes are loaded, which suggests that the issue happens after connection and NodeDB loading, not during initial connection.

## Isolation Test

The issue was isolated by commenting out this line in `main.py`:

```python
self.worker.interface_ready.connect(self.config_tab.set_interface)
```

Changed to:

```python
# self.worker.interface_ready.connect(self.config_tab.set_interface)
```

After disabling that direct signal connection, the app no longer stalls.

## Likely Cause

The worker emits `interface_ready` after the initial connection and NodeDB setup. This directly calls `config_tab.set_interface`, which appears to perform blocking interface/config work on the Qt UI thread.

Because this happens immediately after NodeDB loading, the main UI can become unresponsive even though the connection itself is already established.

## Expected Behaviour

MeshDeck should remain responsive after connecting and loading the NodeDB.

The Config tab should load interface/config data without blocking the rest of the UI.

## Actual Behaviour

MeshDeck becomes unresponsive after loading the local node and initial NodeDB.

## Current Workaround

Disable the direct signal connection:

```python
# self.worker.interface_ready.connect(self.config_tab.set_interface)
```

## Known Side Effect

The Config tab may no longer automatically receive the live Meshtastic interface after connection.

This may affect:

- reading node configuration,
- editing node configuration,
- saving configuration changes,
- reboot-related config workflows.

## Suggested Fix

The Config tab should not synchronously read from the live Meshtastic interface during initial connection.

Possible fixes:

- defer Config tab loading,
- lazy-load Config tab interface only when opened,
- move config reads into a worker thread,
- emit data back to the UI through Qt signals,
- avoid direct synchronous config/interface work inside the `interface_ready` signal path.

## Suggested Implementation Direction

Instead of:

```python
self.worker.interface_ready.connect(self.config_tab.set_interface)
```

Use a deferred handler:

```python
self.worker.interface_ready.connect(self._on_interface_ready)
```

Example:

```python
def _on_interface_ready(self, iface):
    self._pending_config_interface = iface

    if self.tab_widget.currentIndex() == self.TAB_CONFIG:
        QTimer.singleShot(500, lambda: self.config_tab.set_interface(iface))
```

Then initialize the Config tab when it is actually opened:

```python
def _on_tab_changed(self, index):
    if index == self.TAB_CONFIG and getattr(self, "_pending_config_interface", None):
        self.config_tab.set_interface(self._pending_config_interface)
```

If `set_interface()` performs heavy reads, those should still be moved into a worker thread.

## Notes

This does not appear to be a watchdog issue.

The watchdog handles failed reconnect attempts. In this case, the connection is already established and the NodeDB has loaded successfully.
