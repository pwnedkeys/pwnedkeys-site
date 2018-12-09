---
layout: default
title: pwnedkeys -- Frequently Asked Questions
---
You've got questions?  This FAQ (might) have answers!

* **Wait, what do you do?**

  From time to time, private keys (the part of a key which you're supposed to
  keep, well, *private*) become less private.  When that happens, it's a
  *really* bad idea to keep using them, because anyone can then pretend to be
  you.  Various organisations have a duty to try and stop users from using
  compromised keys, such as Certification Authorities (the people who hand out
  SSL certificates), but it's hard for them to know whether a key's been
  compromised because they're... everywhere.

  This site is intended to be *the* place where people can go to determine
  whether a key has been compromised, somehow, and should not be used.

* **How do keys get compromised?**

  A variety of ways.  Most often they just get leaked accidentally by the
  legitimate owner of the key, by putting it somewhere not private, like
  GitHub or pastebin.  Occasionally they'll be embedded in software which
  is publicly released, or stolen by someone breaking into a machine.
  Once or twice, a systemic weakness in a piece of software causes a whole
  *heap* of keys to be compromised all at once.

* **How does pwnedkeys work?**

  If you have a public key, you can generate a "thumbprint" of that key, and
  then ask [the pwnedkeys API](search.html) if that key has been compromised.

* **How do I know you're not lying to me?**

  When [the pwnedkeys API](search.html) sends back a response to your query
  saying "yes, this key is compromised", it also provide some data *signed by
  the private key*.  Since the only way to create that signature is by having
  possession of the private key, if someone can manage to get a signature on
  something that says "this key is pwned", then, well, one way or another, *it
  is*.

* **Where do you get compromised keys from?**

  Mostly by searching the Internet and finding them.  Some get [submitted by
  helpful individuals, too](submit.html).

* **What kinds of keys does pwnedkeys store?**

  The pwnedkeys database keeps records of 2048 bit and larger RSA keys, as well
  as elliptic-curve keys on the P-256, P-384, and P-521 curves.  Support for
  curve25519 and curve448 keys are planned.

  DSA keys, and RSA keys smaller than 2048 bits, are not kept, as these keys are
  generally considered of limited security, and in general should not be used at
  all, whether they are known to be compromised or not.

* **What do I do if my key is reported as compromised?**

  If you are the legitimate owner of the key, then you need to replace it with
  a new one as soon as possible, and work out how the previous one got compromised
  so it doesn't happen again.  If you're a CA or other organisation that wants to
  trust a presented key, *don't*.  There's absolutely no guarantee that the
  signature you're looking at comes from the legitimate owner of the key.

* **Why do you return compromise attestations as JWS objects?**

  Because it seemed like the least-worst option available.  It's nice to have a
  standardised, extensible structure for signed data, with broad software
  support.  Using some sort of X.509 data structure (such as a CSR, or even a
  self-signed certificate of some sort) was seriously considered, but it was
  decided that since one of the primary use-cases for a database of pwnedkeys
  was in the X.509 certificate community, creating certificates (even obviously
  bogus, self-signed ones) might cause confusion and doubt, so it was decided
  to steer well clear of all of that.

* **Who is behind this?**

  The primary developer and maintainer of pwnedkeys is Matt Palmer, a long-time
  security afficionado and generally grumpy geek.

* **I have more questions!**

  Feel free to [send an e-mail](mailto:contact@pwnedkeys.com) and ask away.
