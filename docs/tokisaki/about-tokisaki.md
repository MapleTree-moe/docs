---
sidebar_position: 1
---

# About Tokisaki

Tokisaki is a backup system based on the **[Kopia][1]** open source project.
Kopia can be difficult to get started with entirely from scratch, so Tokisaki
abstracts most of the complexity away by creating a [repository server][2],
which means the end user only needs to know Tokisaki's access URL, their
username, and their password.

## Architecture
<sup>This image is best viewed in dark mode.</sup>
![Docs Version Dropdown](./img/tokisaki-architecture.svg)

The final storage location of all data is in a Backblaze B2 bucket hosted in
the european datacenter.

Clients are able to read and write content ids, that is data "blobs" that are
part of a snapshot manifest that their user account allows them access to. The
clients are unaware of any content ids that are not explicitly bound to a
snapshot manifest that they have access writes to.

For purposes of maintenance, the server is aware of all content ids and capable
of rewriting/relinking content ids into smaller packs, or recognizing when
content ids are orphaned, but is not able to access snapshot manifests in such
a way as to allow reconstruction of the data.

Imagine every 4MB of data is a puzzle piece with a number. Your client knows
how to assemble your puzzle pieces in the correct order, but can't see any
other puzzle pieces. Even if someone else has a file with an identical puzzle
piece, you can't see their map, so you don't know they share a puzzle piece
with you.

The server is able to see all the puzzle pieces and know if someone is using
them, but doesn't have the map to put them all back together. It does this by
referring to content ids and pack ids.

Backblaze has all the puzzle pieces and all the maps, but they're all encrypted
in such a way as to be essentially random data should backblaze attempt to look
at them.

As an example, say you have a snapshot made up of content ids
`[0001, 0002, 0003, 0004]`, which are stored in pack 1. Someone else uploads a
snapshot with `[0002,0005,0006,0004,0007]` stored in pack 2. The server sees
two identical content ids in two different packs, and rewrites 0002 and 0004
into pack 3, then updates the content id registry to point to the new pack.

The server isn't aware of the order of content ids needed to make a file, only
that there are two references in the content id listing to the same bit of
data. Likewise, even though you and another user share the same 4mb of data,
neither of you has enough information to access the other content ids to make
the opposing users file.

## Repository Configuration

```
Storage type:        s3
Storage capacity:    unbounded
Storage config:      [REDACTED]

Hash:                BLAKE2B-256-128
Encryption:          AES256-GCM-HMAC-SHA256
Splitter:            DYNAMIC-4M-BUZHASH
Format version:      2
Content compression: true
Password changes:    true
Max pack length:     21 MB
Index Format:        v2

Epoch Manager:           enabled
Epoch refresh frequency: 20m0s
Epoch advance on:        20 blobs or 10.5 MB, minimum 24h0m0s
Epoch cleanup margin:    4h0m0s
Epoch checkpoint every:  7 epochs
```

The repository is additionally configured with a 5% Reed-Solomon ECC Parity
overhead, in addition to the 17+3 sharding used by Backblaze B2. To put it in
simple terms, from a 21MB file, backblaze would need to suffer three rack
failures, and an additional 1.05MB of the remaining 17 shards would have to be
corrupted for a single 21MB pack to become lost. Even then, the rest of the
files in the snapshot would be unaffected, and any user with that content id
stored locally would automatically re-upload it on their next snapshot.

## Pricing information
Tokisaki is hosted on a [Hetzner CAX-21][3] ARM Ampere server, which is $7.59
per month. The storage is hosted on [Backblaze B2][4] which charges $6/TB per
month, in addition to Class A and C API calls billed at $0.0004/1,000
calls/day and $0.004/1,000 calls/day respectively. Ko-Fi charges between $0.50
and $1.00 per membership, in addition to any fees charged by Paypal.

Users are only charged for their data, and are expected to be honest about
their data usage, as due to the system design there is **no way** to verify how
much data a user has uploaded. This does mean some users buying more data than
they use could subsidized some users buying less data than they use. In the
event a discrepancy comes to light, with the permission of a simple majority
of users, a special user can be created on the server to allow an
administrator to list the size of all snapshots on the system and who they
belong to.

Reports of current storage usage will be made available to any user of Tokisaki
upon request, as well as screenshots of the Hetzner cloud platform page for the
Tokisaki server to verify information listed here.

Users can verify current ACLs on the server through the use of the
`kopia server acl list` command, to verify only they have access to their
snapshots.

[1]: https://kopia.io
[2]: https://kopia.io/docs/repository-server/
[3]: https://www.hetzner.com/cloud/
[4]: https://www.backblaze.com/cloud-storage/pricing
