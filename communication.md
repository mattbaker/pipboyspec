# Communication
Once Discovery has completed, the App connects to the Server on Port TCP/27000.

Example of connecting to TCP/27000 on a PS4 with IP 192.168.0.101:

```
nc 192.168.0.101 27000 #where this IP is the IP address of your PC or PS4
```

## Data Stream Format

Data is streamed via the TCP connection, bi-directionally, between the Server and the App. All data is [little-endian](https://en.wikipedia.org/wiki/Endianness).

The stream is composed of individual Messages of the form:

```C
struct Message {
  uint32_t size,
  uint8_t type,
  uint8_t content[size]
}
```

e.x.

```
A0 00 00 00 03 48 45 4C 4C 4F 57 4F 52 4C 44
```

| size(32) | type(8) | content(size) |
|------------------------------------|
| A0000000 | 03      | HELLOWORLD    |

### Message Types
#### Type 0: Heartbeat

The Server will periodically send a "heartbeat" packet of type 0 and no content. In other words, length 0 and type 0.

e.x.

```
00 00 00 00 00
```

| size(32) | type(8) | content(size) |
|------------------------------------|
| 00000000 | 00      |               |

When the App receives a heartbeat, the App must send the same heartbeat (5 bytes of zeros) back to let the Server know that the App is still running. If the App does not respond with a heartbeat, the Server will close the TCP connection.

Heartbeats should be sent by the App only in response to the Server.

#### Type 1: New Connection

Messages of type 1 are sent when the App first connects to the Server. It contains a JSON string with the language and version of the game.

```JSON
{"lang": "de", "version": "1.1.30.0"}
```

#### Type 2: Busy

Messages of type 2 are sent when the Server is busy. A Server will be busy if a Pipboy Companion App is already connected. The message contains no data.

e.x.
```
00 00 00 00 02
```

#### Type 3: Data Update

#### Type 4: Local Map Update

Messages of type 4 contain binary image data of the current local map if you view the local map in the App.

```
struct Extend {
  float32_t x,
  float32_t y
}

struct Map {
      uint32_t width,
      uint32_t height,
      Extend nw,
      Extend ne,
      Extend sw,
      uint8_t pixel[ width * height ]
}
```

#### Type 5: Command Request

Messages of type 5 are sent by the App to the Server to request an action be taken in the game. The content of the message is a JSON string of the form:

```JSON
{"type": 1, "args": [4207600675, 7, 494, [0, 1]], "id": 3}
```

 * The command type (type) is seen in range of 0 to 14
 * The arguments to the command (args) differ based on the type property
 * The id of the command increments with every command send.

|  Command Type  |  Args                                                         |  Comment                                                                                                            |
|----------------|---------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
|  0             |  `[ <HandleId>, 0, <$.Inventory.Version> ]`                   |  Use an instance of item specified by `<HandleId>`                                                                  |
|  1             |  `[ <HandleId>, <count>, <$.Inventory.Version>, <StackID> ]`  |  Drop `<count>` instances of item, `<StackID>` is the whole list under `StackID`                                    |
|  2             |  `[<HandleId>, <StackID>, <position>, <$.Inventory.Version>]` |  Put item on favorite `<position>` counts from far left 0 to right 5, and north 6 to south 11                       |
|  3             |  `[<ComponentFormId>, <$.Inventory.Version>]`                 |  Toggle *Tag for search* on component specified by `<ComponentFormId>`                                              |
|  4             |  `[<page>]`                                                   |  Cycle through search mode on inventory page ( 0: Weapons, 1: Apparel, 2: Aid, 3: Misc, 4: Junk, 5: Mods, 6: Ammo ) |
|  5             |  `[<QuestId>, ??, ??]`                                        |  Toggle marker for quest                                                                                            |
|  6             |  `[ <x>, <y>, false ]`                                        |  Place custom marker at `<x>,<y>`                                                                                   |
|  7             |  `[]`                                                         |  remove custom marker                                                                                               |
|  8             |   |                                                           |                                                                                                                     |
|  9             |  `[<id>]`                                                     |  Fast travel to location with index `<id>` in database                                                              |
|  10            |   |                                                           |                                                                                                                     |
|  11            |   |                                                           |                                                                                                                     |
|  12            |  `[<id>]`                                                     |  Toggle radio with index `<id>` in database                                                                         |
|  13            |  `[]`                                                         |  Toggle receiving of local map update                                                                               |
|  14            |  `[]`                                                         |  Refresh?? Command with no result                                                                                   |

#### Type 6: Command Response

Messages of type 6 are responses to commands. Currently, it appears that responses are only received for commands of type 9.

```JSON
{"allowed":true,"id":3,"success":true}
```
