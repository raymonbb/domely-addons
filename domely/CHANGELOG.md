# Changelog

What the update dialog in Home Assistant shows. The repository root's `VERSION` is the single
source of the number and [`../../CHANGELOG.md`](../../CHANGELOG.md) is the whole release note;
this file is the add-on's half of it, in the same [Keep a
Changelog](https://keepachangelog.com/en/1.1.0/) shape. `test.sh` fails the build when the two top
entries and `config.yaml` disagree.

## [0.1.0] - unreleased

The first release. Not published yet: no image has been pushed and no tag has been cut, so this
add-on cannot be installed from the store until one is. The date lands here when it is.

### Added

- Domely as a Home Assistant add-on: the adapter, the API layer and the Cloudflare tunnel in one
  container under s6-overlay, talking to the Home Assistant it is installed in through the
  supervisor's own proxy. No long-lived token, and no port forwarding.
- A sidebar entry, which opens this home's administrator screens in the Domely app.
- Everything durable under `/data`, so a Home Assistant backup carries the publish configuration
  and a restore brings back the same device identities the household's shares point at.
