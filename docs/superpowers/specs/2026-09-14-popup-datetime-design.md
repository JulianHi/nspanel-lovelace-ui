# `popupDatetime` — setting an `input_datetime` time from the panel

Status: draft, approved direction pending final read-through.
Scope: **US Portrait** Nextion variant only (`HMI/US/portrait/nspanel_US_P.HMI`), plus both HA backends
(`apps/nspanel-lovelace-ui/luibackend/` AppDaemon app and `nspanel-lovelace-ui/rootfs/usr/bin/mqtt-manager/`
add-on). EU/landscape variants, date-picking, and AM/PM are explicitly out of scope (see "Out of scope").

## Goal

Let a user set the time-of-day on an HA `input_datetime` helper (`has_time: true`, `has_date: false`) from
the panel — e.g. to configure an alarm or an automation trigger time — and optionally flip a linked
boolean-ish entity (e.g. "alarm enabled") from the same popup.

## Why a new page instead of extending `popupTimer`

`popupTimer`'s MM:SS editable-number + zoom-button UX is the part that works well and is worth reusing.
Its lifecycle buttons (Start/Pause/Cancel/Finish, driven by `timer.*` services) are unrelated to
`input_datetime` (which has no start/pause/active state) and are the part described as buggy. Bolting a
second entity domain onto that page would tangle two unrelated service-call paths into an already fragile
page. Instead: clone `popupTimer` → `popupDatetime`, delete everything timer-lifecycle-specific, keep and
simplify the number-editing mechanism.

## Config

New optional per-entity YAML keys, valid for `entity:` items where the underlying HA domain is
`input_datetime` (time-only). Parsed like `effectList` already is today — via the existing raw-dict
fallback (`entity_config.entity_input_config.get(...)` in the AppDaemon app), so **no config schema class
changes are required**, only doc updates (`docs/entities.md`) and reading the two new keys where the
datetime popup payload is built.

| key | optional | type | description |
|---|---|---|---|
| `toggle_entity` | yes | string | Any toggleable HA entity (`input_boolean`, `switch`, `automation`, …). When set, the popup shows an enable/disable toggle that acts on this entity via the existing generic `OnOff` button handler. |
| `subtitle` | yes | string | Free text shown as a small instruction/description line on the popup. Independent of `toggle_entity` — usable with or without it. |

Example:

```yaml
entities:
  - entity: input_datetime.alarm_weekday
    name: Wake up alarm
    icon: mdi:alarm
    subtitle: "Weekdays only"
    toggle_entity: input_boolean.alarm_weekday_enabled
```

## Wire protocol

### Row rendering (`cardEntities` / `cardGrid` / `cardGrid2`)

New `entityTypePanel = "datetime"` (mirrors the existing `"timer"` branch) for `entityType == "input_datetime"`
entities that are time-only (`has_time=true`, `has_date=false`; if an `input_datetime` has a date component it
falls back to being rendered as plain `"text"`, same as today — out of scope to handle date+time here).

### Opening the popup

1. Row tap on the display: existing per-slot handler in `cardEntities`/`cardGrid`/`cardGrid2` gets a new
   branch next to the existing `if(type1.txt=="timer")` ones:
   ```
   if(type1.txt=="datetime")
   {
       pageIcons.tTmp1.txt=dispName1.txt   // entity name, same as other branches
       pageIcons.tTmp2.txt=entn1.txt       // entity id
       pageIcons.tTmp3.txt=icon1.txt       // icon
       page popupDatetime
   }
   ```
   (repeated for slots 2–6, matching the existing per-slot pattern exactly.)
2. `popupDatetime` Preinitialize event (same boilerplate as `popupTimer`'s, minus `tTime`):
   reads `tEntity.txt`/`entn.txt`/`tIcon1.txt` from `pageIcons.tTmp1-3`, resets `dirty=0`, hides the debug
   fields (`tSend`,`tTmp`,`tInstruction`,`tId`), then sends:
   `event,pageOpenDetail,popupDatetime,<entity_id>`

3. Backend `detail_open` (new case) → `generate_datetime_detail_page(entity_id, True)` sends:

   `entityUpdateDetail~{entity_id}~~{icon_color}~{entity_id}~{hour}~{minute}~{toggle_entity_id}~{toggle_state}~{subtitle}`

   | # | field | notes |
   |---|---|---|
   | 0 | `entityUpdateDetail` | instruction |
   | 1 | `entity_id` | match key against `entn.txt`, same convention as every other popup |
   | 2 | *(blank)* | reserved, unused — kept for positional consistency with `popupTimer`'s layout |
   | 3 | `icon_color` | |
   | 4 | `entity_id` | stored into `entn.txt` |
   | 5 | `hour` | `0`–`23`, from `input_datetime` state `HH:MM:SS` |
   | 6 | `minute` | `0`–`59` |
   | 7 | `toggle_entity_id` | empty string if `toggle_entity` not configured → toggle row hidden |
   | 8 | `toggle_state` | `1`/`0`; only meaningful if field 7 non-empty |
   | 9 | `subtitle` | empty string if not configured → subtitle line hidden |

4. `tmSerial` on `popupDatetime` parses this (same `spstr ...,"~",N` pattern as `popupTimer`), then:
   - `n1.val` = hour, `n2.val` = minute
   - `n1.pco = 63488` (active/red), `n2.pco = defaultFontColor` — **hour is active by default**
   - `entToggle.txt` = field 7; `vis bToggle,` 1 if non-empty else 0; if visible, set its on/off visual from field 8
   - `tSubtitle.txt` = field 9; `vis tSubtitle,` 1 if non-empty else 0
   - `dirty=0`

### Editing

- `n1`/`n2` Touch Press Event: no-op if already active (`pco==63488`); otherwise arm this one
  (`pco=63488`) and disarm the other (`pco=defaultFontColor`). *(Unlike `popupTimer`, there is no
  "disable both" state and no edit-mode toggle — exactly one of the two is always active.)*
- `bZ1P/M` … `bZ4P/M`: unchanged from `popupTimer` except the hour clamp changes from `59`→`23` when acting
  on `n1`. Add `dirty=1` at the end of each of the 8 handlers.
- `t0` (the "+/- 1  +/- 5  +/- 10  +/- 15" label) and all 8 `bZ*` buttons are **always visible** — no
  `fToggleEdit`-style show/hide. `fToggleEdit` is removed from the page entirely.
- `bToggle` (new) Touch Press Event, only reachable when visible:
  ```
  tSend.txt="event,buttonPress2,"
  tSend.txt+=entToggle.txt+","
  tSend.txt+="OnOff,"
  tSend.txt+= <"0" if currently on else "1">
  // crc-frame + send, flip local visual state optimistically
  ```
  This reuses the existing entity-agnostic `OnOff` button handler verbatim — **no new backend code for the
  toggle action itself**, only for reporting its id/state on open (step 3 above).

### Closing (save-on-exit, manual or timeout)

`tmSleep`'s existing timeout mechanism already works by simulating `click b0,1` / `click b0,0` — i.e.
timeout *is* a synthetic press of the same exit button used for a manual back-tap. So putting the save
logic in `b0`'s Touch Press Event covers both cases with one code path:

```
Button b0 Touch Press Event:
if(dirty==1)
{
    tSend.txt="event,buttonPress2,"
    tSend.txt+=entn.txt+","
    tSend.txt+="datetime-set,"
    covx n1.val,strTmp.txt,0,0
    if(n1.val<10){ tSend.txt+="0"+strTmp.txt } else { tSend.txt+=strTmp.txt }
    tSend.txt+=":"
    covx n2.val,strTmp.txt,0,0
    if(n2.val<10){ tSend.txt+="0"+strTmp.txt } else { tSend.txt+=strTmp.txt }
    // crc-frame + send
    dirty=0
}
tSend.txt="event,buttonPress2,popupDatetime,bExit"
// crc-frame + send (unchanged from popupTimer's b0 handler)
```

Zero-padding is added here (`popupTimer`'s original minute/second conversion wasn't zero-padded — fine for
a `timer.start` duration string, not safe to rely on for `input_datetime.set_datetime`'s `HH:MM:SS` format).

Backend: `button_press` new case `datetime-set` → `apis.ha_api.get_entity(entity_id).call_service("set_datetime", time=f"{value}:00")`
(mirrors the existing `timer-start` case). `bExit` needs no new code — the existing generic
`if button_type == "bExit": render_card(current_card)` already handles it.

## Component list for `popupDatetime` (clone of `popupTimer`, then edited)

**Keep, unchanged:** `p0`, `b0`, `tEntity`, `tIcon1`, `tId`, `tInstruction`, `tTmp`, `tSend`, `strCommand`,
`tmSerial`, `tc0`.

**Keep, modified:** `n1`/`n2` (simplified arming logic, hour clamp 0–23 for `n1`), `bZ1-4 P/M` (add
`dirty=1`, hour clamp), `t0` (always visible), `tmSleep` (unchanged mechanism, now also triggers the save
path via `b0`).

**Remove:** `b1`, `b2`, `b3`, `va1`, `va2`, `va3`, `fToggleEdit`, `editable`, `tTime`, and the apparently-dead
`mode`, `mode_temp`, `vaModeCur`, `vaModeList`, `vaModePos`, `vaType` (unused in the source page already).
Also remove the `tInstruction.txt=="time"` / `=="date"` branches from `tmSerial` (nothing left to write
them into).

**Add:** `dirty` (`int32`, local, default `0`), `entToggle` (`string`, local, hidden), `tSubtitle` (`Text`,
visible conditionally), `bToggle` (`Button`/switch-style, visible conditionally).

## Backend changes

Both implementations mirror each other closely; same shape of change in each.

**`apps/nspanel-lovelace-ui/luibackend/pages.py`**
- New branch in the entity-row generator: `elif entityType == "input_datetime" and <time-only check>: entityTypePanel = "datetime"`.
- New `generate_datetime_detail_page(self, entity_id, is_open_detail=False)`, modeled on
  `generate_timer_detail_page` (pages.py:1060): reads `input_datetime.*` state, splits `HH:MM:SS`, resolves
  `toggle_entity`/`subtitle` from the item's config (`entity_config.entity_input_config.get(...)`), sends the
  field layout above.

**`apps/nspanel-lovelace-ui/luibackend/controller.py`**
- `detail_open`: new `if detail_type == "popupDatetime": self._pages_gen.generate_datetime_detail_page(entity_id, True)`.
- `button_press`: new `if button_type == "datetime-set": ... call_service("set_datetime", time=f"{value}:00")`.
- No change needed for the toggle (`OnOff` case at controller.py:255 already entity-agnostic) or for `bExit`
  (controller.py:248, already generic).
- `state_change_callback`: add `if entity.startswith("input_datetime"): self._pages_gen.generate_datetime_detail_page(entity)`
  next to the existing `timer` line (controller.py:187), so live HA-side changes refresh an open popup.

**`nspanel-lovelace-ui/rootfs/usr/bin/mqtt-manager/ha_cards.py`**
- Mirror of the above: new card-row type-panel branch, new `case 'popupDatetime' | 'input_datetime':` in
  `detail_open` (next to the existing `case 'popupTimer' | 'timer':` at ha_cards.py:896), new `datetime-set`
  case wherever `timer-start` is handled, and the card-row generator's equivalent of `pages.py`'s switch.

**Icon mapping** (`icon_mapping.py` / `ha_icons`): add a default icon for the `input_datetime` domain,
same as every other domain already has one.

**Docs**: `docs/entities.md` gets the `toggle_entity` / `subtitle` table rows; `HMI/README.md` gets a new
`popupDatetime` section documenting the message formats above, matching its existing style for other popups.

## Error handling

- Entity missing/unavailable on open: follow the existing sibling pattern exactly — the add-on's
  `detail_open` already guards with `if data: ... else: logging.error(...); return` (ha_cards.py:714); the
  AppDaemon app currently doesn't guard for other popups either, so no new pattern introduced, just
  consistency with what's there today.
- `toggle_entity`/`subtitle` not configured: empty string in the corresponding field → display hides the
  component. No error path needed; this is just the default/common case.
- Popup closed without ever touching `bZ*`: `dirty` stays `0`, no `datetime-set` message is ever sent —
  avoids spuriously rewriting the helper's value just from opening the popup.

## Manual QA checklist

There's no automated test harness in this repo for either backend or the Nextion pages, so verification is
manual, against a real panel or the Nextion Editor simulator:

- [ ] Opening the popup on a configured `input_datetime` shows its current time, hour pre-armed (red).
- [ ] Tapping minute arms it (red) and disarms hour; tapping the already-armed field is a no-op.
- [ ] `bZ1-4 P/M` adjust whichever field is armed; hour clamps at 0/23, minute at 0/59.
- [ ] `t0` and all 8 `bZ*` buttons are visible immediately on open, never hidden.
- [ ] Changing the time then tapping back (`b0`) persists the new value to HA.
- [ ] Changing the time then letting the popup sit until the sleep timeout fires also persists the value
      (not just exits).
- [ ] Opening and exiting without touching `bZ*` does **not** rewrite the `input_datetime` entity.
- [ ] With `toggle_entity` configured: toggle shows current state, tapping it flips the linked entity
      immediately (independent of `dirty`/exit).
- [ ] Without `toggle_entity` configured: no toggle row is shown.
- [ ] With/without `subtitle` configured: line shown/hidden accordingly.
- [ ] A live HA-side change to the `input_datetime` or `toggle_entity` while the popup is open is reflected
      (AppDaemon app's `state_change_callback` path).

## Out of scope (for this change)

- EU and landscape Nextion variants (same design would port, but needs its own Nextion Editor pass and
  QA pass per variant).
- Date-setting (`has_date`) or the newer HA `time` helper domain.
- 12-hour/AM-PM display, despite the US variant's clock elsewhere using it — explicitly rejected in favor
  of 24h for this popup.
