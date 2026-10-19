# ZMK Studio Messages

This repository contains the message definitions used to interract with ZMK Studio enabled devices.

## TODO

* Document transport protocol used with these messages
* Release/versioning strategy

## Fork changes: LED color maps

This is the [lenerdz/zmk-studio-messages](https://github.com/lenerdz/zmk-studio-messages) fork of [zmkfirmware/zmk-studio-messages](https://github.com/zmkfirmware/zmk-studio-messages). It adds a small set of requests to the `keymap` subsystem so that a ZMK Studio client can read and edit the per-layer LED color maps of the [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix) module.

Every upstream message and tag is unchanged. The new members of the `oneof` fields use tags `>= 100` (upstream uses small numbers today), so a future upstream addition cannot collide with them. ZMK stores the request tag in a `uint8_t`, so tags must stay below 256.

A firmware that does not include the handlers of `lenerdz/zmk-rgb-matrix` answers these requests with the `RPC_NOT_FOUND` simple error. That is how a client finds out that the feature is missing.

### Slots

A map has `slot_count` slots, and each slot holds one 24-bit `0xRRGGBB` color. On the keyboard this fork is used on (a Corne), `slot_count` is 54:

| Slots | Meaning |
|---|---|
| 0 to 41 | key positions (the same numbering as the keymap bindings) |
| 42 to 47 | left underglow, in LED chain order |
| 48 to 53 | right underglow, in LED chain order |

A map belongs to a layer and is addressed by that layer's id (`Layer.id` from `get_keymap`), so it follows the layer when the layers are reordered.

### Requests

All the requests are secured: the keyboard has to be unlocked, as for any keymap edit.

| Request (tag) | Payload | Response (tag) |
|---|---|---|
| `get_led_info` (100) | `bool` | `LedInfo`: `slot_count`, the ids of the layers that own a map, and `unsaved_changes` |
| `get_led_map` (101) | `uint32` layer id | `LedMap`: the layer id and exactly 54 colors |
| `set_led_colors` (102) | `SetLedColorsRequest`: layer id and up to 16 `LedColorEntry` (`slot`, `rgb`) | `SetLedColorsResponse`: `OK`, `INVALID_LAYER`, `INVALID_SLOT` or `BUSY` |
| `save_led_changes` (103) | `bool` | `SaveChangesResponse`, the same type the keymap uses |
| `discard_led_changes` (104) | `bool` | `bool` |
| `restore_led_defaults` (105) | `uint32` layer id | `LedMap`: the map after it went back to the firmware defaults |

The notification `unsaved_led_changes_status_changed` (tag 100) carries a `bool`: `true` when any map differs from what is saved.

An edit takes effect on the keyboard at once. It is kept across restarts only after `save_led_changes`. `discard_led_changes` brings every map back to the last saved state.

### Limits

The sizes come from `keymap.options.in` (nanopb, no heap and no callbacks):

| Field | Limit |
|---|---|
| `LedMap.colors` | exactly 54 entries |
| `LedInfo.layer_ids` | 8 entries |
| `SetLedColorsRequest.entries` | 16 entries (a client sends a larger edit in several requests) |

### Using it in a ZMK config

Pin this fork in the `west.yml` of the config repository. The project name and the path have to match the upstream entry of ZMK's `app/west.yml`, so this entry replaces it:

```yaml
    - name: zmk-studio-messages
      remote: lenerdz
      revision: <commit>
      path: modules/msgs/zmk-studio-messages
```
