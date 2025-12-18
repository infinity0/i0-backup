Contents

- [btsync](#btsync): synchronise a file over bittorrent
- [caddr-ln](#caddr-ln): manage a repository of content-addressed storage

Various ad-hoc tools for sysadmin/backup purposes, bit-rotted scripts that I'm
not currently using, but may revive in future.

----

# btsync

Synchronise snapshots of a dynamic file to multiple peers over bittorrent.

The bittorrent protocol is transfer-efficient for *in-place* edits, but not
*insertions* or *deletions*, i.e. iff [Hamming distance][] is short, regardless
of [Levenshtein distance][]. This covers the primary design use case, where the
"file" is a block device that holds a (maybe encrypted) filesystem. Typically,
efficiency is maintained when you expand the filesystem, but not necessarily
when you shrink it. Efficiency may also be lost when/if you defragment the
filesystem, but this is a necessary cost.

On the other hand, if, for your use-case, Hamming distance is typically large
but Levenshtein distance is small (e.g. ABCDEFGH -> ABXCDEFGH), `rsync` is more
transfer-efficient between each sender/receiver pair, but the peer-to-peer
nature of bittorrent may still give a quicker overall transfer time when the
total number of peers is large - so run some tests if performance is an issue.
(One way better than either of these, would be to use the rsync diff algorithm
but have peer-to-peer transfers like bittorrent; but this seems very complex to
achieve.)

[Hamming distance]: http://en.wikipedia.org/wiki/Hamming_distance
[Levenshtein distance]: http://en.wikipedia.org/wiki/Levenshtein_distance

## Pre-use

Depends (origin node): bittornado (>= 4.0), python, ssh, (transmission-create | mktorrent), {peer node dependencies}

Depends (peer nodes): (transmission-daemon, transmission-remote, nc) | (rtorrent, screen, python, xmlrpc2scgi)

Manual patching (origin node only):

- bttrack: use the version from bittornado's CVS
	- `$ cvs -d:pserver:anonymous@cvs.degreez.net/cvsroot co bittornado && cd bittornado`
	- `bittornado$ for i in $THIS_GIT_REPO/patches/bttrack_*.patch; do patch -lp0 < "$i"; done`
	- `bittornado$ mkdir -p ~/bin && sed -e '1s/env python/python/g' bttrack.py > ~/bin/bttrack`

If you plan to create torrents from block devices, you'll need to apply one of
the following too, and rebuild+reinstall as appropriate (origin node only):

- mktorrent: [github fork][blkdev_mktorrent]
- transmission-create: TODO send a patch to them for this

[blkdev_mktorrent]: https://github.com/infinity0/mktorrent/tree/blkdev

Other notes:

- [xmlrpc2scgi.py][] ([DL][xmlrpc2scgi.raw]) is included in this git repo for
  your convenience. `btsync` will automatically copy this to each peer, but you
  can optionally install it into the normal PATH of each peer (inc. origin), if
  you prefer not to keep binaries with your run-time data. You should install it
  with the .py extension; this is because it contains a -p option which is
  python-specific and I didn't want to make assumptions about any potential
  "xmlrpc2scgi" tool that might pop up in the future.

[xmlrpc2scgi.py]: http://libtorrent.rakshasa.no/wiki/UtilsXmlrpc2scgi
[xmlrpc2scgi.raw]: http://libtorrent.rakshasa.no/raw-attachment/wiki/UtilsXmlrpc2scgi/xmlrpc2scgi.py

## Use

Run `btsync init` until it stops complaining at you. It will give you a list of
things to customise; please do so if necessary. If the help text in the default
config files are unclear, you can read below for a more detailed explanation.

After everything is validated, run `btsync dist <file> <snapshot label>` each
time you want to synchronise a snapshot. It is assumed that a label is newer
than another if it appears later in sort order (which must be the same across
all peers). Older labels are expired as appropriate, and re-used as initial
data for a newer torrent.

You only ever run `btsync` on the origin host, although every peer, including
the origin, needs to have a work_dir where btsync puts its run-time files. On
the origin, this is given by the standard "current working directory", or the
env var CWD if set. On the remote peers, this is read from `remotes.txt` in the
origin work_dir. (This file is created automatically when you run `btsync init`
enough times.) You need SSH access to each peer; see `remotes.txt` for details.

You also *must* customise `bttrack.vars` (origin host only) - see the operation
section below for a more detailed explanation on what everything means, if the
hints are unclear.

You also may want to customise `btsync.vars` on each peer. This is typically
optional, but may be necessary for certain setups, e.g. if the firewall settings
don't allow incoming traffic on arbitrary ports, which would prevent the default
"random" port setting from working.

### Operation

Each peer (including the origin node) runs a bittorrent client; the origin node
additionally runs a HTTPS bittorrent tracker. The tracker treats knowledge of a
torrent's info_hash as implicit authorisation to obtain data for that torrent.
To protect the info_hash, each client only presents it to a tracker if its
certificate is signed by your own CA cert, which is distributed out-of-band via
SSH to the client as part of the normal operation of this tool (see below).

See [this helpful article][d-a_openssl] for a walkthrough on how to create your
own CA cert and sign a domain certificate using it. TODO make this a script

[d-a_openssl]: http://www.debian-administration.org/articles/284

The only reason we use SSL is because the bittorrent protocol currently doesn't
support any other signature method for the tracker. Treat the tracker's SSL
certificate like a GPG signature, and the CA cert like a GPG public/private key.
In other words, the CA cert must be controlled by YOU, and you should not sign
certificates for other people since they can use it to masquerade as you, for
the purposes of this tool.

As a corollary, note that using an actual root CA cert completely voids these
security properties, since anyone can get a certificate signed by a root CA for
a price, and can then trick your clients into revealing the info_hash simply by
doing MITM on the client-tracker connection.

NB: the security framework only supports one-CA-cert-per-client, meaning that
all the torrents on that client must use the same tracker. One workaround is to
use a different btsync work_dir for each distinct tracker you need to use. (This
doesn't seem easy to fix, as there is no way to instruct "this tracker must be
signed by this particular CA" on a per-torrent basis, only per-client.)

### HTTPS reverse proxy

So far we've been talking about an HTTPS tracker. In actual fact, this tool
currently uses bttrack which is HTTP only. Therefore you need to also set up an
HTTPS reverse proxy to the bttrack process, which listens on the local interface
only. Make sure the reverse proxy sets the X-Forwarded-For header correctly;
most non-anonymising web servers do this.

See `examples/bttrack.*.conf` on how to configure various web servers to set up
the reverse proxy.

You might also need to tweak the TRACKER_HACK variable in bttrack.vars. Due to
this reverse proxy setup, `--fallback_ip` is likely necessary in most cases.

----

# caddr-ln

Never even bothered writing docs. RTFS.
