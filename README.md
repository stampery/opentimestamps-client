# OpenTimestamps Client

Command-line tool to create and validate timestamp proofs with the
OpenTimestamps protocol, using the Bitcoin blockchain as a timestamp notary.
Additionally this package provides timestamping of PGP signed Git commits, and
verification of timestamps for both Git commits as a whole, and individual
files within a Git repository.


## Requirements and Installation

* Python3 >= 3.4.2
* python-bitcoinlib >= 0.6.1
* GitPython >= 2.0.8 (optional, required only for Git commit rehashing support)

Additionally while OpenTimestamps can *create* timestamps without a local
Bitcoin node, to *verify* timestamps you need a local Bitcoin Core node (a
pruned node is fine). You also need to set the `rpcuser` and `rpcpassword`
options in `~/.bitcoin/bitcoin.conf` to allow the OpenTimestamps client to
connect to your node via the RPC interface.

The two required libraries are available via PyPI, and can be installed with:

    pip3 install -r requirements.txt

Once those libraries are installed, you can run the utilities directory out of
the repository; there's no system-wide installation process yet.


## Usage

Creating a timestamp:

    $ ./ots stamp README.md
    INFO:root:Submitting to remote calendar https://a.pool.opentimestamps.org
    INFO:root:Submitting to remote calendar https://b.pool.opentimestamps.org

You'll see that `README.md.ots` has been created with the aid of two remote
calendars. We can't verify it immediately however:

    $ ./ots verify README.md.ots
    INFO:root:Assuming target filename is 'README.md'
    INFO:root:Calendar https://alice.btc.calendar.opentimestamps.org: No timestamp found
    INFO:root:Calendar https://bob.btc.calendar.opentimestamps.org: No timestamp found

It takes a few hours for the timestamp to get confirmed by the Bitcoin
blockchain; we're not doing one transaction per timestamp.

However, the client does come with a number of example timestamps which you can
try verifying immediately. Here's a complete timestamp that can be verified
locally:

    $ ./ots verify examples/hello-world.txt.ots
    INFO:root:Assuming target filename is 'examples/hello-world.txt'
    INFO:root:Success! Bitcoin attests data existed as of Thu May 28 15:41:18 2015 UTC

Incomplete timestamps are ones that require the assistance of a remote calendar
to verify; the calendar provides the path to the Bitcoin block header:

    $ ./ots verify examples/incomplete.txt.ots
    INFO:root:Assuming target filename is 'examples/incomplete.txt'
    INFO:root:Got 1 new attestation(s) from https://alice.btc.calendar.opentimestamps.org
    INFO:root:Success! Bitcoin attests data existed as of Wed Sep  7 05:56:43 2016 UTC

The client maintains a cache of timestamps it obtains from remote calendars, so
if you verify the same file again it'll use the cache:

    $ ./ots verify examples/incomplete.txt.ots
    INFO:root:Assuming target filename is 'examples/incomplete.txt'
    INFO:root:Got 1 attestation(s) from cache
    INFO:root:Success! Bitcoin attests data existed as of Wed Sep  7 05:56:43 2016 UTC

You can also upgrade an incomplete timestamp, which adds the path to the
Bitcoin blockchain to the timestamp itself:

    $ ./ots upgrade examples/incomplete.txt.ots
    INFO:root:Got 1 attestation(s) from cache
    INFO:root:Success! Timestamp is complete

Finally, you can get information on a timestamp, including the actual
commitment operations and attestations in it:

    $ ./ots info examples/two-calendars.txt.ots
    File sha256 hash: efaa174f68e59705757460f4f7d204bd2b535cfd194d9d945418732129404ddb
    Timestamp:
    append 839037eef449dec6dac322ca97347c45
    sha256
     -> append 6b4023b6edd3a0eeeb09e5d718723b9e
        sha256
        prepend 57d46515
        append eadd66b1688d5574
        verify PendingAttestation('https://alice.btc.calendar.opentimestamps.org')
     -> append a3ad701ef9f10535a84968b5a99d8580
        sha256
        prepend 57d46516
        append 647b90ea1b270a97
        verify PendingAttestation('https://bob.btc.calendar.opentimestamps.org')


### Timestamping and Verifying PGP Signed Git Commits

See `doc/git-integration.md`


### Timestamping Git Trees

Read the source code: `python-opentimestamps/opentimestamps/core/git.py`

This functionality needs more peer review before using it can be recommended.


## Privacy Security

Timestamping inherently records potentially revealing metadata - the current
time. If you create multiple timestamps in close succession it's quite likely
that an adversary will be able to link those timestmaps as related simply on
the basis of when they were created; if you make use of the timestamp multiple
files in one command functionality (`./ots stamp <file1> <file2> ... <fileN>`)
most of the commitment operations in the timestamps themselves will be
identical, providing an adversary very strong evidence that the files were
timestamped by the same person. Finally, the REST API used to communicate with
remote calendars doesn't currently attempt to provide any privacy, although it
could be modified to do so in the future (e.g. with prefix filters).

File contents *are* protected with nonces: a remote calendar learns nothing
about the contents of anything you timestamp as it only ever receives an opaque
and meaningless digest. Equally, if multiple files are timestamped at once,
each file is protected by an individual nonce; the timestamp for one file
reveals nothing about the contents of another file timestamped at the same
time.


## Compatibility Expectations

OpenTimestamps is alpha software, so it's possible that timestamp formats may
have to change in the future in non-backward-compatible ways. However it will
almost certainly be possible to write conversion tools for any
non-backwards-compatible changes.

It's very likely that the REST protocol used to communicate with calendars will
change, including in backwards incompatible ways. In the event happens you'll
just need to upgrade your client; existing timestamps will be unaffected.


## Known Issues

* Displaying Bitcoin timestamps down to the second is false precision, and
  misleading. But rounding off to the nearest day is over-doing it in the other
  direction.

* We should consider using the median time past + an offset instead of
  displaying Bitcoin block times directly (more generally, need to rigorously
  analyse what exactly a Bitcoin timestamp means, under what assumptions).

* Need unit tests for the client.

* While it's (hopefully!) not possible for a malicious timestamp to cause the
  verifier to use more than a few MB of RAM, or go into an infinite loop, it is
  currently possible to make the verifier crash with a stack overflow.

* Git tree re-hashing support fails on certain Unicode filenames; this appears
  to be due to bugs in the underlying GitPython library.

* Git annex support only works with the SHA256 and SHA256E backends.

* Errors in the Bitcoin RPC communication aren't handled in a user-friendly
  way.

* It's unclear if SSL certificates for remote calendars are checked correctly,
  probably not on most (all?) platforms.

* We don't do a good job sanity checking timestamps given to us by remote
  calendars. A malicious calendar could cause us to run out of RAM, as well as
  corrupt timestamps in (recoverable) ways (stack overflow comes to mind). Note
  the previous known issue!

* Due to the timestamp cache, a malicious calendar could also cause unrelated
  timestamps to fail validation. However it is _not_ possible for a malicious
  calendar to create a false-positive.
