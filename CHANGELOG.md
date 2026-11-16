# Protocol changelog

Newest first. Dates are when the change landed in the reference server.

## v8 (2026-11)

A breaking version: `protocolVersion` is `8` in every message and the mDNS
`protocol` record is `8`. It does not interoperate with v7; each side
disconnects on the other's version.

- **Application icons.** Sessions may carry an `iconId`. Clients fetch the
  image with an `iconRequest` message and receive `icon` reply lines, which
  go only to the client that asked. Ids are derived from the image content,
  so clients can cache by id indefinitely. Every server answers
  `iconRequest`; one that serves no icons answers `notFound`.
- **Session details.** Sessions may carry an optional `detail`, a short label
  that tells apart sessions of the same application, such as a browser tab's
  title.
- A line from the server is either a full state (`defaultDevice`) or an icon
  reply (`icon`).
- **Session ids are opaque.** The rule that a session id contains its device
  id is gone.

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
