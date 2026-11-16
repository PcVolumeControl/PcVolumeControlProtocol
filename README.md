# PcVolumeControl protocol

The wire protocol between a PcVolumeControl server (the program on the PC that
owns the audio mixer) and its clients (the apps that show and change volumes).

This repository holds only the specification. The implementations live in
their own repositories and follow what is written here.

- **[PROTOCOL.md](PROTOCOL.md)**: the specification.
- **[examples/](examples/)**: messages captured from a running server, usable
  as test fixtures.
- **[CHANGELOG.md](CHANGELOG.md)**: what each protocol version changed.

## At a glance

- Raw TCP, port 3000 by default. One JSON object per line, UTF-8, ended by
  `\n`.
- The server sends the full mixer state on connect and again whenever anything
  changes. Clients send partial states to change the default device, the
  master volume and mute, or a session's volume and mute.
- Sessions may carry an `iconId`; clients fetch the images with an
  `iconRequest`, which every server answers, and the replies go only to the
  client that asked. Sessions may also carry a `detail`, such as a browser
  tab's title, which clients show when present.
- Servers advertise `_pcvolumecontrol._tcp` over mDNS.
- The version on the wire is `8`, in every message and in the mDNS
  `protocol` record. Version 8 does not interoperate with version 7.

## Implementations

| Repository | Role |
| --- | --- |
| [PcVolumeControlServer](https://github.com/PcVolumeControl/PcVolumeControlServer) | Server for Windows and Linux, in Rust. Speaks v8. Its `mock_server` example serves demo data for client development. |
| [PcVolumeControlWindows](https://github.com/PcVolumeControl/PcVolumeControlWindows) | The original Windows server, in C#. Speaks v7 and does not work with v8 apps. |
| [PcVolumeControlAndroid](https://github.com/PcVolumeControl/PcVolumeControlAndroid) | Android client. Released versions speak v7; v8 support is in progress. |
| [PcVolumeControliOS](https://github.com/PcVolumeControl/PcVolumeControliOS) | iOS client. Released versions speak v7; v8 support is in progress. |
| [PcVolumeControlPythonCli](https://github.com/PcVolumeControl/PcVolumeControlPythonCli) | Command-line client for experiments. Written for v6. |
| [PcVolumeControlESP32](https://github.com/PcVolumeControl/PcVolumeControlESP32) | ESP32 hardware client. |

## Changing the protocol

- **A change that an existing client or server would misread gets a new
  version number.** That includes changing the meaning or type of a field,
  making an optional field required, and a new message that existing
  clients would take for a full state.
- **Versions do not interoperate.** Each side checks `protocolVersion`, and
  on a mismatch it disconnects and tells the user clearly which side is
  older and needs updating.
- **Additive members that every side can ignore do not need a new
  version.** Clients and servers ignore fields they do not know, so a new
  optional field that changes nothing for those who skip it can be added
  within a version.

To propose a change, open an issue or a pull request here with the spec text
and example messages. A change is part of the protocol once it is merged
here.

## License

MIT, see [LICENSE](LICENSE).
