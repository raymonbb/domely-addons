# Changelog

What the update dialog in Home Assistant shows. The repository root's `VERSION` is the single
source of the number and [`../../CHANGELOG.md`](../../CHANGELOG.md) is the whole release note;
this file is the add-on's half of it, in the same [Keep a
Changelog](https://keepachangelog.com/en/1.1.0/) shape. `test.sh` fails the build when the two top
entries and `config.yaml` disagree.

## Unreleased

## [0.1.8] - 2026-09-25

### Added

- A device can be set to "Alleen zien": your housemates see it and cannot switch it, you still can.
- Home can hold small blocks, a quarter of the width, and a block for one room.

## [0.1.7] - 2026-09-24

### Fixed

- A thermostat is refused a mode it does not have before the command reaches Home Assistant.
- Unpairing while the add-on is syncing with Domely Cloud no longer logs an error, and no longer
  leaves the old home's access behind.
- Notifications reach only the phones of people who still have access to the home.

## [0.1.6] - 2026-09-23

### Added

- A house whose grid is one signed P1 number now also gets the notification that it is giving
  power away.

### Changed

- A room you rename in Home Assistant, or a device you move to another one, reaches Domely within
  the second. Until now it waited for the next time Home Assistant restarted, and nothing said so.
- The house tells you which modes a climate device actually has, so a room's card no longer offers
  one it does not, such as "Ontvochtigen" on an air conditioner that cannot dry. A device published
  before this ships keeps offering the full list until you republish it.
- A room or a device can no longer be given a name shaped like a Home Assistant entity id, and
  neither can a room the one-tap setup creates. Nothing already named that way is touched.
- `domely:channel-check` tells an adapter address that does not resolve apart from one that has not
  started yet, instead of one line that reads as a wait either way.

### Fixed

- A lamp or a socket that your energy dashboard happens to name is offered as a lamp or a socket
  when you publish it, instead of as an energy meter.
- When a command cannot reach the part of Domely that talks to Home Assistant, the add-on log says
  what went wrong and which address it tried, instead of one sentence that fits every cause.
- A counter and a power sensor carrying the same energy role, such as a P1's import counter and its
  own import power sensor, are no longer added together in what the house reports for insights.
- The automation engine advances its minute counter only after the work that depends on it, closing
  a narrow window where a failure between two reads in the same tick could fire an automation
  twice.
- Beheer, open in your browser, now hears when you rename a room or move a device in Home
  Assistant, instead of only catching up the next time you reload the page.

## [0.1.5] - 2026-09-23

### Added

- The home screen is yours to compose. The house keeps the layout, everybody in it sees what you
  put there, and each of them only the devices they may reach. It also knows the new blocks: what
  needs attention, what is on, and today.
- A P1 that reports the grid as one number, negative while you give back, can be published as the
  grid. Home Assistant's own grid power sensor is suggested as that, and so is the P1's live power
  when the energy dashboard only knows its counters.
- The house tells the app which of its devices are one physical thing, so an air conditioner's
  display light and jet mode are shown with the air conditioner.

### Changed

- Domely's own logo, in the add-on list and on its page, instead of the placeholder square and the
  word spelled out in pixels.

## [0.1.4] - 2026-09-18

### Added

- This house tells Domely which version of each of its three parts it is running, on the calls it
  already makes. It is what makes "which version are you on" answerable without asking you to go
  and look. Nothing about your house or the people in it travels with it.

## [0.1.3] - 2026-09-18

### Added

- Lamps that can change colour can be asked to. Tick "Kleur veranderen" when you share the lamp, or
  press Aanzetten on the line that says the lamp can do more than it was shared with, and a colour
  picker appears on its card in the app. A lamp that only runs warm to cold white is not offered
  one, because there is no colour in it to pick.

## [0.1.2] - 2026-09-17

### Added

- Domely can ask this house to send a test notification, so somebody can watch one arrive instead
  of waiting for the sun. It goes to every phone and browser in the household that turned
  notifications on, and it does not use up the one notification a day about spare power.

### Changed

- The Domely Cloud address is filled in for you. Leave it as it is unless you run a Domely Cloud
  of your own.

## [0.1.1] - 2026-09-17

### Added

- The sidebar's pairing page shows the code large enough to read from across the room, says how
  long it is still good for, and its button opens the app with the code already in the field.
- The house Home Assistant already knows, offered in one pass: rooms and the devices in them, each
  with a sensible suggestion of what your housemates may do with it, published in one confirmation
  instead of built up by hand.
- "Alle lampen uit" as a quick action on the home screen, which switches off every lamp you shared.
- Automations that run in this house rather than in the cloud. Domely stores them here, and the
  engine that runs them reaches only the devices you published, and only the things those devices
  can really do. What they did is kept here as well and is readable in the app.
- One notification, sent by the house itself and never by Domely Cloud: when your roof has been
  giving power away long enough to be worth doing something about, your phone is told, with
  something to do attached. It reads what Domely has already mirrored, so it costs Home Assistant
  nothing.

## [0.1.0] - 2026-08-31

The first release. Add [raymonbb/domely-addons](https://github.com/raymonbb/domely-addons) as a
repository in the add-on store and Domely appears there.

### Added

- Domely as a Home Assistant add-on: the adapter, the API layer and the Cloudflare tunnel in one
  container under s6-overlay, talking to the Home Assistant it is installed in through the
  supervisor's own proxy. No long-lived token, and no port forwarding.
- A sidebar entry, which opens this home's administrator screens in the Domely app.
- Everything durable under `/data`, so a Home Assistant backup carries the publish configuration
  and a restore brings back the same device identities the household's shares point at.
