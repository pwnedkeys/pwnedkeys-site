---
layout: default
title: Do you have a pwnedkey?
---
If you have a disclosed private key you wish to submit to the pwnedkeys
database, firstly: thank you!  Community contributions are how we get most
of our keys.  There are several options for submitting a pwnedkey.


# Method 1: Submit the full private key

The preferred method of submitting a private key is to e-mail us the
complete private key.  This ensures future compatibility and flexibility in
how the database and API evolve.

Keys for submission can be e-mailed as an attachment to
[`submit@pwnedkeys.com`](mailto:submit@pwnedkeys.com), optionally
GPG encrypted to [key
`2B9F7F03741E7305BF06C3CFC41B1E9E74F185F6`](pwnedkeys.gpg).
Please submit the key in PEM format if possible, or at least indicate the
format of the private key in your submission.


# Method 2: Provide a signed attestation of compromise

If, for some reason, you're unwilling to provide the private key itself, you
can instead provide an appropriate formatted attestation of key compromise,
which can be served via [the v1 API](search.html).  The easiest way to do that
is to use [the
pwnedkeys-response-generator](https://github.com/pwnedkeys/pwnedkeys-response-generator).
Installation and usage instructions are in [the
README](https://github.com/pwnedkeys/pwnedkeys-response-generator#readme).

E-mail the generated attestation, along with the associated public key
(preferably in PEM format), to
[`submit@pwnedkeys.com`](mailto:submit@pwnedkeys.com) as attachments.
