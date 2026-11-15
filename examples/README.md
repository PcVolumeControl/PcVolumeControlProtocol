# Example messages

Captured from the reference server's `mock_server` example, which serves
scripted demo sessions. They are pretty-printed here for reading: on the wire
each message is a single line ended by `\n`.

| File | Direction | What it is |
| --- | --- | --- |
| `full-state.json` | server to client | A full state, with `iconId` on every session and `detail` on two of them. |
| `update-default-device.json` | client to server | Change the default device. |
| `update-master.json` | client to server | Change the master volume and mute. |
| `update-session.json` | client to server | Change one session's volume and mute. |
| `icon-request.json` | client to server | Ask for two icons at 16 pixels. The second id does not exist. |
| `icon-reply.json` | server to client | The reply carrying a PNG. |
| `icon-reply-not-found.json` | server to client | The reply for the unknown id. |

There is one full state shape. A server that serves no icons sends the same
state without any `iconId`, and answers every requested id like
`icon-reply-not-found.json`. A server set not to send details leaves out
`detail`.

The ids in the update examples refer to the devices and sessions in
`full-state.json`. Session ids are opaque. The icon ids of the mock server
start with `demo`; a real server derives them from a hash of the image.
