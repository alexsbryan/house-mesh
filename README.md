# house-mesh

A house full of laptops, and each one has something worth sharing: a chore
list, a Jellyfin library, a printer queue. The usual answers are a server
someone has to run, or a cloud account that owns the data and wants everyone to
log in.

This is the other answer. Each app stays on the laptop that runs it. The
people in your mesh reach it by name, and it sees who is asking without a login
page, because the daemon already checked. There's no port forwarding, no VPN,
and no deploy step.

The mesh carries requests and proves identity. Your app owns its data and
decides what it means. That split is the whole idea, and the two scripts in
this repository are small enough to read in one sitting:

- `chores.py` — an app that says hello to each housemate by name.
- `library.py` — asks every housemate's Jellyfin the same question and merges
  the answers into one list.

## Set up the mesh

Install `svrn` on each machine. It's a prebuilt binary; nothing is compiled.

```sh
curl -fsSL https://svrnme.sh/install.sh | sh
```

One machine founds the mesh. If the house will share inference too, make it the
one with the GPU; otherwise any always-on box will do.

```sh
svrn setup
svrn mesh create
svrn mesh rotate                    # read the join key out to the room
```

Everyone else joins. If the house shares inference, point at the GPU box and
skip downloading any model files:

```sh
svrn setup --terminal http://<the-gpu-box>:9741
svrn mesh join '<the join key>'
```

Apps and media keep working when the GPU box is down. It's one machine among
several, not the center of the house.

## An app with no login page

From this directory, on one laptop:

```sh
svrn run --as chores -- python3 chores.py
```

That runs the app on a free localhost port and publishes it for as long as it
runs. No config changes, no daemon restart. Ctrl-C stops it and unpublishes
it. The claim renews while the app is alive and expires on its own if it can't
(an hour by default, `--ttl` to change it), so a crash can't leave a dead app
on the house list.

To change the chores, edit `TASKS` and run the command again.

On someone else's machine:

```sh
svrn mesh app <publisher> chores
```

`<publisher>` is the name `svrn mesh status` shows. The command prints a URL,
probes it, and hands it back ready for a browser. The URL is a local port on
your machine that tunnels to the app on theirs. Nothing was copied.

You can't reach your own app this way. Try it with your own name and it says
`'…' is this node — its published app is already local`, whether or not the
app is running. `svrn publish` shows what your machine is publishing; asking
from another machine is the real test.

The app sees the caller as three HTTP headers:

```text
X-Mesh-Member: Alex
X-Mesh-Node:   node-44ae7614
X-Mesh-Pubkey: 8f2c…
```

The caller didn't send these. The daemon verified them during the handshake,
strips any copies the client tried to send, and adds its own. That's how
`chores.py` greets people by name without a users table. Curl it directly on
the machine running it and you get `someone at this workbench` instead,
because the headers are only added on the way through the mesh.

The app serves its own paths: a request for `chores /tasks` arrives as `/tasks`,
not `/chores/tasks`. Use relative URLs for CSS and other assets.

Any HTTP server on localhost that reads those headers is a house app. Nothing
else is required.

## One person, or everyone

```sh
svrn mesh app <person> chores       # one person's app
svrn mesh app fanout chores /       # every member who publishes apps
```

The fanout returns one row per member it asked, with either the answer or the
reason there isn't one:

```text
/chores/ → 2 member(s) asked
  Alex              node-1f2e…    41 ms  200 1204B  application/json
  Dave              node-44ae…     0 ms  failed — connection refused
```

App names aren't gossiped, so a member who publishes something else answers
`404`. That's a row, not a failure. Neither is a machine that couldn't answer —
laptops close. A house app should show partial results rather than pretend the
missing machine had nothing to say.

For a JSON route:

```sh
svrn mesh app fanout chores /tasks --json
```

The mesh doesn't merge the answers for you. Only your app knows whether two
records are the same thing.

Both commands print the URL first and the probe result under it. Read the probe
line: the tunnel answers even when the far side has no such app.

## Media nobody copies

Each person keeps Jellyfin on their own disk. Declare yours once. The command
edits your config in place, keeps its comments, and refuses to write a config
that won't load:

```sh
svrn mesh media origin 8096      # Jellyfin on this machine
svrn daemon restart              # config is read at start
```

Then:

```sh
python3 library.py check                   # is the bridge up? (no API key needed)
python3 library.py list                    # every title, and who holds it
python3 library.py play <person> <item-id>
```

The list is the union of what people already have, and playback streams from
whoever holds the file. If someone's laptop is asleep, everything else still
plays.

Media gets its own command because players like Jellyfin need an unmodified
HTTP origin for absolute asset paths and range requests. Anything that isn't a
media player should use `svrn mesh app fanout <app-name> <path>` instead.

### Your Jellyfin key stays on your machine

Don't hand your Jellyfin API key to everyone in the house. Declare it once on
the machine that runs that Jellyfin:

```sh
printf 'MediaBrowser Token="%s"' "$KEY" | svrn mesh media declare authorization
svrn daemon restart
```

It's read from stdin, stored privately, and added only to requests headed for
your own media server. Nobody who watches from your library ever sees it.

## Publishing for good

`svrn run --as` is for experiments. For an app that should always be
reachable, write a config entry:

```sh
svrn publish chores 5000
svrn daemon restart
```

The entry says where the app answers; it doesn't start the app, so something
still has to be listening on port 5000. `svrn publish` on its own lists what
the daemon is publishing.

Publishing an app doesn't publish your media, and the reverse. They're separate
choices.

## Sharing knowledge

Documents and local AI follow the same rule, as two settings:

- `query_sharing` — may someone ask a question and get a cited answer?
- `mesh_sharing` — may the index itself be copied to their machine?

So you can let housemates search a licensed collection without the files
leaving your disk, or keep a private collection out of the mesh entirely.

## Reaching a machine isn't permission to use it

Whatever carries the packets — home wifi, a radio link between buildings, a
VPN — being able to reach a machine never means being allowed to use its
services. Membership is checked cryptographically before your app sees a byte,
so a stranger who can route to your laptop is still turned away.

To narrow one app to specific people, list member names or node-id prefixes in
`[iroh] app_allow`. Empty, the default, means every member.

## What you give up

- Laptops aren't always on. Keep one machine running for anything that has to
  answer at 4am.
- Names belong to a group, not the world. Two meshes can both have an Alex.
- Nobody merges your records for you. The app author writes that part.
- No one administrator can quietly remove somebody. Membership changes go
  through the group.

## The real test

Media is the easy case. The question is whether the next app works: a chore
rotation, a fridge inventory, a 3D-printer queue that keeps root on its owner's
machine, a house knowledge helper that works with the internet unplugged,
something someone writes at 1am and shares ninety seconds later.

If only Jellyfin works, this is a media feature with a mesh around it. If
someone else writes a small app and it just works, the mesh did its job.

## When something's wrong

`svrn mesh status` shows who is in the mesh. `svrn mesh app` with no arguments
lists other members publishing apps; `svrn mesh media` lists who offers media.
Neither includes you — `svrn publish` is your own view. If someone is missing,
they either haven't published that kind of service or haven't restarted since
changing their config.

A `never_asked` or `failed` row always carries a reason: no matching origin, a
sleeping machine, a refused connection. Show it to the user rather than hiding
it. If the daemon isn't running:

```sh
svrn daemon start
svrn doctor
```

This is pre-release software. The most useful bug report is the smallest step
where it stopped feeling simple.
