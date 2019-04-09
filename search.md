---
layout: default
title: discover if your key is pwned
---
If you have a public or private key, you can see if the key appears in the
pwnedkeys database using the [pwnedkeys API](/api/index.html).

The list of tools and libraries given below may be helpful to get you
integrating pwnedkeys API queries into your own systems.  If there is no
suitable implementation already available, the [API
specification](/api/index.html) should give you enough information to be able
to develop your own integration.  To have your library or tool added below,
please [contact us](mailto:contact@pwnedkeys.com), or [create a GitHub
PR](https://github.com/pwnedkeys/pwnedkeys-site).


# Tools

All the tools below will both query the API and validate the response.

* [`pwnedkeys-query`](https://github.com/pwnedkeys/pwnedkeys-tools) -- a Ruby-based
  command-line client.


# Libraries

All libraries included below support both querying and validating responses.

* **Ruby**: [`pwnedkeys-api-client`](https://github.com/pwnedkeys/pwnedkeys-api-client)
  [gem](https://rubygems.org/gems/pwnedkeys-api-client).

* **Golang**:
  [`github.com/adamdecaf/pwnedkeys`](https://github.com/adamdecaf/pwnedkeys),
  by Adam Shannon.


# Examples

To help you in verifying that your querying code is working correctly, you can
use the following "test" RSA and ECDSA private keys, which have been used to
generate a dummy CSR and self-signed certificate.  They can be looked up in the
pwnedkeys API, and should return a "yes, this is pwned" response at all times.

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
    <a href="examples/rsa2048_key.pem">PEM</a>
	<br>
	<a href="examples/rsa2048_key.der">DER</a>
   </td>
   <td>
    <a href="examples/rsa2048_csr.pem">PEM</a>
	<br>
	<a href="examples/rsa2048_csr.der">DER</a>
   </td>
   <td>
    <a href="examples/rsa2048_cert.pem">PEM</a>
	<br>
	<a href="examples/rsa2048_cert.der">DER</a>
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
    <a href="examples/p256_key.pem">PEM</a>
	<br>
	<a href="examples/p256_key.der">DER</a>
   </td>
   <td>
    <a href="examples/p256_csr.pem">PEM</a>
	<br>
	<a href="examples/p256_csr.der">DER</a>
   </td>
   <td>
    <a href="examples/p256_cert.pem">PEM</a>
	<br>
	<a href="examples/p256_cert.der">DER</a>
   </td>
  </tr>
 </tbody>
</table>
