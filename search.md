---
layout: default
title: discover if your key is pwned
---
If you have a public or private key, you can see if the key appears in the
pwnedkeys database using the pwnedkeys API.

The simplest way to do that is
to use the `pwnedkeys-query` command in the [`pwnedkeys-tools`](https://github.com/pwnedkeys/pwnedkeys-tools)
package.  It takes a key, public, private, or embedded in a certificate or CSR,
and queries the pwnedkeys API.  Very simple.  Installation and usage instructions
are in [the `pwnedkeys-tools` README](https://github.com/pwnedkeys/pwnedkeys-tools#readme).

If you wish to implement your own querying functionality, integrating it into
an existing key-using service, read on.

# Querying the pwnedkeys API

Making a query involves the following steps:

1. Calculate the SPKI fingerprint of the key.

1. Make a request to `https://v1.pwnedkeys.com/<fingerprint>`.  If the response
   to the request is a `200 OK`, then the key appears in the database, while a
   404 response indicates that the key doesn't appear in the pwnedkeys
   database.

1. (Optional) Verify that pwnedkeys does, indeed, have possession of the
   private key, by validating the JSON Web Signature returned in the response
   to the API request.

Let's examine each of those steps in more detail.


## Step 1: Calculate the SPKI fingerprint

The SPKI fingerprint of a key (or certificate) is the all-lowercase hex-encoded
SHA-256 hash of the DER-encoded form of [the `subjectPublicKeyInfo` ASN.1
structure](https://tools.ietf.org/html/rfc5280#section-4.1) representing a
given public key.  Try saying *that* ten times fast.

You can extract the SHA-256 fingerprint from a private key, CSR, or certificate
using the following command line (adapted from [the Mozilla
documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Public_Key_Pinning):

For an RSA private key:

> `openssl rsa -in rsa-key.pem -outform der -pubout | openssl dgst -sha256 -hex`

For an EC private key:

> `openssl ec -in ec-key.pem -outform der -pubout | openssl dgst -sha256 -hex`

For a CSR:

> `openssl req -in csr.pem -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -hex`

And, finally, for an X.509 certificate:

> `openssl x509 -in cert.pem -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -hex`

Many programming languages have libraries which make the extraction and
calculation of an SPKI fingerprint straightforward.  Just be careful when
you're choosing a function to call that it is (a) hex-encoded, (b) a
**SHA-256** hash (rather than SHA-1 or something else), and (c) taken over the
`subjectPublicKeyInfo` structure, and not the `subjectPublicKey` or entire
certificate.


### Sidebar: SPKI: The Gory Bits

(This section is only important if you need to implement fingerprint calculation
in an environment which doesn't have any existing means of calculating SPKI
fingerprints).

The [`subjectPublicKeyInfo`
structure](https://tools.ietf.org/html/rfc5280#section-4.1) is, in and of
itself, fairly straightforward -- a sequence of an algorithm specifier (itself
a sequence) and a string containing the actual key data.  However, since every
different type of key has its own ideas about what goes into a public key, and
the parameters of an algorithm can vary quite considerably, it can be a bit
of a wrestle to support it all.

For RSA keys, [RFC3279](https://tools.ietf.org/html/rfc3279#section-2.3.1) says
the `subjectPublicKey` should be the DER-encoded sequence containing the
modulus (`n`) and public exponent (`e`) of the key.  The algorithm identifier
is just `rsaEncryption`, and the parameters sequence element MUST be an ASN1 NULL.

EC keys are a bit trickier.  [RFC5480](https://tools.ietf.org/html/rfc5480) is
the full reference, but just quickly, you need to set the algorithm identifier
to `id-ecPublicKey`, while the algorithm parameters is just an object ID specifying
which particular curve is being used (there are a lot of possibilities in the
RFC, but really only a few ever get used).  The `subjectPublicKey` is a direct
encoding of the "point" which represents the public key.  One annoyance of EC
keys is that the same point can be stored in "compressed" or "uncompressed" form,
which results in two different fingerprints.  To avoid potential problems,
pwnedkeys actually calculates *both* fingerprints and will respond to a request
for the fingerprint of either.  You're welcome, world.


## Step 2: Query the pwnedkeys API

To ask the pwnedkeys API whether a key is in the pwnedkeys database, you
make a `GET` requests to a URL of the following form:

    https://v1.pwnedkeys.com/<fingerprint>

Where `<fingerprint>` is the all-lowercase, hex-encoded fingerprint of the key
you wish to query for.  The HTTP request `Accept` header must allow responses
in the `application/json` content type, because that's what's going to
come back.

If the queried key exists in the pwnedkeys database, the HTTP response code
will be `200` (`OK`), and the response body will be a JSON Web Signature which
you can use to verify that the key really is pwned, and we're not just making
things up (more on that in the next section).  If the key is *not* in the
pwnedkeys database, then the response code will be `404` (`Not Found`).

If there was a problem with your request (you didn't provide a valid key
fingerprint, for instance) you will receive a `400` (`Bad Request`) response
code (or another `4xx` series code other than `404`), while if there is a
server-side problem, you will receive a `5xx` series response code.  A
connection failure, premature disconnection, or response time out is also a
server-side problem.


## Step 3: Validate the Response

Whilst pwnedkeys is a *very* trustworthy service, and you should definitely
trust what the API says, it's always a good idea to verify that the key is,
in fact, pwned.

To help you with that, every response that indicates the queried key is
in the pwnedkeys database will include a [JSON Web Signature](https://tools.ietf.org/html/rfc7515)
object which has been signed with the private key being queried.  Thus,
if you have the corresponding public key (from a CSR or certificate, for
instance) you can validate the signature and be quite sure that the key is,
in fact, pwned.

Whilst you can read [the RFC](https://tools.ietf.org/html/rfc7515) and figure
out what's going on, the following description of what a pwnedkeys API response
looks like should be enough to get you going.  If you're familiar with JWS,
the short version is that we're using a flattened JSON encoding with only a protected
header.  If you're not a JWS afficionado, read on.

The HTTP response as a whole is a JSON object -- that is, a bunch of
`"key":"value"` pairs wrapped in braces.  It contains the following keys:

* **`"protected"`** -- a string containing a URL-safe, base64-encoded, JSON
  object which is itself the "protected headers" of the signature.  The keys in
  *this* sub-object will be:
  
  * **`"alg"`** -- the signature algorithm used to generate the signature.  Which
	algorithm is used for signing is determined by the type of key that was
	looked up.  For RSA keys, `alg` will be `RS256` (an RSASSA-PKCS1-v1_5
	signature over a SHA-256 hash), whilst for elliptic curve keys, you'll get
	one of `ES256` (an ECDSA signature using a P-256 key with a SHA-256 hash),
	`ES384` (an ECDSA signature using a P-384 key with a SHA-384 hash), or
	`ES512` (an ECDSA signature using a P-521 key with a SHA-512 hash).

	References to the exact details of each of these schemes can be found in
	[the relevant IANA
	registry](https://www.iana.org/assignments/jose/jose.xhtml#web-signature-encryption-algorithms).

	Note that this algorithm list is not exhaustive, and may be extended over
	time as more key types are supported by the pwnedkeys database.  However,
	since you should only ever be getting a signature matching the type of
	key you are querying for, you shouldn't get a signature algorithm you're
	not prepared to handle.

  * **`"kid"`** -- this is the hex-encoded, SHA-256 fingerprint of the queried
	key, which is the same as the fingerprint used to query the key.

* **`payload`** -- a string containing a URL-safe, base64-encoded string, which
  attests that the key has, in fact, been pwned.  Whilst the exact contents of
  the payload are subject to change, it is guaranteed to contain the ASCII
  string "key is pwned" somewhere, and be no more than 1024 bytes in length
  (before base64 encoding).

* **`signature`** -- a string containing a URL-safe, base64-encoded JWS
  signature, generated as per [RFC7515 s5.1](https://tools.ietf.org/html/rfc7515#section-5.1).

Note: a "[URL-safe base64-encoded string](https://tools.ietf.org/html/rfc4648)"
is just like a [regular base64-encoded
string](https://en.wikipedia.org/wiki/Base64), except the `+` is replaced with
`-`, `/` is replaced with `_`, and the trailing "padding" equals signs are
omitted.


## Examples

To help you in verifying that your querying code is working correctly, you can
use the following "test" RSA and ECDSA private keys, which have been used to
generate a dummy CSR and self-signed certificate.  They can be looked up in the
pwnedkeys API, and should return a JWS that you can verify.

<table class="bordered">
 <thead>
  <tr>
   <th>Type</th>
   <th>Fingerprint(s)</th>
   <th>Key</th>
   <th>CSR</th>
   <th>Cert</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td>2048 RSA</td>
   <td><tt>9e03b56749abe821a6f5299d6f634b35404975f0552eb3347bf3adfad9af1109</tt></td>
   <td>
    <a href="example/rsa2048_key.pem">PEM</a>
	<br>
	<a href="example/rsa2048_key.der">DER</a>
   </td>
   <td>
    <a href="example/rsa_2048_csr.pem">PEM</a>
	<br>
	<a href="example/rsa_2048_csr.der">DER</a>
   </td>
   <td>
    <a href="example/rsa_2048_cert.pem">PEM</a>
	<br>
	<a href="example/rsa_2048_cert.der">DER</a>
   </td>
  </tr>
  <tr>
   <td>P-256 EC</td>
   <td>
    <tt>819f7d1dcd9f07bfcb59b7699f68994d89390c3bcd498cf7fb2e1ef3d272b89b</tt>
	<br>
	<tt>316194405bf1c56c3395c4b6fcf32af83ca0e273fbf0832ef8364069a178ad75</tt>
	</td>
   <td>
    <a href="example/p256_key.pem">PEM</a>
	<br>
	<a href="example/p256_key.der">DER</a>
   </td>
   <td>
    <a href="example/p256_csr.pem">PEM</a>
	<br>
	<a href="example/p256_csr.der">DER</a>
   </td>
   <td>
    <a href="example/p256_cert.pem">PEM</a>
	<br>
	<a href="example/p256_cert.der">DER</a>
   </td>
  </tr>
 </tbody>
</table>
