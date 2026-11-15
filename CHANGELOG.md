# Protocol changelog

Newest first. Dates are when the change landed in the reference server.

## v8 extensions (2026-11)

Optional and additive. The version on the wire stays `7`. Behaviour a client
must detect is announced through `features` in the full state and the
`features` mDNS TXT record.

- **`icons`** feature: sessions may carry an `iconId`. Clients fetch the image
  with an `iconRequest` message and receive `icon` reply lines. Ids are
  derived from the image content, so clients can cache by id indefinitely.
- Sessions may carry a `detail`, a short label that tells apart sessions of
  the same application, such as a browser tab's title. It is an ordinary
  optional member with no feature name: clients show it when present.
- A line from the server is now either a full state (`defaultDevice`) or an
  icon reply (`icon`). Icon replies go only to the client that asked.

## mDNS discovery (2026-06)

Servers advertise `_pcvolumecontrol._tcp` with TXT records `version=1` and
`protocol=7`. No change to the messages.

## v7 (2018-01)

- The version field is `protocolVersion`; the earlier `version` field is no
  longer accepted.
- The full state gains `applicationVersion`, the server build, separate from
  the protocol version.

## v6 (2017-12)

- After applying a client's update, the server sends the new full state to
  every connected client.

## v5 (2017-12)

- Sessions carry an `id`, and session updates address a session by it.
- First documented version: newline-terminated JSON over TCP, full state on
  connect, partial states as client updates.

## v1 to v4 (2017-11)

A `version` field was added to the messages, and servers and clients began
disconnecting on a mismatch.
