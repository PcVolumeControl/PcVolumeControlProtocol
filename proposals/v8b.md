# Proposal: v8b, a reworked protocol with differential updates

Status: draft, set aside. Nothing here is implemented.

Version 8, as described in [PROTOCOL.md](../PROTOCOL.md), keeps the message
design of version 7: the server sends the whole state on every change and
clients send partial states. This proposal is the fuller redesign that was
considered alongside it: typed messages, a handshake, explicit commands, and
updates that carry only what changed. It was judged more than the protocol
needs for now and is kept for reference. If it is ever adopted it will need a
version number of its own, since it does not interoperate with version 8;
where the text below says "v8" for its own messages, read it as that future
version.

It stays newline-delimited JSON over TCP. Binary framing and pairing are a
separate step, in [v9.md](v9.md).

## Goals

- One version number that means what it says: `8` on the wire and in mDNS.
- The server sends what changed, not every device and session each time.
- Clients send commands that say what they want, instead of partial states
  with precedence rules.
- A client that does something wrong is told why.
- Drop everything that only existed to keep v7 clients working.

## What v7 clients and servers do with v8

There is no migration path. A v8 side and a v7 side do not interoperate; the
user updates both.

- **Released Android app, v8 server:** it parses the first line, sees a
  `protocolVersion` other than 7, shows its "wrong version" message and
  disconnects.
- **Released iOS app, v8 server:** it does not check the version. The first
  line fails to decode as a full state and the app reports a connection
  failure, with no hint about updating.
- **v8 app, v7 server:** the first line it receives is a v7 full state with
  `protocolVersion` 7. The app tells the user to update the server.

## Framing

Unchanged: raw TCP, default port 3000, one JSON object per line, UTF-8, ended
by `\n`. A trailing `\r` is tolerated and blank lines are ignored.

Every message is an object with a `type` string. Both sides ignore members
they do not know, and ignore messages whose `type` they do not know. That is
how later additions stay compatible within v8.

## Connection start

1. The server sends `hello` as soon as the connection opens.
2. The client sends its own `hello`. It is required: it must be the client's
   first message and arrive within 10 seconds, or the server closes the
   connection with the error `timeout`.
3. The server sends `state`, then `update` messages as things change.

The server sends no state before the client's `hello`, because that message
says which version the client speaks and which parts of the state it wants.

```json
{"type":"hello","protocolVersion":8,"applicationVersion":"v11","serverName":"wiglaf"}
{"type":"hello","protocolVersion":8,"clientName":"PcVolumeControl for Android 5.0"}
```

`protocolVersion` appears only in the two `hello` messages. If the versions
differ, the server sends an `error` with code `versionMismatch` and closes; a
client that reads a server `hello` with another version closes and tells the
user which side is older.

`serverName` and `clientName` are for display: the app can show which PC it
reached, and the server can list who is connected.

### Asking for less

A client's `hello` may carry `want`, a list of the parts of the state it has
a use for. Without it the client gets everything.

```json
{"type":"hello","protocolVersion":8,"clientName":"Desk knob","want":[]}
{"type":"hello","protocolVersion":8,"clientName":"Small display","want":["sessions"]}
```

| Part | Leaving it out means |
| --- | --- |
| `sessions` | `sessions` is always an empty array, and no update is sent for a session change. |
| `devices` | `devices` is always an empty array. `defaultDeviceId` and `master` are still sent. |
| `icons` | Sessions never carry `iconId`, and no update is sent when one becomes known. |
| `details` | Sessions never carry `detail`, and no update is sent when only a detail changes. |

`defaultDeviceId` and `master` are always sent. A client that wants only
those, such as a hardware volume knob, sends an empty list: it receives one
small `state` and after that a message only when the master volume, the mute
or the default device changes.

The point is less the bytes than the wake-ups: a tab title can change once a
second, and a client with nowhere to show it should not have to read and
parse a message each time. Unknown names in `want` are ignored. Commands are
not restricted by it: a client that left out `sessions` may still send
`setSession` if it knows an id.

## State

The full state, sent once after the handshake and again after `resync`:

```json
{
  "type": "state",
  "rev": 41,
  "devices": [
    {"id": "alsa_output.usb-headset", "name": "Headset"},
    {"id": "alsa_output.pci-analog", "name": "Speakers"}
  ],
  "defaultDeviceId": "alsa_output.pci-analog",
  "master": {"volume": 80.0, "muted": false},
  "sessions": [
    {"id": "s-95", "name": "Firefox", "volume": 100.0, "muted": false,
     "iconId": "4b13f0d55a3779a6ff892414f951be3f", "detail": "Big Buck Bunny - YouTube"},
    {"id": "s-127", "name": "Spotify", "volume": 65.0, "muted": false}
  ]
}
```

| Member | Type | Notes |
| --- | --- | --- |
| `rev` | integer | Revision of this state. See [Revisions](#revisions). |
| `devices` | array | Active playback devices, in the server's order. |
| `defaultDeviceId` | string or null | `null` when there is no playback device. |
| `master` | object or null | Volume (0.0 to 100.0) and mute of the default device. `null` when there is none. |
| `sessions` | array | Sessions on the default device only, in the server's order. |
| `sessions[].iconId` | string | Optional. As in [PROTOCOL.md](../PROTOCOL.md#icon-ids). |
| `sessions[].detail` | string | Optional. As in [PROTOCOL.md](../PROTOCOL.md#session-details). |

The state describes one device: the PC's default playback device. That
device is shared, not chosen per client. `setDefaultDevice` changes the PC's
default for everyone, and every client then receives that device's master
volume and sessions.

Changes from v7: `deviceIds` (a map) becomes `devices` (an ordered array);
the default device's id, volume and mute are top-level; a server with no
playback device says so instead of staying silent. Device and session ids are
opaque: the v7 rule that a session id contains its device id is gone.

That frees the server to hand out short session ids of its own, such as
`s-95`, and map them to whatever the audio system uses. On Windows the
native id is about 200 characters, and ids are stored by clients and sent
back in commands, so this matters most to small clients. A session id means
nothing outside the connection it was received on.

## Updates

After the first `state`, the server sends only what changed:

```json
{"type":"update","rev":42,"master":{"volume":41.0}}
{"type":"update","rev":43,"sessions":{"changed":[{"id":"s-95","detail":"Sintel - YouTube"}]}}
{"type":"update","rev":44,"sessions":{"changed":[{"id":"s-127","muted":true}],"removed":["s-95"]}}
{"type":"update","rev":45,"sessions":{"added":[{"id":"s-140","name":"Discord","volume":90.0,"muted":false}]}}
{"type":"update","rev":46,"devices":[{"id":"alsa_output.pci-analog","name":"Speakers"}]}
{"type":"update","rev":47,"defaultDeviceId":"alsa_output.usb-headset","master":{"volume":55.0,"muted":false},"sessions":{"reset":[]}}
```

Every member except `type` and `rev` is optional, and an absent member means
"unchanged".

| Member | Meaning |
| --- | --- |
| `master` | The members present replace the client's values. |
| `devices` | The whole device list, when it changed. It is short, so it is never patched. |
| `defaultDeviceId` | The new default device. The same update carries a full `master` and `sessions.reset`. |
| `sessions.removed` | Ids of sessions that ended. |
| `sessions.changed` | Objects with an `id` and the members that changed. A member set to `null` is removed (`detail`, `iconId`). |
| `sessions.added` | Complete session objects, appended to the end of the list in the order given. Nothing else changes the order: there is no reordering of existing sessions. |
| `sessions.reset` | A complete session list that replaces the client's. Never combined with the other three. |

A client applies one update in this order: `devices`, `defaultDeviceId`,
`master`, then `sessions.reset`, or else `sessions.removed`, `changed`,
`added`.

The server gathers changes that arrive close together (a few tens of
milliseconds) into one update. A change in `detail` alone is sent no sooner
than one second after the previous update to that client, as today.

An entry in `changed` for an id the client does not have, or an entry in
`added` for one it already has, means the client's copy is wrong: it asks for
a `resync`.

### Revisions

`rev` starts at any value in `state` and each `update` on that connection is
exactly one higher. Over TCP nothing is lost or reordered, so a gap can only
come from a bug on one side. A client that sees a gap sends `resync` and the
server answers with a fresh `state`. That is cheap insurance, and it gives
logs on both sides a common reference.

## Commands

Client to server. Each says one thing:

```json
{"type":"setMaster","volume":64.0}
{"type":"setMaster","muted":true}
{"type":"setSession","id":"s-127","volume":29.0}
{"type":"setSession","id":"s-127","muted":true,"seq":17}
{"type":"setDefaultDevice","deviceId":"alsa_output.usb-headset"}
{"type":"getIcons","ids":["4b13f0d55a3779a6ff892414f951be3f"],"size":96}
{"type":"resync"}
```

- `setMaster` and `setSession` take `volume`, `muted` or both. Changing one no
  longer requires sending the other, which removes the race where a volume
  drag on one client overwrote a mute from another.
- Volumes outside 0 to 100 are clamped.
- The change itself comes back to every client, the sender included, as an
  `update`. A command that changes nothing (the value is already set)
  produces no update.
- `seq` is optional on any command: a number the client chooses. A command
  that carries one gets exactly one reply naming it in `re`: `ok`, or an
  `error`.

```json
{"type":"ok","re":17}
```

`ok` means the server accepted the command and the audio system took the
change, or, for `getIcons` and `resync`, that the request was accepted and
its messages follow. It says nothing about ordering: the `update` describing
the change usually arrives just after the `ok`, and a client must not depend
on which comes first. A command without `seq` gets no `ok`, which suits a
client sending many values during a slider drag: it can leave `seq` off those
and put one on the value the drag ends with.

This replaces the v7 rule that a client message is a partial state checked in
a fixed order (device, then master, then sessions), along with the habit it
forced on clients of echoing back values they had not changed.

## Icons

The model is unchanged from [PROTOCOL.md](../PROTOCOL.md#icon-ids): ids are
derived from the image, clients cache by id and request only what they lack.
What changes:

- The request is the `getIcons` command above. At most 32 ids per command and
  64 outstanding per connection.
- Every server answers `getIcons`; a server with no icons answers
  `notFound`.
- Replies are typed, and still go only to the client that asked:

```json
{"type":"icon","id":"4b13f0d55a3779a6ff892414f951be3f","size":96,"mime":"image/png","data":"iVBORw0KGgo..."}
{"type":"icon","id":"nosuchicon","error":"notFound"}
```

## Errors

```json
{"type":"error","code":"unknownSession","message":"no session s-95","re":17}
{"type":"error","code":"malformed","message":"line 3: volume must be a number","fatal":true}
```

| Code | Fatal | When |
| --- | --- | --- |
| `versionMismatch` | yes | The client's `hello` has another version. |
| `timeout` | yes | No `hello` within 10 seconds of connecting. |
| `malformed` | yes | Not JSON, not an object, a wrong type, or no `hello` first. |
| `tooLarge` | yes | A client line over 64 KiB. |
| `tooManyIcons` | yes | More than 32 ids in one command, or more than 64 outstanding. |
| `unknownSession` | no | `setSession` for a session that is gone. |
| `unknownDevice` | no | `setDefaultDevice` for a device that is gone. |
| `notSettable` | no | The audio system refused the change. |

After a fatal error the server closes the connection. After any other error
the client keeps its state as it is: its optimistic change was not applied,
and no update will arrive to say so.

In v7 the server closed the connection without a word for every one of these.

## Keep-alive

A phone that walks out of Wi-Fi range leaves a connection that looks open to
the server, and a server that loses power looks the same to the phone. Either
side can check:

```json
{"type":"ping"}
{"type":"pong"}
```

- Whoever receives `ping` answers `pong` at once.
- Any message counts as a sign of life, not only `pong`.
- A side that has received nothing for 30 seconds sends `ping`. If nothing
  arrives in the next 10 seconds it treats the connection as lost: the server
  closes it and forgets the client, and a client reconnects.

Answering `ping` is required of every client, including one that asked for
nothing but the master volume. Sending it is optional for clients that have
their own way of noticing a lost network.

## Discovery

`_pcvolumecontrol._tcp` as today, with the new version number in the TXT
`protocol` record.

## What is removed compared with version 8

- `protocolVersion` in every message.
- Partial-state client messages and their precedence rules.
- Telling server lines apart by which key they contain: `type` says it.
- The `deviceIds` map and the nested `defaultDevice` object.
- The rule that a session id contains its device id.
- Silent disconnects.

## What it costs to build

Nothing below has shipped, so all of it can change together.

| Where | Work |
| --- | --- |
| PcVolumeControlServer | New wire models and parser; the broadcaster computes one update per change and keeps a revision; commands map straight onto the audio backend; error replies; one update per distinct `want` list; short session ids mapped to the backend's. The audio backends, the desktop app and icon resolution are unaffected. |
| PcVolumeControlAndroid (open PR) | Replace the line classifier and Gson state model with typed messages and a state store that applies updates; commands replace the partial-state writers. Icon cache, request planning and rows stay. |
| PcVolumeControliOS (draft PR) | The same swap in the repository and view model. Icon cache, planner and views stay. |
| KMP client | The same swap in its protocol module. |
| Python CLI, ESP32 | Rewrite their small protocol code, or leave them on v7 with an old server. |
| This repository | PROTOCOL.md rewritten for v8, new examples. |

Because new apps will not talk to the C# server, Windows users need the Rust
server released and checked on Windows before v8 apps reach the stores.

## Left for a later version

- **Sessions on every device.** The state covers the default device only, as
  in v7. Describing all devices at once would change each client's screens,
  not only its protocol code.
