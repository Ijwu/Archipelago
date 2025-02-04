# Archipelago General Client
## Archipelago Connection Handshake
These steps should be followed in order to establish a gameplay connection with an Archipelago session.

1. Client establishes WebSocket connection to Archipelago server.
2. Server accepts connection and responds with a [RoomInfo](#RoomInfo) packet.
3. Client may send a [GetDataPackage](#GetDataPackage) packet.
4. Server sends a [DataPackage](#DataPackage) packet in return. (If the client sent GetDataPackage.)
5. Client sends [Connect](#Connect) packet in order to authenticate with the server.
6. Server validates the client's packet and responds with [Connected](#Connected) or [ConnectionRefused](#ConnectionRefused).
7. Server may send [ReceivedItems](#ReceivedItems) to the client, in the case that the client is missing items that are queued up for it.
8. Server sends [Print](#Print) to all players to notify them of the new client connection.

In the case that the client does not authenticate properly and receives a [ConnectionRefused](#ConnectionRefused) then the server will maintain the connection and allow for follow-up [Connect](#Connect) packet.

## Synchronizing Items
When the client receives a [ReceivedItems](#ReceivedItems) packet, if the `index` argument does not match the next index that the client expects then it is expected that the client will re-sync items with the server. This can be accomplished by sending the server a [Sync](#Sync) packet and then a [LocationChecks](#LocationChecks) packet.

Even if the client detects a desync, it can still accept the items provided in this packet to prevent gameplay interruption.

When the client receives a [ReceivedItems](#ReceivedItems) packet and the `index` arg is `0` (zero) then the client should accept the provided `items` list as its full inventory. (Abandon previous inventory.)

# Archipelago Protocol Packets
Packets are sent between the multiworld server and client in order to sync information between them. Below is a directory of each packet.

Packets are simple JSON lists in which any number of ordered network commands can be sent, which are objects. Each command has a "cmd" key, indicating its purpose. All packet argument types documented here refer to JSON types, unless otherwise specified.

An object can contain the "class" key, which will tell the content data type, such as "Version" in the following example.

Example:
```javascript
[{"cmd": "RoomInfo", "version": {"major": 0, "minor": 1, "build": 3, "class": "Version"}, "tags": ["WebHost"], ... }]
```

## (Server -> Client)
These packets are are sent from the multiworld server to the client. They are not messages which the server accepts.
* [RoomInfo](#RoomInfo)
* [ConnectionRefused](#ConnectionRefused)
* [Connected](#Connected)
* [ReceivedItems](#ReceivedItems)
* [LocationInfo](#LocationInfo)
* [RoomUpdate](#RoomUpdate)
* [Print](#Print)
* [PrintJSON](#PrintJSON)
* [DataPackage](#DataPackage)
* [Bounced](#Bounced)

### RoomInfo
Sent to clients when they connect to an Archipelago server.
#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| version | NetworkVersion | Object denoting the version of Archipelago which the server is running. See [NetworkVersion](#NetworkVersion) for more details. |
| tags | list\[str\] | Denotes special features or capabilities that the sender is capable of. Example: `WebHost` |
| password | bool | Denoted whether a password is required to join this room.|
| forfeit_mode | str | `auto`, `enabled`, `disabled`, `auto-enabled` or `goal`. |
| remaining_mode | str | `enabled`, `disabled`, `goal` |
| hint_cost | int | The amount of points it costs to receive a hint from the server. |
| location_check_points | int | The amount of hint points you receive per item/location check completed. ||
| players | list\[NetworkPlayer\] | Sent only if the client is properly authenticated (see [Archipelago Connection Handshake](#Archipelago-Connection-Handshake)). Information on the players currently connected to the server. See [NetworkPlayer](#NetworkPlayer) for more details. |
| datapackage_version | int | Data version of the [data package](#Data Package Contents) the server will send. Used to update the client's (optional) local cache. |
| datapackage_versions | dict[str, int] | Data versions of the individual games' data packages the server will send. |
| seed_name | str | uniquely identifying name of this generation |

#### forfeit_mode
Dictates what is allowed when it comes to a player forfeiting their run. A forfeit is an action which distributes the rest of the items in a player's run to those other players awaiting them.

* `auto`: Distributes a player's items to other players when they complete their goal.
* `enabled`: Denotes that players may forfeit at any time in the game.
* `auto-enabled`: Both of the above options together.
* `disabled`: All forfeit modes disabled.
* `goal`: Allows for manual use of forfeit command once a player completes their goal. (Disabled until goal completion)

#### remaining_mode
Dictates what is allowed when it comes to a player querying the items remaining in their run.

* `goal`: Allows a player to query for items remaining in their run but only after they completed their own goal.
* `enabled`: Denotes that players may query for any items remaining in their run (even those belonging to other players).
* `disabled`: All remaining item query modes disabled.

### ConnectionRefused
Sent to clients when the server refuses connection. This is sent during the initial connection handshake.
#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| errors | list\[str\] | Optional. When provided, should contain any one of: `InvalidSlot`, `SlotAlreadyTaken`, `IncompatibleVersion`, or `InvalidPassword`. |

### Connected
Sent to clients when the connection handshake is successfully completed.
#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| team | int | Your team number. See [NetworkPlayer](#NetworkPlayer) for more info on team number. |
| slot | int | Your slot number on your team. See [NetworkPlayer](#NetworkPlayer) for more info on the slot number. |
| players | list\[NetworkPlayer\] | List denoting other players in the multiworld, whether connected or not. See [NetworkPlayer](#NetworkPlayer) for info on the format. |
| missing_locations | list\[int\] | Contains ids of remaining locations that need to be checked. Useful for trackers, among other things. |
| checked_locations | list\[int\] | Contains ids of all locations that have been checked. Useful for trackers, among other things. |
| slot_data | dict | Contains a json object for slot related data, differs per game. Empty if not required. |

### ReceivedItems
Sent to clients when they receive an item.
#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| index | int | The next empty slot in the list of items for the receiving client. |
| items | list\[NetworkItem\] | The items which the client is receiving. See [NetworkItem](#NetworkItem) for more details. |

### LocationInfo
Sent to clients to acknowledge a received [LocationScouts](#LocationScouts) packet and responds with the item in the location(s) being scouted.
#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| locations | list\[NetworkItem\] | Contains list of item(s) in the location(s) scouted. See [NetworkItem](#NetworkItem) for more details. |

### RoomUpdate
Sent when there is a need to update information about the present game session. Generally useful for async games.
#### Arguments
The arguments for RoomUpdate are identical to [RoomInfo](#RoomInfo) barring two:

| Name | Type | Notes |
| ---- | ---- | ----- |
| hint_points | int | New argument. The client's current hint points. |
| players | list\[NetworkPlayer\] | Changed argument. Always sends all players, whether connected or not. |

All arguments for this packet are optional.

### Print
Sent to clients purely to display a message to the player.
#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| text | str | Message to display to player. |

### PrintJSON
Sent to clients purely to display a message to the player. This packet differs from [Print](#Print) in that the data being sent with this packet allows for more configurable or specific messaging.
#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| data | list\[JSONMessagePart\] | See [JSONMessagePart](#JSONMessagePart) for more details on this type. |
| type | str | May be present to indicate the nature of this message. Known types are Hint and ItemSend.
| receiving | int | Is present if type is Hint or ItemSend and marks the destination player's ID.
| item | NetworkItem | Is present if type is Hint or ItemSend and marks the source player id, location id and item id.

### DataPackage
Sent to clients to provide what is known as a 'data package' which contains information to enable a client to most easily communicate with the Archipelago server. Contents include things like location id to name mappings, among others; see [Data Package Contents](#Data-Package-Contents) for more info.

#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| data | DataPackageObject | The data package as a JSON object. More details on its contents may be found at [Data Package Contents](#Data-Package-Contents) |

### Bounced
Sent to clients after a client requested this message be sent to them, more info in the Bounce package.

#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| data | dict | The data in the Bounce package copied |


## (Client -> Server)
These packets are sent purely from client to server. They are not accepted by clients.

* [Connect](#Connect)
* [Sync](#Sync)
* [LocationChecks](#LocationChecks)
* [LocationScouts](#LocationScouts)
* [StatusUpdate](#StatusUpdate)
* [Say](#Say)
* [GetDataPackage](#GetDataPackage)
* [Bounce](#Bounce)

### Connect
Sent by the client to initiate a connection to an Archipelago game session.

#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| password | str | If the game session requires a password, it should be passed here. |
| game | str | The name of the game the client is playing. Example: `A Link to the Past` |
| name | str | The player name for this client. |
| uuid | str | Unique identifier for player client. |
| version | NetworkVersion | An object representing the Archipelago version this client supports. |
| tags | list\[str\] | Denotes special features or capabilities that the sender is capable of. |

#### Authentication
Many, if not all, other packets require a successfully authenticated client. This is described in more detail in [Archipelago Connection Handshake](#Archipelago-Connection-Handshake).

### Sync
Sent to server to request a [ReceivedItems](#ReceivedItems) packet to synchronize items.
#### Arguments
No arguments necessary.

### LocationChecks
Sent to server to inform it of locations that the client has checked. Used to inform the server of new checks that are made, as well as to sync state.

#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| locations | list\[int\] | The ids of the locations checked by the client. May contain any number of checks, even ones sent before; duplicates do not cause issues with the Archipelago server. |

### LocationScouts
Sent to the server to inform it of locations the client has seen, but not checked. Useful in cases in which the item may appear in the game world, such as 'ledge items' in A Link to the Past. The server will always respond with a [LocationInfo](#LocationInfo) packet with the items located in the scouted location.

#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| locations | list\[int\] | The ids of the locations seen by the client. May contain any number of locations, even ones sent before; duplicates do not cause issues with the Archipelago server. |

### StatusUpdate
Sent to the server to update on the sender's status. Examples include readiness or goal completion. (Example: defeated Ganon in A Link to the Past)

#### Arguments
| Name | Type | Notes |
| ---- | ---- | ----- |
| status | int | One of [Client States](#Client-States). Follow the link for more information. |

### Say
Basic chat command which sends text to the server to be distributed to other clients.

#### Arguments
| Name | Type | Notes |
| ------ | ----- | ------ |
| text | str  | Text to send to others. |

### GetDataPackage
Requests the data package from the server. Does not require client authentication.

#### Arguments
| Name | Type | Notes |
| ------ | ----- | ------ |
| exclusions | list[str]  | Optional. If specified, will not send back the specified data. Such as, ["Factorio"] -> Datapackage without Factorio data.|

### Bounce
Send this message to the server, tell it which clients should receive the message and 
the server will forward the message to all those targets to which any one requirement applies.

#### Arguments
| Name | Type | Notes |
| ------ | ----- | ------ |
| games | list[str] | Optional. Game names that should receive this message |
| slots | list[int] | Optional. Player IDs that should receive this message |
| tags | list[str] | Optional. Client tags that should receive this message |
| data | dict | Any data you want to send |


## Appendix
### NetworkPlayer
A list of objects. Each object denotes one player. Each object has four fields about the player, in this order: `team`, `slot`, `alias`, and `name`. `team` and `slot` are ints, `alias` and `name` are strs.

Each player belongs to a `team` and has a `slot`. Team numbers start at `0`. Slot numbers are unique per team and start at `1`. Slot number `0` refers to the Archipelago server; this may appear in instances where the server grants the player an item.

`alias` represents the player's name in current time. `name` is the original name used when the session was generated. This is typically distinct in games which require baking names into ROMs or for async games.

```python
from typing import NamedTuple
class NetworkPlayer(NamedTuple):
    team: int
    slot: int
    alias: str
    name: str
```

Example:
```js
[
    {"team": 0, "slot": 1, "alias": "Lord MeowsiePuss", "name": "Meow"}, 
    {"team": 0, "slot": 2, "alias": "Doggo", "name": "Bork"},
    {"team": 1, "slot": 1, "alias": "Angry Duck", "name": "Angry Quack"},
    {"team": 1, "slot": 2, "alias": "Mountain Duck", "name": "Honk"}
]
```

### NetworkItem
Items that are sent over the net (in packets) use the following data structure and are sent as objects:
```python
from typing import NamedTuple
class NetworkItem(NamedTuple):
    item: int
    location: int
    player: int
```
In JSON this may look like:
```js
[
    {"item": 1, "location": 1, "player": 0},
    {"item": 2, "location": 2, "player": 0},
    {"item": 3, "location": 3, "player": 0}
]
```

### JSONMessagePart
Message nodes sent along with [PrintJSON](#PrintJSON) packet to be reconstructed into a legible message. The nodes are intended to be read in the order they are listed in the packet.

```python
from typing import TypedDict, Optional
class JSONMessagePart(TypedDict):
    type: Optional[str]
    color: Optional[str]
    text: Optional[str]
```

`type` is used to denote the intent of the message part. This can be used to indicate special information which may be rendered differently depending on client. How these types are displayed in Archipelago's ALttP client is not the end-all be-all. Other clients may choose to interpret and display these messages differently.
Possible values for `type` include:
* player_id
* item_id
* location_id

`color` is used to denote a console color to display the message part with. This is limited to console colors due to backwards compatibility needs with games such as ALttP. Although background colors as well as foreground colors are listed, only one may be applied to a [JSONMessagePart](#JSONMessagePart) at a time.

Color options:
* bold
* underline
* black
* red
* green
* yellow
* blue
* magenta
* cyan
* white
* black_bg
* red_bg
* green_bg
* yellow_bg
* blue_bg
* purple_bg
* cyan_bg
* white_bg

`text` is the content of the message part to be displayed.

### Client States
An enumeration containing the possible client states that may be used to inform the server in [StatusUpdate](#StatusUpdate).

```python
import enum
class CLientStatus(enum.IntEnum):
    CLIENT_UNKNOWN = 0
    CLIENT_READY = 10
    CLIENT_PLAYING = 20
    CLIENT_GOAL = 30
```

### NetworkVersion
An object representing software versioning. Used in the [Connect](#Connect) packet to allow the client to inform the server of the Archipelago version it supports.
```python
from typing import NamedTuple
class Version(NamedTuple):
    major: int
    minor: int
    build: int
```

### Data Package Contents
A data package is a JSON object which may contain arbitrary metadata to enable a client to interact with the Archipelago server most easily. Currently, this package is used to send ID to name mappings so that clients need not maintain their own mappings.

We encourage clients to cache the data package they receive on disk, or otherwise not tied to a session. You will know when your cache is outdated if the [RoomInfo](#RoomInfo) packet or the datapackage itself denote a different version. A special case is datapackage version 0, where it is expected the package is custom and should not be cached.

Note: 
 * Any ID is unique to its type across AP: Item 56 only exists once and Location 56 only exists once.
 * Any Name is unique to its type across its own Game only: Single Arrow can exist in two games.

#### Contents
| Name | Type | Notes |
| ------ | ----- | ------ |
| games | dict[str, GameData] | Mapping of all Games and their respective data |
| version | int | Sum of all per-game version numbers, for clients that don't bother with per-game caching/updating. |

#### GameData
GameData is a **dict** but contains these keys and values. It's broken out into another "type" for ease of documentation.
| Name | Type | Notes |
| ---- | ---- | ----- |
| item_name_to_id | dict[str, int] | Mapping of all item names to their respective ID. |
| location_name_to_id | dict[str, int] | Mapping of all location names to their respective ID. |
| version | int | Version number of this game's data |
