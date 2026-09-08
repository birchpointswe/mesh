# mesh

An SSH overlay network.

Stock OpenSSH is the transport. Mesh produces and manages its configuration via
`~/.ssh/config` and `~/.ssh/authorized_keys`.

It attempts to be portable, running under Linux, Mac, Android/Termux, and
similar environments.

## Model

**Members** are your devices. Each holds one device keypair plus its sshd
host key. A member can open a shell on any other member, subject to that
member's own allow list.

**Guests** are dial-only: named, host-key-pinned and routable, but they hold no
mesh key, run no mesh, and are never contacted in the background. You reach them
with an agent, or with the device key if you flag them.

**Boards** are members with a static address. Any member without one parks a
persistent reverse tunnel on every board and advertises it, so a device that can
never accept an inbound connection is still reachable by dialling the board.

**Forward aliases** come from a guest's `fwd=` lines and generate a `Host
<guest>-<suffix>` carrying a `LocalForward`, reaching a port that only the guest
can see.

State is split in two. `~/.config/mesh/conf` is a git repo of public material
only (node metadata, device public keys, host public keys, the admin list,
signed group assignments) and is gossiped between peers. `~/.config/mesh/state`
is private, never leaves the device, and holds the device key, the roster and
the generated `known_hosts`.

## Example

The rest of this document uses one example topology to explain the functionality
of `mesh`:

    desk     desktop, always on, at home
    book     laptop, moves between home, the office and cafes
    phone    Android under Termux
    vps      cloud instance with a static address
    web01    a client's web server, not yours, holds no mesh key
    ci       build box, launched per job, different address every time

`desk`, `book`, `phone` and `vps` are members; `vps` is also the board, since a
board is just a member with a static address. `web01` and `ci` are guests.

## Getting started

`init` and `join` name the device they are run on. `mesh init desk` means "this
machine is `desk`", and `mesh join book ...` means "join the mesh as `book`",
not "join something called book".

On `desk`, the first device:

    mesh init desk
    mesh schedule

Adding `book` takes one command on each side. On `desk`:

    mesh invite --serve

That prints a one-use token and an address, and serves the conf repo for 30
minutes. On `book`, naming itself and pointing at the address `desk` printed:

    mesh join book 192.168.1.10:43117 --token <token>

The token is single use, the joiner's proof is bound to its own device key, and
both sides pin the other's host key during the exchange. Neither machine needs
any prior credential for the other. Repeat for `phone`.

On `vps`, which has a fixed address:

    mesh init vps --static 203.0.113.10

That flag is what makes it a board. From their next tick, `book` and `phone`
park reverse tunnels on it without being told to, so they stay reachable from
anywhere, including from each other over cellular.

`ssh desk`, `ssh book`, `ssh phone` and `ssh vps` now work from any of them.

## Guests

Guests work the other way around: you declare them from your own device, and
nothing is ever installed on or run against them. `web01` belongs to a client,
holds no mesh key, and is never contacted in the background. On any member:

    mesh guest web01 198.51.100.7 --user deploy
    ssh web01

Authentication is whatever your agent offers, so no key for the client needs to
sit on your disk, and nothing of yours is left on their machine.

For a machine you do own, `--identity` authenticates with the device key each
member already has, instead of the agent:

    mesh guest ci --user root --dynamic --identity

It mints nothing. Authorize the members once by appending every
`conf/keys/*.pub` to the target's `authorized_keys`, and from then on any member
can add or revoke a device there over ssh. Leave the flag off for anything you
do not own: agent auth is the default for a reason.

Its database listens on localhost only. A forward alias reaches it:

    mesh guest web01 198.51.100.7 --user deploy --fwd db:15432:127.0.0.1:5432
    ssh web01-db

That holds local port 15432 open to `127.0.0.1:5432` on `web01` for as long as
it runs.

`ci` has a new address on every launch, so it carries a resolver instead of an
address:

    mesh guest ci --user root --resolve 'cloud-tool address ci'

Alternatively put an executable named `mesh-resolve-ci` on `PATH`. Either way it
prints the address on line one and the host public key on line two.

## Resolution

`ssh <peer>` runs `mesh dial <peer>` as its `ProxyCommand`, which walks a chain
until something answers:

1. the cached roster entry
2. a `static=` address from the node's conf
3. the endpoints in the node's signed, gossiped record
4. a reverse tunnel the peer parked on a board
5. a relay through another member that can currently reach it
6. an nmap sweep of the local subnets

Every candidate must accept a connection and present a host key pinned to that
node, so the wrong host at the right address is rejected.

Steps 4 and 5 look similar and solve different problems. A board is a standing
rendezvous, provisioned in advance, for peers that can never accept an inbound
connection. A relay needs nothing beforehand and uses whichever member happens
to see both ends, which covers two machines on one network that cannot reach
each other, more common than it should be. If `book` and `desk` are both on the
house wifi but blind to each other while `phone` sees both, the roster records
`jump phone <phone addr> <desk addr>` and dials run `ssh -W` through `phone`. A
direct route replaces it as soon as one works.

Whichever device resolves a guest publishes a signed record of it, so `phone`
reaches `ci` without holding the cloud tooling that `book` used to find it.

## Access control

The conf repo enforces its own ACL in a `pre-receive` hook. Node `n` is the only
writer of `nodes/n.conf`, `keys/n.pub` and `hostkeys/n.pub`, and pushes are
rejected unless signed by the key they claim to speak for. Adding a new node is
the one exception, and changing the admin list requires a signature from a
current admin.

Groups are assigned in `conf/groups`, signed by an admin and carrying a
monotonic serial so an old copy cannot be replayed. Verification is fail-closed.
Each node keeps its own `allow` list locally; `mesh compile` writes an
`authorized_keys` holding only the members whose groups intersect it, so sshd
enforces the policy. The admin key can live in an agent.

Putting `desk` and `book` in one group and `phone` in another, then restricting
what `desk` accepts:

    mesh group desk trusted
    mesh group book trusted
    mesh group phone mobile

    mesh allow trusted        (run on desk)

`book` still opens a shell on `desk`. `phone` no longer can, because its key is
absent from `desk`'s `authorized_keys`. `phone` keeps its own inbound policy;
`allow` is local to the node that runs it.

## Hardening

By default a member's device key opens a shell on every other member, so a copy
of that file is a copy of your access. `mesh restrict` changes that for the node
it runs on: member keys arriving there are forced through `mesh serve`, which
speaks only gossip verbs, scoped `git-upload-pack` and `git-receive-pack` for
the conf repo plus `recs`, `drop` and `probe`. A stolen key then moves public
conf and nothing else. It cannot open a shell, request sftp, or point git at
another repository.

Shells come instead from an operator key listed in `conf/operators`. `compile`
authorises those on every member without group or `allow` filtering, which also
makes them the way back in when a policy change locks you out. Recording one
requires an admin signature, so a stolen device key cannot mint itself a shell.
Keep the private half in an agent and never on disk.

    mesh operator me ~/op.pub --as me
    mesh restrict

Peers learn the node is restricted through ordinary conf gossip and switch to
verbs on their next tick; their stanzas for it stop offering the device key at
all. Nothing coordinates this, and each node changes only its own inbound door,
so you can convert one machine at a time.

The device key stays on disk, because the daemon has to resolve, gossip and back
up with nobody present. What changes is what possession of it is worth.

The one attack that survives, a process running as you appending its own key to
`authorized_keys`, is watched instead: every tick hashes the file outside the
managed block and raises a warning (and a Termux notification) when it changes.
Accept a deliberate change by removing `state/ak.base` after inspecting.

## Scheduling

`mesh schedule` installs the platform's supervisor: Termux:Boot plus a
job-scheduler backstop on Android, a systemd user unit with lingering on Linux,
or cron. The daemon is event-driven rather than polled: it reacts to address
changes, membership changes and unresolved peers, with a long backstop. Poll
intervals follow a profile chosen from whether the machine roams or stays put.

## Commands

    init <name>          name this device, create its identity and state
    invite [--serve]     mint a one-use token and serve conf to a joiner
    join <name> <addr>   join an existing mesh as <name>, using a token
    adopt <user@host>    pull a node in from this side, or --scan the subnet
    guest <name> [addr]  declare a dial-only node, from your side only
    tick                 resolve peers, sync conf, regenerate config
    sync                 gossip the conf repo with reachable peers
    publish              rebuild and sign this node's endpoint record
    compile              regenerate authorized_keys, known_hosts, ssh_config
    dial <peer>          transport pipe, used as ProxyCommand
    status               node, peers, tunnels, managed blocks
    show                 full view including guests and forward aliases
    roster               print resolved peers
    admin / group        manage admins and signed group assignments
    operator             record or remove a login key, admin-signed
    restrict [--off]     force arriving member keys through 'mesh serve'
    serve                the restricted endpoint, run by sshd, not by you
    allow / groups       set local ingress policy and self-declared groups
    schedule             install the platform supervisor
    daemon [--once]      run the convergence loop
    reset [--all]        remove managed blocks and scheduling

Unknown subcommands dispatch to `mesh-<command>` on `PATH`, git style, so
extensions plug in without editing the script. `mesh backup` runs `mesh-backup`
with `MESH_NODE`, `MESH_ROOT`, `MESH_CONF` and `MESH_STATE` exported, so an
extension knows what this node is called and where its conf and state live
rather than hardcoding paths. Calling `mesh-backup` directly still works; it
just does not get told any of that.

## Requirements

`bash`, `git`, OpenSSH client and server, `ssh-keygen`, and `flock`. `nmap` is
optional and only used for the subnet sweep. On Android, Termux with
`openssh`, and Termux:Boot plus Termux:API if you want it to start at boot.
macOS does not ship `flock`; install one (Homebrew has it) before use.

Both ends of a join need a reachable sshd. mesh does not open ports for you.

## Termux note

Termux has no `/usr/bin/env`, and the shebang is only rewritten inside login
shells, so a `#!/usr/bin/env bash` script fails with `bad interpreter` under the
job scheduler, Termux:Boot, or an ssh `ProxyCommand`. Install with an absolute
interpreter:

    { printf '#!%s/bin/bash\n' "$PREFIX"; tail -n +2 mesh; } > ~/bin/mesh
    chmod 755 ~/bin/mesh

## What this is not

Not a VPN. It routes no traffic and gives you nothing ssh does not; it manages
identity, discovery and configuration so ssh keeps working as machines move.

conf holds public material only. Private keys never leave the device that
generated them.

Two nodes that cannot reach each other and share no reachable peer stay
unreachable. There is no hosted relay.

## License

MIT License

Copyright (c) 2026 Birch Point SWE

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
