# Communication
Once Discovery has completed, the app connects to the server on Port TCP/27000.

Example of connecting to TCP/27000 on a PS4 with IP 192.168.0.101 with netcat:

```
$ nc 192.168.0.101 27000
```

## Data Stream Format

Data is streamed via the TCP connection, bi-directionally, between the server and the app. All data is [little-endian](https://en.wikipedia.org/wiki/Endianness).

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
|----------|---------|---------------|
| A0000000 | 03      | HELLOWORLD    |

### Message Types
#### Type 0: Heartbeat

The server will periodically send a "heartbeat" packet of type 0 and no content. In other words, length 0 and type 0.

e.x.

```
00 00 00 00 00
```

| size(32) | type(8) | content(size) |
|----------|---------|---------------|
| 00000000 | 00      |               |

When the app receives a heartbeat, the app must send the same heartbeat (5 bytes of zeros) back to let the server know that the app is still running. If the app does not respond with a heartbeat, the server will close the TCP connection.

Heartbeats should be sent by the app only in response to the server.

#### Type 1: New Connection

Messages of type 1 are sent when the app first connects to the server. It contains a JSON string with the language and version of the game.

```JSON
{"lang": "de", "version": "1.1.30.0"}
```

#### Type 2: Busy

Messages of type 2 are sent when the server is busy. A server will be busy if a Pipboy Companion app is already connected. The message contains no data.

e.x.
```
00 00 00 00 02
```

#### Type 3: Data Update

Messages of type 3 contain updates to the database.
The database is a collection of values where each value is represented by an ID.

A Data Update payload is a sequence of data updates. Each update begins with a header:

```C
struct UpdateHeader {
  uint8_t type,
  uint32_t id
}
```

This is followed by data depending on the value type. The possible types are as follows:

| Type ID | Type     | Format of update data                                                                    |
| ------- | -------- | ---------------------------------------------------------------------------------------- |
| 0       | BOOLEAN  | `uint8_t`, value is true if data is non-zero                                             |
| 1       | INT8     | `int8_t`                                                                                 |
| 2       | UINT8    | `uint8_t`                                                                                |
| 3       | INT32    | `int32_t`                                                                                |
| 4       | UINT32   | `uint32_t`                                                                               |
| 5       | FLOAT    | `float32_t`                                                                              |
| 6       | STRING   | `0x00`-termined byte string                                                              |
| 7       | ARRAY    | `uint16_t length, uint32_t ids[length]`, ids are the value IDs of previously sent values |
| 8       | OBJECT   | See below.                                                                               |

Objects are complex. There are two parts - a set of (key, value) pairs to add, followed by a set of old values to remove.
The first time an object is sent, the remove set will be empty.

The first part, (key, value) pairs to add, begins with a `uint16_t length`,
followed by `length` lots of (`uint32_t id`, `0x00`-terminated byte string `key`).
This maps `key` to the previously sent value with ID `id`.

The second part is `uint16_t length, uint32_t ids[length]`, where `ids` contains the IDs of values
which the object currently maps to. Any (key, value) pair which maps to such a value should be removed.

Note that it is possible to contain a removal and an addition for the same key in one update, for example
if the update said to add the (key, value) pair ("foo", 1234) and remove the value which "foo" currently maps to.
In this case, the new value replaces the old value.

Objects are unordered and keys will not be repeated.

e.x.

        |               |      |           |            | bytes        |
------- | ------------- | ---- | --------- | ---------- | ------------ |
size    |               |      |           |            | 3b000000     |
type    |               |      |           |            | 03           |
content | first update  | type |           |            | 03           |
        |               | id   |           |            | 0a000000     |
        |               | data |           |            | 2a000000     |
        | second update | type |           |            | 07           |
        |               | id   |           |            | 0b000000     |
        |               | data | length    |            | 0200         |
        |               |      | first id  |            | 01000000     |
        |               |      | second id |            | 02000000     |
        | third update  | type |           |            | 08           |
        |               | id   |           |            | 0c000000     |
        |               | data | added     | length     | 0200         |
        |               |      |           | first id   | 05000000     |
        |               |      |           | first key  | 666f6f00     |
        |               |      |           | second id  | 06000000     |
        |               |      |           | second key | 68656c6c6f00 |
        |               |      | removed   | length     | 0200         |
        |               |      |           | first id   | 03000000     |
        |               |      |           | second id  | 04000000     |

corresponds to an update that:
* sets value with id `10` to be a `uint32` equal to `42`
* sets value with id `11` to be an array containing the values with ids `1, 2`
* updates value with id `12`, an object, to add keys `"foo": 5, "hello": 6` and remove values `3, 4`

#### Type 4: Local Map Update

Messages of type 4 contain binary image data of the current local map if you view the local map in the app.

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

Messages of type 5 are sent by the app to the server to request an action be taken in the game. The content of the message is a JSON string of the form:

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
