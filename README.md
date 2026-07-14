# Introduction

This is BeagleHash and friends: a family of hash functions developed
using *genetic algorithm* techniques. Each hash function has a different
set of trade offs, primarily relating to the size of the seed, the
run time state size of the hash, the method of seeding the hashes'
state, the block size for reads, and the size of the final hash.

| Hash | WordBits | SeedBits | StateBits | HashBits |
| ---- | -------- | -------- | --------- | -------- |
| BeagleHash | 64 | Variable | 128 | 64 |
| Zaphod64 | 64 | 191 | 192 | 64 |
| StadtX | 64 | 128 | 256 | 64 |
| Zaphod32 | 32 | 95 | 96 | 32 |
| Phat4 | 32 | 96 | 128 | 32 |
| SBOX | 64 | 128 | 524480 | 64 |

BeagleHash is named in honor of the the H.M.S. Beagle, the ship
that carried Charles Darwin to the Galapogos Islands. Zaphod is named
after the character from *The Hitchhiker's Guide to the Galaxy,* and is
also a play on the fact that Microsoft has patented a hash called "Marvin."
StadtX is a hash function inspired by the MetroHash family of hashes,
and was named accordingly, "stadt" being the German word for city.
Phat4 needs a new name.

SBOX is named after how it works, which is substitution box (compare tabulation) hashing.
SBOX is a toy, designed to demonstrate how "perfect unbreakable hashing" might behave.
It hashes by looking up a random 64 bit value for each input byte (and then switches to
Zaphod64-like hashing after 32 bytes). The purpose is purely for testing. I do not
recommend you use it.

# What is it for

The primary intention of the hash functions contained in this
repository is for use in scripting languages such as Perl and other
contexts where there is a single seed used to hash many keys,
including ones from untrusted sources, and where there may be
"leakage" of details about how the hash behaves. In particular, one
of the assumptions of the hash functions contained here is that
they will be *rarely* seeded, but that we will hash many times
with the same seed.

## BeagleHash

> Errata: Some of these functions do not appear to exist.

BeagleHash is one core function exposed as a set of different
functions each intended for a different seed or hash size.

The core routine is the function beagle_hash_64_128_a() which
expects a 128 bit random seed which has been "prepared" for use.
Specifically the seed is expected to have certain properties:

    - Neither the first 64 bits nor the last may be either all
      zeros, nor all ones.
    - There must be a reasonable mix of ones and zeros.

This function is wrapped by various other functions:

    beagle_hash_64_127_a()      beagle_hash_32_127_a()
    beagle_hash_64_112_a()      beagle_hash_32_112_a()
    beagle_hash_64_96_a()       beagle_hash_32_96_a()
    beagle_hash_64_64_a()       beagle_hash_32_64_a()

The _96, _112 and _127 variants are exposed primarily for testing
and are NOT recommended for general use as they include key
preparation as part of their implementation. Instead one should
use the functions

    beagle_hash_seed_prep_127_128_a()
    beagle_hash_seed_prep_112_128_a()
    beagle_hash_seed_prep_96_128_a()
    beagle_hash_seed_prep_64_128_a()

to prepare the seed for use with

    beagle_hash_64_128_a()
    beagle_hash_32_128_a()

both of which do no seed preparation at all.

Only the _64 variant is recommended for use with variable seeds,
as it uses a fast method for ensuring that seed is of sufficient
quality.

# Hash function security

Security for a hash function is a complex matter. Many contexts do not
rely on security. If you are hashing the words in an "actual" dictionary,
or hashing pointer addresses, then you probably don't need to worry about
the security of that hash function and can focus on speed and suitability
to the data being hashed. On the other hand, if you are using hashing for
signing messages, then you really need to worry about using an academically
peer reviewed and *cryptographic* grade hash function.

Both of these markets are relatively well served by the existing corpus
of hash functions, and it is relatively straightforward to choose one
appropriate to the task at hand.

However, there is a middle ground between these two use cases where things
are less clear. Perl is a good example of this middle ground. Perl, like
many scripting languages, makes heavy use of hashing and hash tables. The
hash function it uses for strings is seeded once (at process start up) and
is then used for all subsequent hashing requirements: from internal purposes
like namespace management to application level logic like hashing arbitrary
user input, for example building a hash of the arguments to a web request. While
the hash values generated are considered "private" to the process (and transient
to that process), information about the hash function being used can and often does
leak in the form of printing out the keys of a hash in native key order.
Another key point here is that the hash table size is variable. In many of
the tables, only a few bits (usually the low bits) of the hash are actually used.

Since we have to supporting hashing arbitrary user data, we have to assume that
sometimes that data may be hostile, and that someone may be setting up a collision
attack on the hash function. We must ensure that our hash function is
sufficiently robust to make collision attacks impractical, which in turn means
we have to make seed-discovery attacks impractical.

## Collision attacks

### Universal multicollision attacks

The most serious type of collision attack is the "multi-collision" attack, where
it is possible to construct colliding keys *without knowing the seed* of the hash.
Hashes which are based around the 64 bit Intel CRC32 intrinsic often suffer
from this problem. It takes a few seconds to find a CRC32 collision, and once you
have that, you can manufacture arbitrary numbers of keys which collide. Even if
the seed changes, and thus the hash value produced changes, all the keys will
still collide. There are other variants to this, for instance some hashes have
the property that there is a key K such that H(P)=H(PK), or have the associative
property such that H(AB)=H(BA). Any such weaknesses are a fatal blow to a hash
function that needs to be robust to untrusted input data.

At this time I do now know of any multi-collision attacks on the hash functions
in this package. I believe that they are by design robust to such attacks, always
mixing the length of the key into the state during hashing, and of sufficient
complexity and construction that such attacks are unlikely.

### Brute-force attacks

An unfortunate fact of life is that if an attacker knows the seed for your hash
function, and you are only using a few bits of the hash, an attacker can
readily brute force collisions. For instance, if you are using 32 bits of the hash,
you can brute force a collision of keys longer than 32 bits in about 160k
random tries (collisions in a hash are normal, excess collisions are not). What
this means is that our hash function must be robust to key-discovery attacks.
In other words, the problem moves from avoiding collisions to keeping our
seed secret.

## Seed discovery attacks

### Brute-force

If the seed is small enough, we can brute force the hash function and simply
try every seed and see what hash (or key order) we get. Note the hash size doesnt
matter much here: if the seed is 32 bits, the hash function can be brute forced
in short order. This means that any decent hash function needs to have a seed of at
least 64 bits, if not longer.

All of the hash functions in this package support a seed larger than 64 bits.

### Solver Attacks

If an attacker knows the hash function being attacked (and even if they don't to a
certain extent), they can use an SMT solver to compute the seed from a set of input/output pairs.
This means that the complexity of the hash function must be sufficient to make
solving the hash impractical. Modern solver technology is very advanced and can
make short order of an insufficiently complex hash function.

I do not know the solver status of these hash functions. If I had to guess, I would
say that the solver resistance of the hash function is inversely proportional to the
hash function's speed, but obviously the relative sizes of the seed, state and hash
will play a big role. This needs further study.

## Relatively Speaking

All of these hash functions use less state, or use less complex mix functions, than
even SipHash 1-3. On the other hand, they are faster than
any commonly used form of SipHash and probably have a higher level of security than most
"non-cryptographic" hash functions.

# Status

This project is a work in progress. It may change at any time until
this notice is removed. Please do not use this code in any production
systems without contacting Yves Orton first.

# License

You may use this software either under the LGPL 3.0 or later, or under
express license after contacting the author.

# Author

Yves Orton wrote and ran the software (currently not included in this
package) that evolved the internals used by BeagleHash, and wrote
the "skeleton" that the genetic-algorithm "filled in."

Yakesuanyan revised the documentation and is working on further analysis
of the hash functions.

# Acknowledgements

The author would like to thank the following people who have one way or
another influenced the development of this software:

* Damian Gryski - A smart cookie who has interest in this area and who
                    has helped with some of the analysis.
* Brett Mulvey  - Author of http://papa.bretmulvey.com/post/124027987928/hash-functions
* JP Aumasson "veorq" - One of the authors of SipHash which has been very
                    influential in how this function was structured.
* Ryan Bastic   - Porting, Doc fixes, actually building stuff, etc.
* Larry Wall    - Author of Perl, where I got my chops, and why I started
                    on this in the first place.

