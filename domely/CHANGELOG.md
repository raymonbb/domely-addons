# Changelog

What the update dialog in Home Assistant shows. The repository root's `VERSION` is the single
source of the number and [`../../CHANGELOG.md`](../../CHANGELOG.md) is the whole release note;
this file is the add-on's half of it, in the same [Keep a
Changelog](https://keepachangelog.com/en/1.1.0/) shape. `test.sh` fails the build when the two top
entries and `config.yaml` disagree.

## Unreleased

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
