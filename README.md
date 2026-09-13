# house-mesh

Any HTTP server running on your laptop can be reached by everyone else in the
house, by name, with the caller's verified identity already attached to the
request. No port forwarded, no VPN, no login page, no deploy step, and no box
that somebody owns and gets blamed for at 11pm.

That is the rule. Media is the example, because everyone in the house already
runs a Jellyfin and the payoff needs no explaining: the house library becomes
the union of what everyone already has, and nothing gets copied anywhere. But
the machinery underneath it does not know what a film is. It carries the chore
rotation, the 3D printer queue, and the thing you write at 1am the same way.

This repo is two files — these instructions and `library.py`, about ninety lines
that merge everyone's Jellyfin catalogue into one list. `library.py` is not the
product. It is a worked example of the one thing the substrate deliberately
will not do for you, and it is short so you can see how little there is to it.

## Setup

One person needs a machine with a GPU. Everyone else downloads no models.

The GPU box, once:

```sh
curl -fsSL https://svrnme.sh/install.sh | sh
svrn setup                      # downloads models, this is the slow one
svrn mesh create                # then `svrn mesh rotate` prints the join link
```

Everyone else, about two minutes:

```sh
curl -fsSL https://svrnme.sh/install.sh | sh
svrn setup --terminal http://<the-gpu-box>:9741    # downloads NOTHING
svrn mesh join '<the join link>'
```

`--terminal` writes no `[models]` section and pulls no model files. It routes
chat and embeddings to the GPU box and otherwise makes you a full member. The
media federation below keeps working when the GPU box is down.

## Publish something

Whatever is listening on a localhost port:

```sh
svrn publish chores 5000        # durable; writes one config entry
svrn publish                    # what am I publishing
svrn unpublish chores
```

For something you are only running right now, skip the config entry entirely:

```sh
svrn run --as chores -- python app.py
```

That publishes for as long as the command runs and leaves nothing behind — no
line to forget, and no `connection refused` row in the house's fan-out six
months from now.

Either way, housemates reach it at `svrn mesh app <you> chores`, and their
requests arrive with the name stripped from the path: your app serves `/tasks`,
not `/chores/tasks`. Use relative URLs for assets, which a one-file Flask app
does anyway.

## What your app is handed

Every request that reaches you carries three headers your daemon verified in
the connection handshake:

```
X-Mesh-Member: LittleMac
X-Mesh-Node:   node-44ae7614
X-Mesh-Pubkey: 8f2c…
```

A client cannot forge them — your daemon strips any copy it was sent and adds
the ones it proved. This is why there is no users table and no login page: the
question "who is asking" is answered before your code runs. Read the header.

Who may reach your apps is `[iroh] app_allow`, empty by default meaning every
member, and it is separate from `media_allow` — publishing a print queue to the
house does not open your film library.

## Media is its own route, for one reason

Jellyfin gets a config line rather than `svrn publish`:

```toml
[iroh]
enabled = true
media_origin = "127.0.0.1:8096"    # Jellyfin's default port
```

Add the line to the `[iroh]` section you already have. Do not append a second
`[iroh]` header — TOML rejects the duplicate and the daemon will not boot. Then:

```sh
svrn daemon restart
svrn mesh media       # the house appears
```

A restart, not a reload: `iroh.media_origin` is one of the fields a running
daemon reports as restart-required, because the acceptor binds its routes at
startup.

The reason media has its own route is that a published app is reached under a
name and gets that name stripped from the path, while a media origin is bridged
whole with no prefix at all. Jellyfin's web UI emits absolute asset paths and
would break behind a prefix; unprefixed, it is Jellyfin's own API unmodified.
The bridge copies bytes and never parses HTTP, so range requests pass through
and seeking works as if the library were local. That is the trade for the
special case: one origin per machine, no name, exact bytes.

`svrn mesh media` lists every member offering a media origin without dialing any
of them. Missing people have not added the line or have not restarted.

## Ask the whole house at once

```sh
svrn mesh media fanout /System/Info/Public
svrn mesh app fanout chores /tasks
```

One request reaches every member publishing that thing, over its own encrypted
connection, and you get one attributed row back per person.
`/System/Info/Public` is Jellyfin's unauthenticated info endpoint, so it answers
before anyone sets up a key — run it first.

Then pick someone:

```sh
svrn mesh media alex           # a localhost URL reaching Alex's Jellyfin
svrn mesh app alex chores      # same, for a published app
```

Open either in a browser or point VLC at the first one.

Then unplug the router's WAN and do all of it again.

## Some machines are off, and that is normal

This is the part that takes longest to get used to, and it is the only genuinely
new habit here. A member who cannot answer comes back as a row saying why, never
as silence — asleep, no such origin, connection refused. Your code renders that
row beside the eleven answers it did get; it does not raise.

Cloud engineering trains you to treat a partial answer as an incident. Here it
is Tuesday, because laptops close.

## The one thing the substrate will not do

Fan-out returns what each member said, by member, and stops. It does not merge,
dedupe, or impose a schema, because merging means knowing what an item IS — and
that is Jellyfin's business, or Plex's, or whatever you run next year. The
system stops one step short on purpose, and the step it left is yours.

`library.py` is that step for the media case: merge by IMDb id, print who has
what. Titles more than one person holds list every holder, so you play from
whoever is awake.

```sh
python3 library.py check      # no auth needed, proves the mesh works
python3 library.py list       # the merged library
python3 library.py play <peer> <item-id>
```

Ninety lines, and about forty of them are the merge. That ratio is the argument.

## Your key stays on your machine

Jellyfin API keys are per-server, so passing them around a house is the wrong
shape: twenty-five people each holding twenty-four other people's credentials.

Declare your own key once, on your own machine. Your daemon adds it to requests
on their way to *your* Jellyfin, on your side of the wire, after the caller has
been admitted as a member. Nobody else holds it. A viewer who sends a token of
their own has it discarded — yours displaces theirs, so exactly one reaches your
server and it is the one you chose.

```sh
# Dashboard -> API Keys in Jellyfin, then:
printf 'MediaBrowser Token="%s"' "$KEY" | svrn mesh media declare authorization
svrn daemon restart
```

`authorization` is the header name because it is the only one Jellyfin 12
answers to. Probed 2026-09-12 against a live 12.0.0: `X-Emby-Token`,
`X-MediaBrowser-Token` and `?api_key=` each returned 401 on `/Items`, only
`Authorization` returned 200, and the OpenAPI document declares that one scheme
and no other. On Jellyfin 10 the spelling is `x-emby-token` with the bare key.
The declaration is a filename either way, so moving between them is a rename on
the machine holding the key, not an upgrade everyone has to take.

The value is read from stdin so it never lands in your shell history. It is
stored 0600 and never printed back; `svrn mesh media declare --list` shows which
headers are set, not what they are. None of it goes in `config.toml`, so the key
never rides along with anything that gets shared, synced, backed up, or sent to
a peer.

## What to build next

This is the part that decides whether any of the above was worth doing. If the
only thing this house ever publishes is Jellyfin, then it was a media feature
with a long README and you should say so.

So: the chore rotation. The fridge inventory. The 3D printer queue that keeps
root on his own printer. The board of what everyone is working on. Thirty lines
of Flask reading one header, `svrn run --as it -- python app.py`, and a
housemate looking at it ninety seconds later.

Everyone here has abandoned a side project at the OAuth step. That step is gone.

## If something is wrong

Nobody in `svrn mesh media` or `svrn mesh app` means they have not published
anything or have not restarted their daemon; `svrn mesh status` shows who is in
the mesh at all, which separates the two. A row saying the member offers no
origin is the same thing seen from the other side. A `failed` row means their
machine is off or their server is not running — normal, and your code should
render it rather than raise. `svrn publish` with no arguments, run on their
machine, is the fastest way for someone to see what they are actually offering.
If `svrn` is not found, the installer puts binaries in `~/.local/bin`; add it to
your `PATH`. Everything else: `svrn doctor`.

---

*Pre-release software. It will have rough edges; tell us which ones.*
