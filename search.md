---
layout: default
title: discover if your key is pwned
---
If you have a public or private key, you can see if the key appears in the
pwnedkeys database using the pwnedkeys API.  It involves the following steps:

1. Calculate the SHA-256 thumbprint of the key.

1. Make a request to `https://v1.pwnedkeys.com/<thumbprint>`.  If the response
   to the request is a `200 OK`, then the key appears in the database, while a
   404 response indicates that the key doesn't appear in the pwnedkeys
   database.

1. (Optional) Verify that pwnedkeys does, indeed, have possession of the
   private key, by validating the JSON Web Signature returned in the response
   to the API request.

Let's examine each of those steps in more detail.


# Calculate the SHA-256 Thumbprint

In general, a thumbprint is a hash of the public key parameters of a key.  For our
purposes, we use the same algorithm as that used in X.509 certificates, with the
hash encoded as a hex string.

You can extract the SHA-256 thumbprint from a private key, CSR, or certificate using
the following command line (adapted from [the Mozilla documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Public_Key_Pinning):

For an RSA private key:

> `openssl rsa -in rsa-key.pem -outform der -pubout | openssl dgst -sha256 -hex`

For an EC private key:

> `openssl ec -in ec-key.pem -outform der -pubout | openssl dgst -sha256 -hex`

For a CSR:

> `openssl req -in csr.pem -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -hex`

And, finally, for an X.509 certificate:

> `openssl x509 -in cert.pem -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -hex`


## Gory Details

(This section is only important if you need to implement thumbprint calculation
in an environment which doesn't have any existing means of calculating thumbprints)

An [X.509 key ID
thumbprint](https://tools.ietf.org/html/rfc5280#section-4.2.1.2) is the SHA-256
hash of the `subjectPublicKey` bit string.  What this looks like is dependent
on the type of key, and is typically defined in an RFC somewhere.

For RSA keys, [RFC3279](https://tools.ietf.org/html/rfc3279#section-2.3.1) says
that it should be the DER-encoded sequence containing the modulus (`n`) and
public exponent (`e`) of the key.

For ECDSA keys, [RFC5480](https://tools.ietf.org/html/rfc5480#section-2.2) has
a couple of options.

For other types of keys, if you're using them you probably know where to find
the details of what the `subjectPublicKey` looks like.


# Query the pwnedkeys API

To ask the pwnedkeys API whether a key is in the pwnedkeys database, you
make a `GET` requests to a URL of the following form:

    https://v1.pwnedkeys.com/<thumbprint>

Where `<thumbprint>` is the all-lowercase, hex-encoded thumbprint of the key
you wish to query for.  The HTTP request `Accept` header must allow responses
in the `application/json` content type, because that's what's going to
come back.

If the queried key exists in the pwnedkeys database, the HTTP response code
will be `200` (`OK`), and the response body will be a JSON Web Signature which
you can use to verify that the key really is pwned, and we're not just making
things up (more on that in the next section).  If the key is *not* in the
pwnedkeys database, then the response code will be `404` (`Not Found`).

If there was a problem with your request (you didn't provide a valid key
thumbprint, for instance) you will receive a `400` (`Bad Request`) response
code (or another `4xx` series code other than `404`), while if there is a
server-side problem, you will receive a `5xx` series response code.  A
connection failure, premature disconnection, or response time out is also a
server-side problem.


# Validate the Response

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
  
  * **`"alg"` -- the signature algorithm used to generate the signature.  Which
	algorithm is used for signing is determined by the type of key that was
	looked up.  For RSA keys, `alg` will be `RS256` (an RSASSA-PKCS1-v1_5
	signature over a SHA-256 hash), whilst for elliptic curve keys, you'll get
	one of `EC256` (an ECDSA signature using a P-256 key over a SHA-256 hash),
	`EC384` (an ECDSA signature using a P-384 key over a SHA-384 hash), or
	`EC512` (an ECDSA signature using a P-521 key over a SHA-512 hash).  If
	you're on the cutting edge and are using [Edwards-Curve
	keys](https://tools.ietf.org/html/rfc8032), then the value of `alg` will be
	`[EdDSA](https://tools.ietf.org/html/rfc8037)`.

	References to the exact details of each of these schemes can be found in
	[the relevant IANA
	registry](https://www.iana.org/assignments/jose/jose.xhtml#web-signature-encryption-algorithms).

	Note that this algorithm list is not exhaustive, and may be extended over
	time as more key types are supported by the pwnedkeys database.  To prevent
	shenanigans, you RFC2119-MUST ensure that the signature algorithm of the
	response is the correct one for the type of key you're querying.

  * **`"kid"` -- this is the hex-encoded, SHA-256 thumbprint of the queried
	key, which is the same as the thumbprint used to query the key.

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


# Examples

To help you in verifying that your querying code is working correctly, you can
use the following "test" RSA and ECDSA private keys, which have been used to
generate a dummy CSR and self-signed certificate.  They can be looked up in the
pwnedkeys API, and should return JWS that you can verify.

<table>
 <thead>
  <tr>
   <th>Key type</th>
   <th>Thumbprint</th>
   <th>Key</th>
   <th>CSR</th>
   <th>Cert</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td>2048 RSA</td>
   <td><tt>xyzzy</tt></td>
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
   <td><tt>xyzzy</tt></td>
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
