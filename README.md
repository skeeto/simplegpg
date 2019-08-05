# SimpleGPG

SimpleGPG is a Bourne shell script wrapper around GnuPG intended to
provide a simple, keyring-less signature interface very similar to
[Signify][signify], but that is still fully compatible with the OpenPGP
ecosystem. GnuPG users can verify SimpleGPG signatures without being
aware that it exists, and SimpleGPG can verify signatures made by GnuPG
users.

The main difference is that **the GnuPG keyring is not used**, and keys
are instead managed as standalone files. This bypasses the usual
problems with the web of trust and keyservers. SimpleGPG is like an
audio player that doesn't want to manage and own your entire audio
library.

Note: Despite the similar interface, the keys and signatures produced by
SimpleGPG are *not compatible* with Signify or [Minisign][minisign].
They have the same underlying primitive (Ed25519), but OpenPGP
signatures are fundamentally incompatible with Signify signatures.

## Usage

In terms of user interface, SimpleGPG could nearly serve as a drop-in
replacement for Signify:

```
usage: simplegpg -G [-n] [-c comment] -p pubkey -s seckey
       simplegpg -S [-x sigfile] -s seckey -m message
       simplegpg -V [-q] [-x sigfile] -p pubkey -m message
```

* `-G` Generate a new keypair.
* `-S` Sign a message, creating a signature.
* `-V` Verify a signature with a message

OpenPGP data is always ASCII-armored with no option for binary output,
though binary input is still accepted as input. The file conventions for
SimpleGPG are a bit different in order to accommodate PGP conventions:

* "*keyname*.asc" for public keys
* "*keyname*.pgp" for secret keys
* "*message*.sig" for signatures

Why not ".pub" and ".sec"? Because this doesn't follow PGP's file
conventions, plus this differentiates these files from Signify/Minisign.
Why ".pgp" instead of ".gpg"? Because this file is just OpenPGP data,
and there's nothing specific to GnuPG.

The `-c` comment sets the OpenPGP user ID for the keypair. It's also
added as a human-readable "Comment:" to the key files. Unfortunately,
due to limitations of OpenPGP, there's no reliable way to indicate that
the comment is untrusted, as Signify does.

### Example

Generate a new keypair protected by a passphrase:

    $ simplegpg -G -p keyname.asc -s keyname.pgp

Publish and distribute `keyname.asc` via the appropriate channels for
your community. Later, sign `document.txt`, producing `document.txt.sig`:

    $ simplegpg -S -s keyname.pgp -m document.txt

On the other end, given both these files:

    $ simplegpg -V -p keyname.asc -m document.txt

## Notes

The signify options `-C` (checksum list), `-e` (embed), and `-z` (gzip
archive) are currently unsupported. The `-t` (keytype) option doesn't
make sense in this context.

To create the illusion that no keyring is used, a temporary keyring
directory is created, the operation is performed, and the keyring is
destroyed. Currently this keyring is created in the working directory,
but a more suitable (and configurable) location should be used instead.
GnuPG requires a keyring to do anything, so there's no way around it.

The String-to-Key (S2K) algorithm, i.e. the passphrase KDF, is run
*three times* when using `-G`. Attacks on the protection passphrase only
need to run it once per guess, giving attackers and edge. This is simply
a limitation of GnuPG. The problem is that GnuPG uses its own internal
S2K scheme separate from the exported, OpenPGP S2K scheme. When
generating a key, the internal S2K is run on the passphrase for storage
on the internal keyring. Then to export the secret key it's run a second
time to decrypt it, then a third time with the export S2K to re-encrypt
it.

The GnuPG interface is not powerful enough to fix this. The protection
passphrase settings cannot be controlled independently when exporting a
key (`--s2k-*` options are silently ignored), and gpg-agent isn't smart
enough to learn freshly-generated keys. Fortunately this isn't a problem
for `-S` since gpg-agent learns secret keys when their imported.

Just as Signify keys have no concept of expiration, punting that
question up to the context in which it's used, SimpleGPG keys are
generated with no expiration date. Perhaps this feature of OpenPGP
should be used, though?

GnuPG is loud and noisy with useless messages despite `--quiet` and
`--batch`, so, in several cases, the script redirects its output to
`/dev/null` and relies only on the exit status. However, when something
unusual goes wrong, the script's error messages are vague since it can't
tell exactly what went wrong, just that GnuPG failed. The problem is
that GnuPG does a poor job at communicating errors (see [EFAIL][efail]),
and even `--status-fd` doesn't convey enough information. For a great
example of how shell commands *should* communicate a variety of possible
error conditions to scripts in a simple way, see curl.

### passphrase2pgp

SimpleGPG meshes perfectly with [passphrase2pgp][p2p]. There's no need
to produce or store a secret key ("keyname.pgp") since it's stored in
your brain. It covers both `-G` and `-S`, leaving `-V` to SimpleGPG.

    $ passphrase2pgp -ap > keyname.asc
    $ passphrase2pgp -S document.txt
    $ simplegpg -V -p keyname.asc -m document.txt


[efail]: https://efail.de/
[minisign]: https://jedisct1.github.io/minisign/
[p2p]: https://github.com/skeeto/passphrase2pgp
[signify]: https://man.openbsd.org/signify
