# mesh

An SSH overlay network.

Stock OpenSSH is the transport. Mesh produces and manages its configuration via
`~/.ssh/config` and `~/.ssh/authorized_keys`.

It attempts to be portable, running under Linux, Mac, Android/Termux, and
similar environments.

## Model

**Members** are your devices. Each holds one device keypair plus its sshd
host key. The device key is what mesh runs on: it resolves, gossips and
authenticates machine to machine, and it never opens a shell anywhere.

**Operators** are the keys that do open shells. They live in `conf/operators`,
are authorized on every member, and are meant to sit in an ssh-agent rather
than on a disk. A mesh has at least one from the moment it is created.

**Guests** are dial-only: named, host-key-pinned and routable, but they hold no
mesh key, run no mesh, and are never contacted in the background. Authentication
is whatever your agent offers.

**Forward aliases** come from a guest's `fwd=` lines and generate a `Host
<guest>-<suffix>` carrying a `LocalForward`, reaching a port that only the guest
can see.

State is split in two. `~/.config/mesh/conf` is a git repo of public material
only (node metadata, device public keys, host public keys, the operator list)
and is gossiped between peers. `~/.config/mesh/state` is private, never leaves
the device, and holds the device key, the roster and the generated
`known_hosts`.

## Example

The rest of this document uses one example topology to explain the functionality
of `mesh`:

    desk     desktop, always on, at home
    book     laptop, moves between home, the office and cafes
    phone    Android under Termux
    web01    a client's web server, not yours, holds no mesh key
    ci       build box, launched per job, different address every time

`desk`, `book` and `phone` are members. `web01` and `ci` are guests.

## Getting started

Start with the key that will open your shells. Keep the private half in an
agent; mesh only ever needs the public half:

    ssh-keygen -t ed25519 -C me -f ~/.ssh/op
    ssh-add ~/.ssh/op

`init` and `join` name the device they are run on. `mesh init desk` means "this
machine is `desk`", and `mesh join book ...` means "join the mesh as `book`",
not "join something called book".

On `desk`, the first device:

    mesh init desk --operator ~/.ssh/op.pub
    mesh schedule

Adding `book` takes one command on each side. On `desk`:

    mesh invite --serve

That prints a one-use token and an address, and serves the conf repo for 30
minutes. On `book`, naming itself and pointing at the address `desk` printed:

    mesh join book 192.168.1.10:43117 --token <token>

The token is single use, the joiner's proof is bound to its own device key, and
both sides pin the other's host key during the exchange. Neither machine needs
any prior credential for the other, and no `--operator` is given, because conf
already carries the operator list. Repeat for `phone`.

`ssh desk`, `ssh book` and `ssh phone` now work from any of them, authenticated
by the operator key in your agent.

A member with a fixed address can say so, which makes it resolve in one step
instead of by search:

    mesh init desk --static 203.0.113.10

## Guests

Guests work the other way around: you declare them from your own device, and
nothing is ever installed on or run against them. `web01` belongs to a client,
holds no mesh key, and is never contacted in the background. On any member:

    mesh guest web01 198.51.100.7 --user deploy
    ssh web01

Authentication is whatever your agent offers, so no key for the client needs to
sit on your disk, and nothing of yours is left on their machine. For a machine
you do own but do not want to enrol, authorize your operator public key there
once and it behaves the same way.

Its database listens on localhost only. A forward alias reaches it:

    mesh guest web01 198.51.100.7 --user deploy --fwd db:15432:127.0.0.1:5432
    ssh web01-db

That holds local port 15432 open to `127.0.0.1:5432` on `web01` for as long as
it runs.

`ci` has a new address on every launch, so it is declared without one:

    mesh guest ci --user root --dynamic

Each device that should reach it supplies an executable named `mesh-resolve-ci`
on `PATH`, printing the address on line one and the host public key on line two.
Whichever device resolves it publishes a signed record, so the others reach `ci`
without holding the cloud tooling.

## Resolution

`ssh <peer>` runs `mesh dial <peer>` as its `ProxyCommand`, which walks a chain
until something answers:

1. the cached roster entry
2. a `static=` address from the node's conf
3. the endpoints in the node's signed, gossiped record
4. a relay through another member that can currently reach it
5. an nmap sweep of the local subnets

Every candidate must accept a connection and present a host key pinned to that
node, so the wrong host at the right address is rejected.

Step 4 covers two machines on one network that cannot reach each other, more
common than it should be. If `book` and `desk` are both on the house wifi but
blind to each other while `phone` sees both, the roster records `jump phone
<phone addr> <desk addr>` and dials run `ssh -W` through `phone`. Nothing is
provisioned in advance, and a direct route replaces it as soon as one works.

Two nodes that share no reachable peer and have no fixed address stay
unreachable. Mesh has no rendezvous server and parks no tunnels.

## Access control

No key on any disk opens a shell. Every device key in every member's
`authorized_keys` carries
`restrict,port-forwarding,command="<mesh> serve"`, and `serve` accepts four
verbs: `recs`, `probe`, and `git-upload-pack`/`git-receive-pack` scoped to the
conf repo. A stolen device key moves public conf and nothing else. It cannot
open a shell, request sftp, or point git at another repository.

Shells come from the keys in `conf/operators`, which are authorized on every
member unfiltered. Changing that file requires a signature from a key already in
it, so a stolen device key cannot mint itself a shell:

    mesh operator alice ~/alice.pub --as me

The device key stays on disk, because the daemon has to resolve, gossip and
converge with nobody present. What changes is what possession of it is worth.

The conf repo enforces its own ACL in a `pre-receive` hook, and the rule is
default-deny: conf holds `nodes/`, `keys/`, `hostkeys/` and `operators`, and a
push touching anything else is rejected. Node `n` is the only writer of
`nodes/n.conf`, `keys/n.pub` and `hostkeys/n.pub`, and pushes must be signed by
the key they claim to speak for. Introducing a new node is the one exception,
since it has no key in conf yet.

The one attack that survives, a process running as you appending its own key to
`authorized_keys`, is watched instead: every tick hashes the file outside the
managed block and raises a warning (and a Termux notification) when it changes.
Accept a deliberate change by removing `state/ak.base` after inspecting.

Losing every operator private key means console access on one machine and
`mesh operator` from there. Mesh refuses to remove the last one.

## Scheduling

`mesh schedule` installs the platform's supervisor: Termux:Boot plus a
job-scheduler backstop on Android, a systemd user unit with lingering on Linux,
or cron. The daemon is event-driven rather than polled: it reacts to address
changes, membership changes and unresolved peers, with a long backstop. Poll
intervals follow a profile chosen from whether the machine roams or stays put.

## Commands

    init <name> --operator <pubkey>
                         name this device, create its identity and state
    invite [--serve]     mint a one-use token and serve conf to a joiner
    join <name> <addr>   join an existing mesh as <name>, using a token
    adopt <user@host>    pull a node in from this side, or --scan the subnet
    guest <name> [addr]  declare a dial-only node, from your side only
    operator             show, record or remove a shell key, operator-signed
    tick                 resolve peers, sync conf, regenerate config
    status               node, members, guests, forwards, operators, drift
    schedule             install the platform supervisor
    daemon [--once]      run the convergence loop
    reset [--all]        remove managed blocks and scheduling

Five more exist and are not for you to type: `dial` is the `ProxyCommand`,
`serve` is what sshd runs for an arriving device key, `confcheck` and `compile`
are the conf repo's git hooks, and `sync` is the gossip step.

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

Not a secret store and not an access-request system. It authorizes keys you
already decided to trust.

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
