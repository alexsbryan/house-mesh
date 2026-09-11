# house-mesh

Everyone in the house publishes their own media server. Nobody runs a house
server. The library is the union of what everyone already has, and nothing gets
copied anywhere.

This repo is two files: these instructions, and `library.py` — about ninety lines
that merge everyone's catalogue into one list. That is the whole thing. It is
meant to be read in five minutes and rewritten by you.

## Why this instead of one big box

The house server plan means somebody buys disks, somebody has root, somebody gets
woken up when it breaks, and everybody's media gets copied off the drives it
already lives on. That last part is the silly one and you already know it.

Here, your files stay on your machine. When someone plays your copy of something,
it streams off your disk, seeks properly, and no port was forwarded to make that
happen. When your laptop is closed you are simply a row that says "asleep", and
the other eleven answers still arrive.

## Setup

One person needs a machine with a GPU. Everyone else does not download anything.

**The GPU box, once:**

```sh
curl -fsSL https://svrnme.sh/install.sh | sh
svrn setup                      # downloads models, this is the slow one
svrn mesh create                # then `svrn mesh rotate` prints the join link
```

**Everyone else, about two minutes:**

```sh
curl -fsSL https://svrnme.sh/install.sh | sh
svrn setup --terminal http://<the-gpu-box>:9741    # downloads NOTHING
svrn mesh join '<the join link>'
```

`--terminal` writes no `[models]` section and pulls no model files. It routes
chat and embeddings to the GPU box and otherwise makes you a full member.

**Then publish your media server.** Open `~/.svrnmesh/config.toml`, find the
`[iroh]` section, and add one line to it:

```toml
[iroh]
enabled = true
media_origin = "127.0.0.1:8096"    # Jellyfin's default port
```

Do not paste a second `[iroh]` header — most configs already have one and TOML
will reject the duplicate. If yours has no `[iroh]` section at all, add the whole
block.

```sh
svrn daemon reload    # applies config without dropping anything
svrn mesh media       # the house appears
```

That last command lists every member offering a media origin, from gossip alone
— nothing is dialed. If people are missing, they have not added the line or have
not reloaded.

## The moment

```sh
svrn mesh media fanout /System/Info/Public
```

One request, sent to every member's media server through its own encrypted
bridge, one row back per person. `/System/Info/Public` is Jellyfin's
unauthenticated info endpoint, so this works before anyone sets up API keys —
which makes it the right thing to run first.

Then pick someone:

```sh
svrn mesh media alex
```

That prints a `127.0.0.1:<port>` URL that reaches Alex's Jellyfin. Open it in a
browser or point VLC at it. It is Jellyfin's own API, unmodified — the bridge
copies bytes and never parses HTTP, so range requests pass through and seeking
works exactly as if the library were local.

Now unplug the router's WAN and do all of it again. That is the part worth
watching someone's face for.

## The library

`library.py` merges everyone's catalogue by IMDb id and prints who has what.
Titles held by more than one person list every holder, so you play from whoever
is awake.

```sh
python3 library.py check      # no auth needed, proves the mesh works
python3 library.py list       # the merged library
python3 library.py play <peer> <item-id>
```

### The authentication bit, honestly

Jellyfin's `/Items` needs an API key, and keys are per-server — so one header
cannot authenticate you to twenty-five different Jellyfins. `library.py` handles
this by asking each person's server separately with that person's key, read from
`~/.house-mesh/tokens.json`:

```json
{ "alex": "abc123...", "sam": "def456..." }
```

Each person generates one in Jellyfin under Dashboard → API Keys and shares it
with the house. It works, and it is the annoying part of this repo.

**The better answer, if someone wants a real project:** your daemon already hands
your media server the caller's verified identity as an `X-Mesh-Member` header,
with any forged one stripped first. A twenty-line proxy sitting between the
bridge and your Jellyfin could read that header and add *your own* token, so
nobody ever shares a key with anybody. That is a genuinely good afternoon and it
would delete this whole section.

## What to build next

The media thing is the demo, not the point. The same machinery carries anything
that speaks HTTP on localhost, and every request arrives with `X-Mesh-Member`
already verified — so nothing you write here needs a login page, a users table,
or a deploy step.

The chore rotation. The fridge. The 3D printer queue. The thing you write at 1am
and want to show someone before you go to bed.

## If something is wrong

- **Nobody appears in `svrn mesh media`** — they have not set `media_origin`, or
  have not run `svrn daemon reload`. `svrn mesh status` shows who is in the mesh
  at all, which separates the two.
- **A row says the member offers no media origin** — same thing, from the other
  side.
- **A row is `failed`** — their machine is off or their Jellyfin is not running.
  This is normal and your code should render it, not raise.
- **`svrn` not found** — the installer puts binaries in `~/.local/bin`; add it to
  your `PATH`.
- **Everything else** — `svrn doctor`.

---

*Pre-release software. It will have rough edges; tell us which ones.*
