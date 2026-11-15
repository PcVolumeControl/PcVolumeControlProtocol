# PcVolumeControl protocol

The wire protocol between a PcVolumeControl server (the program on the PC that
owns the audio mixer) and its clients (the apps that show and change volumes).

This repository holds only the specification. The implementations live in
their own repositories and follow what is written here.

- **[PROTOCOL.md](PROTOCOL.md)**: the specification.
- **[examples/](examples/)**: messages captured from a running server, usable
  as test fixtures.
- **[CHANGELOG.md](CHANGELOG.md)**: what each protocol version added.

## At a glance

- Raw TCP, port 3000 by default. One JSON object per line, UTF-8, ended by
  `\n`.
- The server sends the full mixer state on connect and again whenever anything
  changes. Clients send partial states to change the default device, the
  master volume and mute, or a session's volume and mute.
- Servers advertise `_pcvolumecontrol._tcp` over mDNS.
- The version on the wire is `7`. "Protocol v8" is a set of optional,
  additive extensions to v7: application icons, announced by the `icons`
  feature name, and an optional `detail` on each session, such as a browser
  tab's title, which clients show when present. A v7 client and a v8
  server, or the reverse, keep working together.

## Implementations

| Repository | Role |
| --- | --- |
| [PcVolumeControlServer](https://github.com/PcVolumeControl/PcVolumeControlServer) | Server for Windows and Linux, in Rust. Implements v7 and the v8 extensions. Its `mock_server` example serves demo data for client development. |
| [PcVolumeControlWindows](https://github.com/PcVolumeControl/PcVolumeControlWindows) | The original Windows server, in C#. v7. |
| [PcVolumeControlAndroid](https://github.com/PcVolumeControl/PcVolumeControlAndroid) | Android client. |
| [PcVolumeControliOS](https://github.com/PcVolumeControl/PcVolumeControliOS) | iOS client. |
| [PcVolumeControlPythonCli](https://github.com/PcVolumeControl/PcVolumeControlPythonCli) | Command-line client for experiments. Written for v6. |
| [PcVolumeControlESP32](https://github.com/PcVolumeControl/PcVolumeControlESP32) | ESP32 hardware client. |

## Changing the protocol

Shipped apps reject any `protocolVersion` other than `7`, and the Android app
ignores mDNS adverts whose `protocol` record is not `7`. So changes follow
these rules:

- **Additive only.** New optional fields on existing objects, new message
  types, new feature names. Never change the meaning or type of an existing
  field, and never make an optional field required.
- **The wire version stays 7.** Behaviour a client must detect before using
  it, such as a server that answers a new request, is announced by a feature
  name in `features`, not by a new number. A plain optional field needs no
  feature name: clients use it when present and ignore it otherwise.
- **A server with its extensions turned off sends exactly the v7 bytes.**
- **New server-to-client message types go only to clients that asked for
  them.** Older clients parse every line as a full state.
- **Clients ignore what they do not know:** fields, feature names and line
  types.

To propose a change, open an issue or a pull request here with the spec text
and example messages. An extension is part of the protocol once it is merged
here.

## License

MIT, see [LICENSE](LICENSE).
