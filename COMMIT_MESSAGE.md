fix: prevent post-NodeDB UI stall from Config tab interface binding

Disabled the direct `interface_ready -> config_tab.set_interface` connection after identifying it as the cause of a UI stall immediately after initial NodeDB load.

MeshDeck now remains responsive after connection and NodeDB loading.

This is a temporary workaround. The Config tab may no longer automatically receive the live Meshtastic interface after connection. A follow-up should move Config tab interface loading to a deferred, lazy, or asynchronous flow.
