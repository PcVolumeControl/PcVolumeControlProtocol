# Example messages

Captured from the reference server's `mock_server` example, which serves
scripted demo sessions. They are pretty-printed here for reading: on the wire
each message is a single line ended by `\n`.

| File | Direction | What it is |
| --- | --- | --- |
| `full-state-v7.json` | server to client | A full state from a server with no extensions. |
| `full-state-v8.json` | server to client | A full state with `features`, `iconId` and `detail`. |
| `update-default-device.json` | client to server | Change the default device. |
| `update-master.json` | client to server | Change the master volume and mute. |
| `update-session.json` | client to server | Change one session's volume and mute. |
| `icon-request.json` | client to server | Ask for two icons at 16 pixels. The second id does not exist. |
| `icon-reply.json` | server to client | The reply carrying a PNG. |
| `icon-reply-not-found.json` | server to client | The reply for the unknown id. |

The ids in the update examples refer to the devices and sessions in
`full-state-v7.json`. The icon ids of the mock server start with `demo`; a
real server derives them from a hash of the image.
