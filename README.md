# ZMK Studio Messages

This repository contains the message definitions used to interract with ZMK Studio enabled devices.

## TODO

* Document transport protocol used with these messages
* Release/versioning strategy

## Fork changes: LED color maps

This is the [lenerdz/zmk-studio-messages](https://github.com/lenerdz/zmk-studio-messages) fork of [zmkfirmware/zmk-studio-messages](https://github.com/zmkfirmware/zmk-studio-messages). It adds a small set of requests to the `keymap` subsystem so that a ZMK Studio client can read and edit the per-layer LED color maps of the [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix) module, and its display (OLED) settings, which include the layout of the screens (see [Fork changes: display settings](#fork-changes-display-settings)).

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

The sizes come from `keymap.options.in` (nanopb, no heap). The fields below are arrays with a fixed capacity; `DisplayInfo.settings` is the one list that has none (see [Fork changes: display settings](#fork-changes-display-settings)):

| Field | Limit |
|---|---|
| `LedMap.colors` | exactly 54 entries |
| `LedInfo.layer_ids` | 15 entries (layer ids 0 to 14) |
| `SetLedColorsRequest.entries` | 16 entries (a client sends a larger edit in several requests) |

## Fork changes: display settings

The same fork adds three more requests to the `keymap` subsystem, for the display (OLED) settings that the [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix) module keeps: the contrast of each screen, when the screens turn off, the light toast of [lenerdz/zmk-nice-oled](https://github.com/lenerdz/zmk-nice-oled) and, with the OLED layout, which widgets of its screens are drawn, where, and which animation the right screen plays. Their tags are 106 to 108, in the same reserved range (`>= 100`), and a firmware without the handlers answers them with `RPC_NOT_FOUND`, like the light requests.

The settings are a list of numbered integers. For each setting it has, the firmware reports the current value and the range, so a client can draw a control without knowing the setting; the meaning of the ids is fixed (table below) so that a client can also label them. A change takes effect on the keyboard at once and the firmware keeps it across restarts by itself, so there is no save or discard request and no notification. All the requests are secured: the keyboard has to be unlocked.

The list has no fixed size. `DisplayInfo.settings` has no `max_count` in `keymap.options.in`, so nanopb makes it a callback field: the firmware writes as many entries as it has, one after the other, and a client reads all that arrive. The wire format is the one of any repeated field, so the generated TypeScript client does not change, and the response does not enlarge the union that ZMK Studio keeps on its stack.

| Request (tag) | Payload | Response (tag) |
|---|---|---|
| `get_display_info` (106) | `bool` | `DisplayInfo`: the list of `DisplaySetting` (`id`, `value`, `min`, `max`, `default_value`) |
| `set_display_setting` (107) | `SetDisplaySettingRequest`: `id` and `value` | `SetDisplaySettingResponse`: `OK`, `INVALID_ID` or `INVALID_VALUE` |
| `restore_display_defaults` (108) | `bool` | `DisplayInfo`: the settings after they went back to the firmware defaults |

The ids this fork defines (the firmware of [lenerdz/zmk-rgb-matrix](https://github.com/lenerdz/zmk-rgb-matrix) is the reference; ids stay below 256, because the firmware sends them to a peripheral in 8 bits):

| Id | Setting | Range | Default |
|---|---|---|---|
| 1 | Contrast of the left (central) screen | 0 to 255 | 128 |
| 2 | Contrast of the right (peripheral) screen | 0 to 255 | 128 |
| 3 | When the screen goes off after the keyboard goes idle: 0 never, 1 only on battery (it stays on while USB is connected), 2 always | 0 to 2 | 0 |
| 4 | Seconds without activity before the screen goes off; not less than the idle timeout of the keyboard (`CONFIG_ZMK_IDLE_TIMEOUT`, 30 s by default); only used when id 3 is not 0 | the idle timeout to 3600 | 60 |
| 5 | Light toast on the right screen: 0 off, 1 on (only has an effect when the right half is built with the toast widget) | 0 to 1 | 1 |
| 6 | How long the toast stays on screen, in milliseconds | 500 to 5000 | 1500 |

From id 16 up come the settings of the OLED layout: one block of eight ids per widget, at `16 + 8 * n`. In a block, `+0` is whether the widget is drawn (0 or 1, default 1), `+1` is its x and `+2` its y (pixels, -160 to 320) and, for the animation of the right screen, `+3` is its kind and `+4` the time of one cycle. The firmware lists only the widgets it was built with, and the defaults are what the build says (the `CONFIG_NICE_OLED_WIDGET_*` options of [lenerdz/zmk-nice-oled](https://github.com/lenerdz/zmk-nice-oled)), so a client has to read them from `default_value`.

| Ids | Widget | Screen |
|---|---|---|
| 16 to 20 | animation or still image: shown, x, y, kind (0 crystal, 1 cat, 2 head, 3 spaceman, 4 pokemon, 5 vim, 6 vip_marcos), time of one cycle in milliseconds (0, or 100 to 60000; 0 is the time of the animation) | right |
| 24 to 26 | battery of the right half | right |
| 32 to 34 | Bluetooth icon | right |
| 41 and 42 | light toast: x and y of its first line (it has no switch of its own: the switch is id 5) | right |
| 48 to 50 | USB or Bluetooth icon of the active output | left |
| 56 to 58 | battery | left |
| 64 to 66 | the five Bluetooth profile dots | left |
| 72 to 74 | number of the active Bluetooth profile | left |
| 80 to 82 | name of the active layer | left |
| 88 to 90 | WPM gauge | left |
| 96 to 98 | WPM number | left |
| 104 to 106 | WPM graph | left |

The coordinates are those of the canvas the widgets are drawn on, with one exception, the animation of the right screen. The conditions of each entry and the conversion of the coordinates of the animation are in the section [OLED layout](https://github.com/lenerdz/zmk-rgb-matrix/blob/corne-fixes/README.md#oled-layout) of the README of the module.

A client should draw a control from the `min`, `max` and `default_value` the firmware reports, not from this table: the table says what each id means. A firmware may omit a setting it does not have, and a client should show an id it does not know as a plain number.

## Using it in a ZMK config

Pin this fork in the `west.yml` of the config repository. The project name and the path have to match the upstream entry of ZMK's `app/west.yml`, so this entry replaces it:

```yaml
    - name: zmk-studio-messages
      remote: lenerdz
      revision: <commit>
      path: modules/msgs/zmk-studio-messages
```
