Contents

- [apt-clean](#apt-clean): clean and simplify APT package state
- [bmount](#bmount): mirror subtrees of a filesystem
- [xtsync](#xtsync): sync subtrees between remote hosts
- [git-etc](#git-etc): track/restore system files and metadata
- [rsconf](#rsconf): simple backup framework for rsnapshot
- [luksblk](#luksblk): manage simple LUKS block devices
- [btsync](#btsync): synchronise a file over bittorrent

Various ad-hoc tools for sysadmin/backup purposes.

I'll describe why I wrote my own rather than use existing ones as well :)

----

# apt-clean

Simplify and clean up redundant or incorrect APT package state.

Over the lifetime of a Debian installation, the APT package state can become
cluttered with redundant or "incorrect" (from a human point of view) state. This
can interfere with its dependency resolution algorithms, which makes upgrading
(both manual and automatic) much harder. See issues section for details.

## Pre-use

Depends: aptitude, dialog, git

## Use

Run as root and follow the instructions, which should be self-explanatory. It
will save your answers into `$PWD/apt-clean.txt`, so make sure that is writable.
This is a subset of your APT package state that is "interesting" from a human
perspective. (This is somewhat subjective and based on about 5 years of me
adminstrating various Debian systems; those familiar with the APT object model
are welcome to read the source and file pull requests for improvements.)

The cleanup proceeds in rounds; each round consists of a series of steps, and
the program will execute successive identical rounds until no changes are
detected. Each step represents a type of installation profile (for lack of a
better word), e.g. "automatically installed top-level packages". You are asked
to identify the subset that intentionally belongs to that profile, as opposed
to being an accidental by-product of the issues below. Pay attention to the
instructions on how to achieve this; it encourages you to eliminate redundancy.

The first time you run this program, your APT state is likely to be very untidy
and it will require the most effort to clean up. However, once you figure out
the initial non-redunant clean state file, subsequent runs of this program will
require much less effort to detect new redundancies or untidiness.

Note: if you terminate the program with ctrl-C, your terminal will probably be
a bit screwed. Simply run `reset` (part of ncurses, which is a dependency of
dialog) and it will go back to normal. Even if you can't see those characters,
typing `<ctrl-C> reset <enter>` should hopefully work.

## Issues

Issues with APT/aptitude:

- Imperfect manual maintenance
	- If you are manually trying to resolve dependencies, it's very common to run
	  "aptitude install A B C", where A B C are the dependencies of some target
	  package T. However, this is actually incorrect, because these packages will be
	  marked *manual*, meaning they will remain on the system even if no longer
	  needed. This by itself may not seem like a big deal, but it interferes with
	  future dependency resolution, because the algorithm will try to satisfy the
	  constraints "do not remove A or B or C", which may be impossible and yet
	  unnecessary, hence failing the resolution unnecessarily.
- Subtle aptitude bugs or design flaws
	- The APT object model is very complex, e.g.
		- OR-dependencies and virtual packages means that "reverse depends" is not a
		  well-defined concept (see e.g. bug #594237). Another side effect is that
		  "aptitude remove X; aptitude install X;" in certain cases may not be a no-op,
		  which is misleading and non-intuitive.
		- the ability for users to change at any time whether "automatically installed"
		  should include Depends:, Recommends: or Suggests: relationships means you
		  cannot be sure what "automatic" really means.
	- Because of this, it is not implausible that the dependency resolver algorithm
	  makes different assumptions in different cases, which can result in undesired
	  or (from a higher point of view) "inconsistent" package state.
	- There have been many bugs in APT/aptitude relating to "auto" flags being
	  cleared en-masse or otherwise corrupted. There are too many to list; at the time
	  of writing, a debian bugs search on aptitude for "auto" gave ~50 results, of
	  with ~20 was relevant to this.

Dealing with these issues manually requires a deep understanding of the APT
object model, and even then it's very tedious to remember which search patterns
to use to do the job correctly. This tool aims to automate much of the process,
by giving natural-language descriptions of these issues and instructions on how
to proceed, and also puts your previous answers under version control both to
avoid repetition and data corruption due to APT/aptitude bugs.

## Terminology

In the explanation text, "top-level" refers to an installed package that is not
predependant on / dependant on / recommended by another installed package, and
"absolute top-level" refers to the subset of those that are additionally also
not suggested by another installed package.

----

# bmount

Mirror subtrees between different parts of the filesystem using bind-mounts.

Example use-cases:

- you want to put specific parts of a server config under version control, but
  don't want to blanket-track everything under /etc. (see git-etc below for
  more functionality in this area.)
- you want to store secrets in some secure medium, and temporarily link it to a
  location where it can be used (e.g. ~) but only when you need it.

## Pre-use

Depends: python (>= 2.7), linux (>= 2.6.26)

## Use

TODO

## Related

Two similar tools exist already, but weren't suited to my purposes. However,
they may be more suitable for what you need. (+) means an advantage compared to
bmount; (-) means a disadvantage.

[etckeeper][]:

- (+) "fire and forget" - very little manual config needed to setup initially.
  however, restoring your backup later may not be so easy, due to the spam of
  unnecessary extra information (see next point).
- (-) cannot handle arbitrary subtrees; tracks too many files (all of /etc) by
  default, with no simple way to ignore most of them
	- e.g. I really don't care about /etc/rc*.d, or /etc/ssl/certs, and I don't
	  care about config changes due to package upgrades
	- trivial differences (e.g. different package versions) make it hard to
	  compare systems that are otherwise identical in the *important* areas.
- (-) only one repo that is permanently active and holds everything
- git-etc provides some additional advantages over the git-specific parts of
  etckeeper; see below. bmount+git-etc is my take on etckeeper.

[live-persist][]:

- can handle arbitrary subtrees, like bmount
- (+) is even more flexible in the types of mirror it can handle: bind-mounts,
  symlink trees, and different types of unionfs.
- (-) can only read config from the root of a file system
- (-) can only activate the mirror, not deactivate it. whilst active, cannot
  edit the config (e.g. adding new subtrees) and have it applied automatically
  onto the existing activation.
- intended for use with the "live-boot" system by Debian, so a bit harder to
  install on a normal system

[etckeeper]: http://joeyh.name/code/etckeeper/
[live-persist]: http://live-systems.org/manpages/stable/en/html/persistence.conf.5.html

----

# xtsync

Synchronise subtrees between remote hosts using rsync.

Example use-cases:

- you've got your own personal way of organising many packages and projects in
  a nice tidy filesystem tree under your home directory. you have multiple
  devices, and you'd like to maintain the identical nice tidy structure across
  all of them. however, you don't want to copy the *entire* tree to every
  device, but only the projects that you're working on at the current time.

## Pre-use

Depends: rsync

## Use

TODO

## Related

After I wrote xtsync, I found out about [unison][] which works with the same
idea of a "base tree" and allows the user to select subtrees of that to sync.
The major difference is that it does a *lot* of prompting; sometimes you just
want the tool to shut up and work. You can use this tool and unison on the same
data though, depending on what you prefer at the time.

[unison]: https://www.cis.upenn.edu/~bcpierce/unison/

----

# git-etc

Track and restore system files and metadata in git. Use together with bmount.

More advanced use case compared to etckeeper:

- can put the repo somewhere other than /etc/.git
- does not need to run as root (except when restoring). therefore, can split
  repo by security levels, e.g.:
	- public config in a repo owned by a normal user. you can share this with
	  others for backup/review.
	- private config (e.g. passwords) in a repo owned by root

## Pre-use

Depends: git, liblchown-perl

## Use

TODO

----

# rsconf

A backup framework on top of rsnapshot - which is flexible and powerful, but
because of this, its config files are fairly complex to set up and understand.

Current restrictions - **READ**:

- only backs up to local fs. This includes mounted remote resources (e.g. nfs,
  sshfs), but not non-fs resources even if rsnapshot supports it (e.g. ssh://)
- only backs up files, not e.g. database dumps, but could be easily extended to

Features on top of rsnapshot:

- provides sensible defaults for a large chunk of a typical rsnapshot config.
	- separates backup source config vs. rotation/schedule config. admins can focus
	  on customising the former; developers can refine the latter for better reuse.
	- joins rsnapshot rotation times with crontab schedules, which unfortunately are
	  separated in rsnapshot and annoying to keep consistent.
- utility methods to automate more complex but important backup cases.
	- detects system config files customised by the user, and ignores files that are
	  unchanged from installation.

## Pre-use

Depends: rsnapshot (>= 1.3.1), debsums, dlocate

You may also need to patch rsnapshot:

- if using upstream 1.3.1, you need to apply [r401][], [r406][]
- if using debian 1.3.1-1, you need to apply [r401][], [r406][]
- if using debian 1.3.1-2 or -3, you need to apply [r406][]
- if using debian 1.3.1-4, you can use this with no further patches

[r401]: http://rsnapshot.cvs.sourceforge.net/viewvc/rsnapshot/rsnapshot/rsnapshot-program.pl?r1=1.400&r2=1.401&view=patch
[r406]: http://rsnapshot.cvs.sourceforge.net/viewvc/rsnapshot/rsnapshot/rsnapshot-program.pl?r1=1.405&r2=1.406&view=patch

## Use

Run `rsconf` to print a quick checklist for setting up a backup. It will give
you a list of things to customise; please do so if necessary. If the help text
in the default config files are unclear, you can read below for a more detailed
explanation.

Each backup is stored in a directory `$BACKUP_ROOT` (we'll refer to this as ~)
on the local filesystem. `~/backup.list` defines the source files of the backup,
and it also points to a scheme file which defines when a snapshot is made, how
rotations work, when old snapshots are expired, etc (we'll refer to this as the
rotation/schedule).

Both backup.list and *.scheme are divided up into "chapters" (delimited by `==`)
and "sections" (delimited by `--`). Each chapter defines a named backup target,
stored in ~/[NAME]/. Each backup target effectively has its own rsnapshot config
(which rsconf dynamically generates), including the rotation/schedule config.

The chapters in backup.list MUST match the ones in the scheme it points to.

Scheme files are intended to be distributed together with rsconf to save the
user having to specify their own rotation/schedule (which is quite fiddly) but
you can write your own as well if the ones provided really don't satisfy your
needs. Currently scheme files are found in the /share/ subdirectory of the real
path of `rsconf`, but this could be extended to be more easily customisable.

### backup.list chapters

Each section has a heading (which is the method used to select files) and some
content (usually a list) to apply to the method. See `examples/backup.list` for
a list of all the methods and brief notes on what the content should be.

You can test the output of some content applied to a particular find-method by
sending it to `rsconf test_find_method [METHOD]`. e.g.

	$ echo /etc | rsconf test_find_method all
	backup	/etc	$BACKUP_TARGET/

Note that $BACKUP_TARGET/ will be replaced by the actual backup target.

There is currently no support for adding and using your own methods, but I might
accept a patch for it. However, I would prefer to add new methods to rsconf
itself, rather than encourage users to implement their custom solutions. We are
trying to save work and encourage reuse, after all.

### scheme file chapters

There are two sections, RETAIN and CRONTAB.

RETAIN is what goes into the generated rsnapshot config - see the "retain"
keyword in the rsnapshot manual for more details.

CRONTAB is what goes into your crontab, when you install your backup.list.
$RS_CONF will automatically be set to the correct filename. The rsnapshot manual
tells you how you set up your crontab entries to match your "retain" settings.

See 2tier.scheme for an example.

----

# luksblk

A tool for managing simple LUKS block devices.

Current restrictions - **READ**:

- the LUKS-formatted block device must contain exactly one normal (mountable)
  filesystem, and not some other abstraction like partition tables or LVM

Features on top of cryptsetup:

- provides simple commands for common high-level operations
- keeps metadata together at a pre-defined location so commands mostly take no
  arguments
- additional helper commands for acting on a disk image file

## Pre-use

Depends: cryptsetup

## Use

Run `luksblk` for detailed help text. This should be enough to get you started.

### Editing the crypto settings

If you need to edit the crypto settings, you'll need to re-backup the new LUKS
header afterwards using the `header` subcommand.

### Non-root read access

You probably want non-root read access to the LUKS-formatted block devices
managed by this script - e.g. to use `btsync` to do remote backups. Read access
is only to the encrypted form of the data, so there less potential for a leak.

You can create a single-purpose user for this, but you probably also need to
give it access that persists across reboots. One way to do this is via udev. If
your arrangement is quirky enough that online help isn't easily available, see
`examples/99-local-*.rules` for some not-so-common use cases. In most cases, you
probably want to set `GROUP="<NON_ROOT_USER>" MODE="0640"`.

On a related note, `luksblk` itself tries to avoid using root privileges. It
assumes it has read access to the block device, so you need to set up the above
if you want to run this script not as root. It assumes that it needs root for
write access and various other things such as cryptsetup and device access;
these operations use sudo automatically.

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
