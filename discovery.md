# Discovery
Before data can be sent and received, the App must discover the server.

### PC & PS4

The App sends a broadcast message to **UDP/28000** in the form of the following JSON string:

```json
{"cmd": "autodiscover"}
```

The server responds to the app over UDP with a message of the form:

```json
{"IsBusy": false, "MachineType": "PC"}
```

### XBox One

The app sends a 16 byte broadcast message to port **UDP/5050**. The format is not currently known.
