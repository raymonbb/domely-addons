# Domely

Domely gives the people you live with a simple app for the home, without any of them ever seeing
Home Assistant. You decide which entities exist for them, what those are called, and who may reach
which ones. Everything else in Home Assistant stays yours.

## What you need before you start

- **A Domely Cloud account.** Domely is two halves: this add-on at your house, and an account that
  holds who you are, who you shared with, and the address your home is reachable at. Create the
  account first, in the app, at the address you are going to put in `cloud_url` below.
- **Nothing else.** No port forwarding, no reverse proxy, no certificate, no long-lived access
  token. The add-on reaches Home Assistant through the supervisor, with a token the supervisor
  hands it, and reaches the outside world through an outbound tunnel Domely Cloud provisions.

## Installing

1. Settings, Add-ons, Add-on store, then the three-dot menu, Repositories, and add
   `https://github.com/raymonbb/domely`.
2. Install **Domely** from the store.
3. On the Configuration tab, fill in **Domely Cloud address**. The add-on will not start without
   it. Everything else can stay as it is.
4. Start the add-on and open its **Log** tab.

## Pairing

The first time it starts, the add-on asks Domely Cloud for a pairing code and prints it in the log,
along with the address to redeem it at. Open that address, sign in to your Domely account, and type
the code. That is the whole of pairing:

- The Agent gets a token of its own, which it keeps and never shows again.
- Domely Cloud provisions this home's tunnel and hands over its credentials.
- The tunnel comes up, and the app can reach the home from anywhere.

The code expires, and the add-on asks for a new one and prints that instead. An add-on that is
already paired never prints a code again: pairing once is deliberate, and re-pairing is something
you ask for rather than something that can happen by accident.

Open the app after that and it proposes the house: every lamp, switch and thermostat Home Assistant
already has an area for, grouped by that area, ticked and ready to publish in one tap. Nothing is
published until you confirm it, and a lamp Home Assistant placed nowhere is offered too, unticked,
under its own group.

## Options

| Option | What it does |
| --- | --- |
| `cloud_url` | The Domely Cloud you have an account with, including `https://`. Required. |
| `home_name` | What the app calls this home. Empty means Home. |
| `log_level` | `debug`, `info`, `warn` or `error`. `debug` logs every state change Home Assistant sends, which is a lot. |
| `test_light_entity` | A light the add-on switches on and off once at startup to prove it can control Home Assistant. Empty skips the test. |
| `api_threads` | How many app connections this home serves at once. One is held for as long as a phone has the app open, so count the phones and leave room. |
| `cors_origins` | Which addresses may open this home in a browser, comma separated. Empty means the Domely Cloud address above, which is where the app is served from. |

Everything else is fixed by the image on purpose: the paths under `/data`, the addresses the two
halves of the Agent meet on, and the shared secret between them, which is generated the first time
the add-on starts and kept with the rest of your data.

## The sidebar entry, and the sign-in behind it

The **Domely** item in the Home Assistant sidebar opens a small page served by the add-on. That
page is a link, not an administration screen: it takes you to this home's administrator screens in
the Domely app, where you publish entities, name them, put them in rooms and share them.

There is a real trade-off in that, and it is better said than discovered:

- Home Assistant has already authenticated you when you click the sidebar item.
- **The Domely app then asks you to sign in again**, with your Domely account.

That second sign-in is not an oversight. The administrator screens are part of the app that Domely
Cloud serves, and they are the same screens someone reaches from their phone, from outside the
house. Building a second administration UI inside the add-on, with a session of its own, would mean
two implementations of every publish and sharing screen, and a second thing to get authorization
right in. See ADR-0009 and ADR-0010 in the repository for the whole argument.

The sidebar page is reachable only from Home Assistant itself. The add-on refuses it to anything
that did not arrive through the supervisor's ingress proxy, and the tunnel refuses it at the edge,
so it is not reachable from the internet at all.

## The Home Assistant this talks to

**The one it is installed in, and no other.** The supervisor injects a token for its own Home
Assistant and the add-on uses that, which is exactly why you never create a long-lived token here.
There is no option for a different address, and adding one would mean a second credential source
for a single connection.

If what you want is Domely at a house whose Home Assistant runs somewhere else, that is the other
supported way to run it: the **standalone** stack, a small compose file with the same three
processes in three containers, which reaches Home Assistant over your network with a long-lived
token you create. It is not a lesser option or a development mode, and a large part of the Home
Assistant community runs that way. See `deploy/standalone` in the repository.

## Backups

A Home Assistant backup that includes this add-on captures everything Domely keeps: the database
with your rooms, devices, names and capabilities, the pairing identity, and the tunnel credentials.
All of it lives in `/data`, which is the whole of what a per-add-on backup contains.

Just before the backup is taken, the add-on also writes `/data/publish-config.json`: your rooms and
devices in a form you can read, and a different Agent can import. It is written into the backup for
one reason. Every share you have handed out points at a device by an identity that lives only here,
so a restore that lost those identities would leave the household looking at devices that no longer
exist. The export exists so that can be put back by hand if it ever has to be.

## Uninstalling

**Uninstalling the add-on runs nothing.** The supervisor has no uninstall hook, so nothing here
gets the chance to tell Domely Cloud that this home is gone. What is left behind is a home in your
account that will never check in again, the people you shared it with still seeing it, and a tunnel
hostname still pointing at a house that is no longer answering.

So remove the home in the app first:

1. Open the administrator screens in the Domely app, from the sidebar or from your browser.
2. Remove this home. That ends everyone's access, releases the address the home was reachable at,
   and tells the cloud to forget its credentials.
3. Then uninstall the add-on.

Doing it in the other order is recoverable, but only by hand and only from the app.

## Re-pairing, and what it costs today

Removing the home in the app tells Domely Cloud to forget this house. It does not tell the add-on:
the Agent still holds the identity it was handed at pairing, and it will not ask for a new code
until it is made to forget that identity. Forgetting it is one command inside the container today,
and the add-on has no button for it. That is a real gap rather than a design, and it is worth
asking about rather than working around.

What it is not is a reason to uninstall. Uninstalling takes `/data` with it, and `/data` is where
your rooms, devices and above all their identities live, the identities every share you handed out
is written against. If you do uninstall, restore a backup afterwards rather than starting again:
the household gets the same home back instead of a new one that looks like it.

## When something is wrong

**The add-on stops instead of restarting.** That is deliberate, and it means one of the two long
running processes refused to start rather than crashed. The reason is the line above the one saying
the add-on is stopping: a restart loop would scroll it away. Read it, fix what it names, and start
the add-on again. If Home Assistant was itself still starting, starting the add-on again is the
whole fix.

**The log says the Agent is not paired.** Nothing is served publicly until it is. The tunnel raises
nothing, the app finds no home, and the add-on prints the pairing code every time it asks for a new
one.

**The app cannot reach the home from a browser.** The browser is refusing the answer rather than
the home refusing the question. Check `cors_origins`: empty means the address in `cloud_url`, and
if you serve the app from somewhere else, that address has to be named here.

**Nothing at all in the log.** Check the Configuration tab first. The supervisor refuses to start an
add-on whose options do not match its schema, and `cloud_url` is required.

## License

Domely is open source, [Apache-2.0](https://github.com/raymonbb/domely/blob/main/LICENSE). The
add-on you install here, and everything it runs, is code you can read. Domely Cloud, the part that
handles accounts and pairing, is not open source.
