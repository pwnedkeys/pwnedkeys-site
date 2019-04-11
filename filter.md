---
layout: default
title: Bloom filter format, version 1
---
If you are checking a lot of keys, and you don't expect that many of them
actually *are* compromised, hitting the pwnedkeys API for every single key can
be a bit of a buzzkill.  To make your life easier, we provide a [bloom
filter](https://en.wikipedia.org/wiki/Bloom_filter) dataset for [commercial
users of pwnedkeys.com](commercial.html), which you can use
to answer the question "is this key probably known to be compromised?"

One properly of a bloom filter is that it can tell you that a key is
*definitely not* in the set of known-compromised keys, but it can only say that
the key is *probably* in the set.  So if the bloom filter comes back
"positive", you still need to run that key against the API, both to get a
definitive "yes, this is compromised" answer, and also to collect the
verifiable compromise attestation.  However, assuming that you are testing keys
at random, only having to hit [the pwnedkeys API](api/index.html) when the
filter comes back positive should be a significant reduction in traffic.

This page describes the file format used to store the bloom filter data
provided by pwnedkeys.com, and also describes how to query the filter.


# File Format

The filter file consists of two parts:

1. a header containing an identity marker, version and creation time
   information, entry count, and the filter parameters; and

2. The filter data itself.


# File Header

The first 24 bytes of the filter data file comprise the file header, which
contains the filter metadata.

The fields are as follows.  All integers are in network (big endian) byte order.

<table>
 <thead>
  <tr>
   <th>Offset<br>(bytes)</th>
   <th>Size<br>(bytes)</th>
   <th>Type</th>
   <th>Description</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td>0</td>
   <td>6</td>
   <td>String</td>
   <td>
    <b>File format marker</b>.  This will always be the string <tt>pkbfv1</tt>,
    which corresponds to the hex bytes <tt>70 6B 62 66 76 31</tt>.
   </td>
  </tr>
  <tr>
   <td>6</td>
   <td>4</td>
   <td>32-bit unsigned integer</td>
   <td>
    <b>File revision counter</b>.  The revision counter is used when obtaining
    incremental updates to the filter data file.  See the section "Updating
    the data file", below, for more information.
   </td>
  </tr>
  <tr>
   <td>10</td>
   <td>8</td>
   <td>64-bit unsigned integer</td>
   <td>
    <b>Last update time</b>.  Provides an indication of when this filter file's data
    was last updated.  Represents the number of seconds elapsed since the
    UNIX epoch (1970-01-01 00:00:00 UTC).
   </td>
  </tr>
  <tr>
   <td>18</td>
   <td>4</td>
   <td>32-bit unsigned integer</td>
   <td>
    <b>Entry count</b>.  Indicates how many keys are represented in the filter data.
    This can be used, along with the filter parameters, to estimate the
    false-positive rate likely to exist within the data.
   </td>
  </tr>
  <tr>
   <td>22</td>
   <td>1</td>
   <td>8-bit unsigned integer</td>
   <td>
    <b>Hash count</b>.  Specifies the <tt>k</tt> parameter of the filter, which is
    the number of bits which are set for each entry in the filter.
   </td>
  </tr>
  <tr>
   <td>23</td>
   <td>1</td>
   <td>8-bit unsigned integer</td>
   <td>
    <b>Hash length</b>.  Specifies the length of each value in the hash, in
    bits.  The size of the lookup table, in bits, (known as <tt>m</tt>) is thus
    <tt>2<sup>(hash length)</sup></tt>.  This can also be used to find the size
    of the bloom filter data on disk (or in memory) in bytes, by
    <tt>2<sup>(hash length)</sup> / 8</tt>.
   </td>
  </tr>
  <tr>
   <td>24</td>
   <td>variable</td>
   <td>bit field</td>
   <td>
    <b>Bloom filter data</b>.  It is used as a bit array.
   </td>
  </tr>
 </tbody>
</table>

# Querying the bloom filter

To answer the question, "is this key probably known to be compromised?", you need to
determine the bloom filter bit positions for the [SPKI
fingerprint](../api/v1.html#step-1-calculate-the-spki-fingerprint) of the key
you're testing, and then test whether all those bit positions are set, using
the following algorithm:

1. Extract or construct the [SPKI data structure](../api/v1.html#step-1-calculate-the-spki-fingerprint) of the key you're testing,
   and convert that into a DER-encoded binary string, which will will refer to as `spki`.

2. Calculate XXH64(`spki`, `0`) -- that is, apply the [XXH64 hash function](https://github.com/Cyan4973/xxHash) over the DER-encoded SPKI,
   with a hash seed value of `0`.
   We will refer to this value as <tt>h<sub>1</sub></tt>.

3. Calculate XXH64(`spki`, `1`).  If this value is even,
   it must be incremented by one, in order to make it odd.  We will refer to
   this value as <tt>h<sub>2</sub></tt>.

4. Calculate a total of `k` numbers, representing bits in the bloom filter to test,
   using the following formula:

    <tt>f<sub>i</sub> = (h<sub>1</sub> + i * h<sub>2</sub> + (i<sup>3</sup> - i) / 6) mod m</tt>

    Where `i` is in the range `0` to `k - 1`.

5. For each of the `k` numbers calculated, which are each in the range `0` to
   `m - 1`, check whether the bit at that position in the bloom filter data is
   set.  If any relevant bit position is unset (equal to `0`), then the key you
   are testing definitely does *not* exist in the filter, and the algorithm can
   be terminated with the result "not known to be compromised".

6. If all bits at the positions calculated are set (equal to `1`), then the key
   being tested *probably* exists in the pwnedkeys database.  You should
   perform an online lookup using [the pwnedkeys API](api/index.html) to be
   sure, and also to retrieve a cryptographic proof of compromise.


# Existing Implementations

The reference implementation of the filter querying code is the [pwnedkeys-filter](https://rubygems.org/gem/pwnedkeys-filter) ruby gem.

If you have an open source implementation you would like to have listed here,
please [get in touch](mailto:contact@pwnedkeys.com).


# Examples

For testing purposes, the following bloom filter data files are available.
They each contain the [canonical pwnedkeys example keys](search.html#examples)
only.  There are a number of different combinations of hash count and hash
length, so you can validate the operation of your lookup code in different
conditions.

* [2 hashes, 4 bit hash length](examples/2_4_filter_example.pkbf)
* [3 hashes, 6 bit hash length](examples/3_6_filter_example.pkbf)
* [5 hashes, 12 bit hash length](examples/5_12_filter_example.pkbf)
* [12 hashes, 18 bit hash length](examples/12_18_filter_example.pkbf)


# References

The algorithm for calculating the bit positions to test comes from section 5.2 of [Bloom
Filters in Probabilistic Verification](http://www.ccs.neu.edu/home/pete/pub/bloom-filters-verification.pdf), by
Peter C. Dilinger and Panagiotis Manolios.
