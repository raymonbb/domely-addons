# Domely

Share a simple, premium app for your home with the people who live in it, without any of them ever
seeing Home Assistant.

You publish the lights, switches and meters you want shared, give them names the household
recognises, and put them in rooms. Everyone else gets an app: their phone, their own login, only
the devices you shared with them. No dashboards, no entity IDs, no YAML, and nothing to explain.

Away from home works the same as at home. The add-on opens an outbound tunnel to Domely Cloud, so
there is no port to forward, no certificate to renew and nothing of Home Assistant on the public
internet. Home Assistant stays the source of truth: Domely never writes configuration into it, and
it only ever touches what you published.

**You need a Domely Cloud account.** The add-on prints a pairing code in its log the first time it
starts, you redeem it in the app, and that is the whole of setup. See the Documentation tab.

**Nothing has been published yet.** No image has been pushed and no version has been tagged, so
this add-on cannot be installed from the store until the first release is cut.

## In this repository

The add-on is built from the repository root, not from this folder: it assembles the same three
components [`../standalone`](../standalone/README.md) ships as three containers, and they live
above this directory.

```sh
docker buildx build -f deploy/addon/Dockerfile .
```

That is why `config.yaml` carries an `image:` key. The supervisor builds a local add-on with the
add-on folder as its context, which could not work here, so this one is installed from the
published image, which is how add-ons are meant to ship anyway.

| File | What it is |
| --- | --- |
| `config.yaml` | The manifest: what the store shows, what the operator is asked for, what the container may do |
| `Dockerfile` | The three components on the Home Assistant base image, built from the root context |
| `rootfs/` | Copied into the image with one `COPY`, so a path here is the path in the container |
| `apparmor.txt` | The profile the supervisor loads, named for the slug |
| `translations/en.yaml` | The name and help text of every option |
| `test.sh` | What `make test` runs over all of the above |
| `generate-images.mjs` | Draws `icon.png` and `logo.png`, committed beside it |

`test.sh` is the only thing here that can be checked without a Home Assistant to install into, so
it checks the mistakes that would otherwise be found on somebody's hardware: a manifest key the
supervisor needs, a version that disagrees with `VERSION`, an option with no help text, an s6
service that is not in the bundle, a durable path that stopped being under `/data`.

The add-on has **not** been installed on a real HAOS. The image builds for both architectures and
the manifest is checked mechanically, and that is a different claim.
