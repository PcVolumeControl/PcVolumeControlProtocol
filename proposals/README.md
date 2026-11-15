# Proposals

Drafts under discussion. Nothing in this folder is part of the protocol, and
no implementation should follow it until it has been merged into
[PROTOCOL.md](../PROTOCOL.md).

| Proposal | What it changes |
| --- | --- |
| [v8-rework.md](v8-rework.md) | Makes v8 a breaking version: typed messages, a handshake, explicit commands with acknowledgements, error replies, a keep-alive, and updates that carry only what changed. Still JSON lines. No open questions. |
| [v9.md](v9.md) | On top of that: pairing, TLS, and binary frames that carry icons as bytes. |
