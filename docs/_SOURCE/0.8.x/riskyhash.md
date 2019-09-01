---
title: facil.io - Risky Hash
sidebar: 0.8.x/_sidebar.md
---
# {{{title}}}

Risky Hash isn't a stable Hash algorithm, it is more of a statement - this algorithm was chosen although it is "risky", not cryptographically safe.

The assumption is that the risk management is handled by other modules. In facil.io, Risky Hash is used for the Hash Map keys and the risk management is handled by the Map type (which could accept malicious hashes).

The facil.io implementation MAY change at any time, to mitigate risk factors, improve performance or any other reason.

## Current Implementation

Risky Hash is a keyed hashing function which was inspired by both xxHash and SipHash.

It's meant to provide a fast alternative to SipHash, under the assumption that some of the security claims made by SipHash are actually a (hash) Map implementation concern.

Risky Hash wasn't properly tested for attack resistance and shouldn't be used to resist [hash flooding attacks](http://emboss.github.io/blog/2012/12/14/breaking-murmur-hash-flooding-dos-reloaded/) (see [here](https://medium.freecodecamp.org/hash-table-attack-8e4371fc5261)). Hash flooding attacks are decidedly a Hash Map concern. A Map implementation should safe regardless of Hash values.

Risky Hash was tested with [`SMHasher`](https://github.com/rurban/smhasher) ([see results](#smhasher-results)).

Sometime around 2019, the testing suit was updated in a way that exposed an issue Risky Hash has with sparsely hashed data. This was mitigated (but not completely resolved) by an update to the algorithm.

A non-streaming [reference implementation in C is attached](#in-code) The code is easy to read and should be considered an integral part of this specification.

## Status

> Risky Hash is still under development and review. This specification should be considered a working draft.
 
> Risky Hash should be limited to testing and safe environments until it's fully analyzed and reviewed.

This is the third draft of the RiskyHash algorithm and it incorporates updates from community driven feedback.

* Florian Weber (@Florianjw) [exposed coding errors (last 8 byte ordering) and took a bit of time to challenge the algorithm](https://www.reddit.com/r/crypto/comments/9kk5gl/break_my_ciphercollectionpost/eekxw2f/?context=3) and make a few suggestions.

    Following this feedback, the error in the code was fixed, the initialization state was updated and the left-over bytes are now read in order with padding (rather than consumed by the 4th state-word).

* Chris Anderson (@injinj) did amazing work exploring a 128 bit variation and attacking RiskyHash using a variation on a Meet-In-The-Middle attack, written by Hening Makholm (@hmakholm), that discovers hash collisions with a small Hamming distance ([SMHasher fork](https://github.com/hmakholm/smhasher)).

    Following this input, RiskyHash updated the way the message length is incorporated into the final hash and updated the consumption stage to replace the initial XOR with ADD.

After [Risky Hash Version 2](riskyhash_v2) many changes were made. The `LROT` value was reduced (avoiding it becoming an `RROT`). Each consumption vector was allotted it's own prime multiplied. Seed initialization was simplified. Prime numbers were added / updated. The mixing round was simplified.

After [Risky Hash Version 1](riskyhash_v1), the consumption approach was simplified to make RiskyHash easier to implement.

The [previous version can be accessed here](riskyhash_v1).

## Purpose

Risky Hash is designed for fast Hash Map key calculation for both big and small keys. It attempts to act as a 64 bit keyed PRF.

It's possible to compile facil.io with Risk Hash as the default hashing function (the current default is SipHash1-3) by defining the `FIO_USE_RISKY_HASH` during compilation (`-DFIO_USE_RISKY_HASH`).

## Algorithm

A portable (naive) C implementation can be found at the `fio.h` header [and later on in this document](#in-code).

Risky Hash uses 4 reading vectors (state-words), each containing 64 bits.

Risky Hash allows for a 64 bit "seed". that is processed as if it were a 256bit block of data pre-pended to the data being hashed (as if the seed is "copied" to that block 4 times).

Risky Hash uses the following prime numbers:

```txt
FIO_RISKY3_PRIME0 0xCAEF89D1E9A5EB21ULL
FIO_RISKY3_PRIME1 0xAB137439982B86C9ULL
FIO_RISKY3_PRIME2 0xD9FDC73ABE9EDECDULL
FIO_RISKY3_PRIME3 0x3532D520F9511B13ULL
FIO_RISKY3_PRIME4 0x038720DDEB5A8415ULL
```

Risky Hash has three or four stages:

* Initialization stage.

* Seed / Secret reading stage (optional).

* Reading stage.

* Mixing stage.

The hashing algorithm uses an internal 256 bit state for a 64 bit output and the input data is mixed twice into the state on different bit positions (left rotation).

This approach **should** minimize the risk of malicious data weakening the hash function.

The following operations are used:

* `~` marks a bit inversion.
* `+` marks a mod 2^64 addition.
* `XOR` marks an XOR operation.
* `MUL(x,y)` a mod 2^64 multiplication.
* `LROT(x,bits)` is a left rotation of a 64 bit word.
* `>>` is a right shift (not rotate, some bits are lost).
* `<<` is a left shift (not rotate, some bits are lost).

### Initialization

In the initialization stage, Risky Hash attempts to achieves a single goal:

* It must start initialize each reading vector so it is unique and promises (as much as possible).

The four consumption vectors are initialized using a few set bits to promise that they react differently when consuming the same input:

```txt
V1 = 0x0000001000000001
V2 = 0x0000010000000010
V3 = 0x0000100000000100
V4 = 0x0001000000001000
```

### Seed / Secret Reading Stage (optional).

In the seed / secret reading stage, Risky Hash attempts to achieves a single goal:

* Update the hash state using a "secret" (key / salt / seed) in a way that will result in the "secret" having a meaningful impact on the final result.

* Make sure the hash internal state can't be reversed/controlled by a maliciously crafted message.

A seed is a 64bit "word" with at least a single bit set (the secret will be discarded when zero).

When the seed is set, each of the consumption vectors will be multiplied by that seed. Later the last three consumption vectors will be XORed with the seed.

In pseudo code:

```c
if(seed){
    V0 = MUL(V0, seed);
    V1 = MUL(V1, seed);
    V2 = MUL(V2, seed);
    V3 = MUL(V3, seed);
    V1 = V1 XOR seed;
    V2 = V2 XOR seed;
    V3 = V3 XOR seed;
}
```

### Consumption

In the consumption stage, Risky Hash attempts to achieves three goals:

* Consume the data with minimal bias (bits have a fairly even chance at being set or unset).

* Allow parallelism.

   This is achieved by using a number of distinct and separated reading "vectors" (independent state-words).

* Repeated data blocks should produce different results according to their position.

   This is attempted by mixing a number of operations (OX, LROT, addition and multiplication), so the vector is mutated every step (regardless of the input data).

* Maliciously crafted data won't be able to weaken the hash function or expose the "secret".

    This is attempted by reading the data twice into different positions in the consumption vector. This minimizes the possibility of finding a malicious value that could break the state vector.

    However, Risky Hash still could probably be attacked by carefully crafted malicious data that would result in a collision.

Risky Hash consumes data in 64 bit chunks/words.

Each vector reads a single 64 bit word within a 256 bit block, allowing the vectors to be parallelized though any message length of 256 bits or longer.

`V0` reads the first 64 bits, `V0` reads bits 65-128, and so forth...

The 64 bits are read in network byte order (Big-Endian) and treated as a numerical value.

Any trailing data that doesn't fit in a 64 bit word is padded with zeros. The last byte in a padded word (if any) will contain the least significant 8 bits of the length value (so it's never 0). That word is then consumed the next available vector.

Each vector performs the following operations in each of it's consumption rounds (`Vi` is the vector, `word` is the input data for that vector as a 64 bit word):

```txt
Vi = Vi + word
Vi = LROT(Vi, 29)
Vi = Vi + word
Vi = MUL(Vi, Pi)
```

It is normal for some vectors to perform more consumption rounds than others when the data isn't divisible into 256 bit blocks.

### Hash Mixing

In the mixing stage, Risky Hash attempts to achieves three goals:

* Be irreversible.

   This stage is the "last line of defense" against malicious data. For this reason, it should be infeasible to extract meaningful data from the final result.

* Produce a message digest with minimal bias (bits have a fairly even chance at being set or unset).

* Allow all consumption vectors an equal but different effect on the final hash value.

The following intermediate 64 bit result is calculated:

```txt
result = LROT(V1,17) + LROT(V2,13) + LROT(V3,47) + LROT(V4,57)
```

At this point the length of the input data is finalized an can be added to the calculation.

The consumed (unpadded) message length is treated as a 64 bit word. It is shifted left by 33 bits and XORed with itself. Then, the updated length value is added to the intermediate result:

```txt
length = length * 0x0000001000000001
result = result + length
```

The vectors are mixed in with the intermediate result in different positions (using `LROT`):

```txt
  result += v[0] XOR v[1];
  result = result XOR (fio_lrot64(result, 13));
  result += v[1] XOR v[2];
  result = result XOR (fio_lrot64(result, 29));
  result += v[2] XOR v[3];
  result += fio_lrot64(result, 33);
  result += v[3] XOR v[0];
  result = result XOR (fio_lrot64(result, 51));
```

Finally, the intermediate result is mixed with itself to improve bit entropy distribution and hinder reversibility.

```txt
  result = result XOR MUL((result >> 29), P4);
```

## Performance

Risky Hash attempts to balance performance with security concerns, since hash functions are often use by insecure hash table implementations.

However, the design should allow for fairly high performance, for example, by using SIMD instructions or a multi-threaded approach (up to 4 threads).

In fact, even the simple reference implementation at the end of this document offers fairly high performance.

## Attacks, Reports and Security

No known attacks exist for this draft of the Risky Hash algorithm and no known collisions have been found...

...However, it's too early to tell.

At this early stage, please feel free to attack the Risky Hash algorithm and report any security concerns in the [GitHub issue tracker](https://github.com/boazsegev/facil.io/issues).

Later, as Risky Hash usage might increase, attacks should be reported discretely if possible, allowing for a fix to be provided before publication.

## In Code

In C code, the above description might translate like so:

```c
/*
Copyright: Boaz Segev, 2019
License: MIT
*/

/** 64 bit left rotation, inlined. */
#define fio_lrot64(i, bits)                                                    \
  (((uint64_t)(i) << ((bits)&63UL)) | ((uint64_t)(i) >> ((-(bits)) & 63UL)))

/** Converts an unaligned network ordered byte stream to a 64 bit number. */
#define fio_str2u64(c)                                                         \
  ((uint64_t)((((uint64_t)((uint8_t *)(c))[0]) << 56) |                        \
              (((uint64_t)((uint8_t *)(c))[1]) << 48) |                        \
              (((uint64_t)((uint8_t *)(c))[2]) << 40) |                        \
              (((uint64_t)((uint8_t *)(c))[3]) << 32) |                        \
              (((uint64_t)((uint8_t *)(c))[4]) << 24) |                        \
              (((uint64_t)((uint8_t *)(c))[5]) << 16) |                        \
              (((uint64_t)((uint8_t *)(c))[6]) << 8) |                         \
              ((uint64_t)0 + ((uint8_t *)(c))[7])))

/* Risky Hash primes */
#define FIO_RISKY3_PRIME0 0xCAEF89D1E9A5EB21ULL
#define FIO_RISKY3_PRIME1 0xAB137439982B86C9ULL
#define FIO_RISKY3_PRIME2 0xD9FDC73ABE9EDECDULL
#define FIO_RISKY3_PRIME3 0x3532D520F9511B13ULL
#define FIO_RISKY3_PRIME4 0x038720DDEB5A8415ULL
/* Risky Hash initialization constants */
#define FIO_RISKY3_IV0 0x0000001000000001ULL
#define FIO_RISKY3_IV1 0x0000010000000010ULL
#define FIO_RISKY3_IV2 0x0000100000000100ULL
#define FIO_RISKY3_IV3 0x0001000000001000ULL

#ifdef __cplusplus
/* the register keyword was deprecated for C++ but is semi-meaningful in C */
#define register
#endif

/*  Computes a facil.io Risky Hash. */
uint64_t fio_risky_hash(const void *data_, size_t len,
                                         uint64_t seed) {
  const uint64_t primes[] = {
      FIO_RISKY3_PRIME0,
      FIO_RISKY3_PRIME1,
      FIO_RISKY3_PRIME2,
      FIO_RISKY3_PRIME3,
  };
  register uint64_t v[] = {
      FIO_RISKY3_IV0,
      FIO_RISKY3_IV1,
      FIO_RISKY3_IV2,
      FIO_RISKY3_IV3,
  };
  const uint8_t *data = (const uint8_t *)data_;

#define FIO_RISKY3_ROUND64(vi, w)                                              \
  v[vi] += w;                                                                  \
  v[vi] = fio_lrot64(v[vi], 29);                                               \
  v[vi] += w;                                                                  \
  v[vi] *= primes[vi];

#define FIO_RISKY3_ROUND256(w0, w1, w2, w3)                                    \
  FIO_RISKY3_ROUND64(0, w0);                                                   \
  FIO_RISKY3_ROUND64(1, w1);                                                   \
  FIO_RISKY3_ROUND64(2, w2);                                                   \
  FIO_RISKY3_ROUND64(3, w3);

  if (seed) {
    /* process the seed */
    v[0] *= seed;
    v[1] *= seed;
    v[2] *= seed;
    v[3] *= seed;
    v[1] ^= seed;
    v[2] ^= seed;
    v[3] ^= seed;
  }

  for (size_t i = len >> 5; i; --i) {
    /* vectorized 32 bytes / 256 bit access */
    FIO_RISKY3_ROUND256(fio_buf2u64(data), fio_buf2u64(data + 8),
                        fio_buf2u64(data + 16), fio_buf2u64(data + 24));
    data += 32;
  }
  switch (len & 24) {
  case 24:
    FIO_RISKY3_ROUND64(2, fio_buf2u64(data + 16));
    /* fallthrough */
  case 16:
    FIO_RISKY3_ROUND64(1, fio_buf2u64(data + 8));
    /* fallthrough */
  case 8:
    FIO_RISKY3_ROUND64(0, fio_buf2u64(data + 0));
    data += len & 24;
  }

  uint64_t tmp = (len & 0xFF); /* add offset information to padding */
  /* leftover bytes */
  switch ((len & 7)) {
  case 7:
    tmp |= ((uint64_t)data[6]) << 8; /* fallthrough */
  case 6:
    tmp |= ((uint64_t)data[5]) << 16; /* fallthrough */
  case 5:
    tmp |= ((uint64_t)data[4]) << 24; /* fallthrough */
  case 4:
    tmp |= ((uint64_t)data[3]) << 32; /* fallthrough */
  case 3:
    tmp |= ((uint64_t)data[2]) << 40; /* fallthrough */
  case 2:
    tmp |= ((uint64_t)data[1]) << 48; /* fallthrough */
  case 1:
    tmp |= ((uint64_t)data[0]) << 56;
    /* the last (now padded) byte's position */
    switch ((len & 24)) {
    case 24: /* offset 24 in 32 byte segment */
      FIO_RISKY3_ROUND64(3, tmp);
      break;
    case 16: /* offset 16 in 32 byte segment */
      FIO_RISKY3_ROUND64(2, tmp);
      break;
    case 8: /* offset 8 in 32 byte segment */
      FIO_RISKY3_ROUND64(1, tmp);
      break;
    case 0: /* offset 0 in 32 byte segment */
      FIO_RISKY3_ROUND64(0, tmp);
      break;
    }
  }

  /* irreversible avalanche... I think */
  uint64_t r = (len)*0x0000001000000001ULL;
  r += fio_lrot64(v[0], 17) + fio_lrot64(v[1], 13) + fio_lrot64(v[2], 47) +
       fio_lrot64(v[3], 57);
  r += v[0] ^ v[1];
  r ^= fio_lrot64(r, 13);
  r += v[1] ^ v[2];
  r ^= fio_lrot64(r, 29);
  r += v[2] ^ v[3];
  r += fio_lrot64(r, 33);
  r += v[3] ^ v[0];
  r ^= fio_lrot64(r, 51);
  r ^= (r >> 29) * FIO_RISKY3_PRIME4;
  return r;
}
```

## SMHasher results

The following results were produced on a 2.9 GHz Intel Core i9 machine and won't be updated every time.

Note that the Sparse Hash test found a single collision on a 3 bit sparse hash with a 512 bit key:

```txt
Keyset 'Sparse' - 512-bit keys with up to 3 bits set - 22370049 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      1 (36862.59x) !!!!!
```

For some reason this didn't mark the testing as a failure.

**These are the full results**:


```txt
-------------------------------------------------------------------------------
--- Testing RiskyHash64 "Risky Hash 64 bits v.3" GOOD

[[[ Sanity Tests ]]]

Verification value 0xC073C71D ....... PASS
Running sanity check 1     .......... PASS
Running AppendedZeroesTest .......... PASS

[[[ Speed Tests ]]]

Bulk speed test - 262144-byte keys
Alignment  7 -  5.708 bytes/cycle - 16330.69 MiB/sec @ 3 ghz
Alignment  6 -  5.447 bytes/cycle - 15584.30 MiB/sec @ 3 ghz
Alignment  5 -  5.729 bytes/cycle - 16390.67 MiB/sec @ 3 ghz
Alignment  4 -  5.768 bytes/cycle - 16501.35 MiB/sec @ 3 ghz
Alignment  3 -  5.826 bytes/cycle - 16668.09 MiB/sec @ 3 ghz
Alignment  2 -  5.715 bytes/cycle - 16352.08 MiB/sec @ 3 ghz
Alignment  1 -  5.827 bytes/cycle - 16670.62 MiB/sec @ 3 ghz
Alignment  0 -  5.874 bytes/cycle - 16804.73 MiB/sec @ 3 ghz
Average      -  5.737 bytes/cycle - 16412.82 MiB/sec @ 3 ghz

Small key speed test -    1-byte keys -    23.71 cycles/hash
Small key speed test -    2-byte keys -    24.22 cycles/hash
Small key speed test -    3-byte keys -    25.00 cycles/hash
Small key speed test -    4-byte keys -    26.00 cycles/hash
Small key speed test -    5-byte keys -    25.73 cycles/hash
Small key speed test -    6-byte keys -    26.64 cycles/hash
Small key speed test -    7-byte keys -    26.74 cycles/hash
Small key speed test -    8-byte keys -    29.70 cycles/hash
Small key speed test -    9-byte keys -    29.00 cycles/hash
Small key speed test -   10-byte keys -    29.51 cycles/hash
Small key speed test -   11-byte keys -    29.88 cycles/hash
Small key speed test -   12-byte keys -    30.38 cycles/hash
Small key speed test -   13-byte keys -    29.74 cycles/hash
Small key speed test -   14-byte keys -    29.97 cycles/hash
Small key speed test -   15-byte keys -    29.86 cycles/hash
Small key speed test -   16-byte keys -    29.62 cycles/hash
Small key speed test -   17-byte keys -    29.89 cycles/hash
Small key speed test -   18-byte keys -    30.55 cycles/hash
Small key speed test -   19-byte keys -    30.82 cycles/hash
Small key speed test -   20-byte keys -    30.69 cycles/hash
Small key speed test -   21-byte keys -    30.81 cycles/hash
Small key speed test -   22-byte keys -    31.05 cycles/hash
Small key speed test -   23-byte keys -    30.62 cycles/hash
Small key speed test -   24-byte keys -    30.59 cycles/hash
Small key speed test -   25-byte keys -    30.06 cycles/hash
Small key speed test -   26-byte keys -    30.25 cycles/hash
Small key speed test -   27-byte keys -    30.92 cycles/hash
Small key speed test -   28-byte keys -    31.08 cycles/hash
Small key speed test -   29-byte keys -    30.53 cycles/hash
Small key speed test -   30-byte keys -    31.11 cycles/hash
Small key speed test -   31-byte keys -    30.42 cycles/hash
Average                                    29.196 cycles/hash

[[[ Avalanche Tests ]]]

Testing   24-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.582000%
Testing   32-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.710667%
Testing   40-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.700000%
Testing   48-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.749333%
Testing   56-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.661333%
Testing   64-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.690667%
Testing   72-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.618000%
Testing   80-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.720667%
Testing   96-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.756000%
Testing  112-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.770000%
Testing  128-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.746000%
Testing  160-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.700667%
Testing  512-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.803333%
Testing 1024-bit keys ->  64-bit hashes, 300000 reps worst bias is 0.828000%

[[[ Keyset 'Sparse' Tests ]]]

Keyset 'Sparse' - 16-bit keys with up to 9 bits set - 50643 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected          0.6, actual      0 (0.00x)
Testing collisions (high 19-26 bits) - Worst is 22 bits: 300/611 (0.49x)
Testing collisions (high 12-bit) - Expected      50643.0, actual  46547 (0.92x)
Testing collisions (high  8-bit) - Expected      50643.0, actual  50387 (0.99x) (-256)
Testing collisions (low  32-bit) - Expected          0.6, actual      1 (1.67x) (1)
Testing collisions (low  19-26 bits) - Worst is 26 bits: 25/38 (0.65x)
Testing collisions (low  12-bit) - Expected      50643.0, actual  46547 (0.92x)
Testing collisions (low   8-bit) - Expected      50643.0, actual  50387 (0.99x) (-256)
Testing distribution - Worst bias is the 13-bit window at bit 30 - 0.497%

Keyset 'Sparse' - 24-bit keys with up to 8 bits set - 1271626 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected        376.5, actual    198 (0.53x)
Testing collisions (high 24-36 bits) - Worst is 36 bits: 23/23 (0.98x)
Testing collisions (high 12-bit) - Expected    1271626.0, actual 1267530 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    1271626.0, actual 1271370 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected        376.5, actual    182 (0.48x)
Testing collisions (low  24-36 bits) - Worst is 36 bits: 18/23 (0.76x)
Testing collisions (low  12-bit) - Expected    1271626.0, actual 1267530 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    1271626.0, actual 1271370 (1.00x) (-256)
Testing distribution - Worst bias is the 17-bit window at bit  2 - 0.075%

Keyset 'Sparse' - 32-bit keys with up to 7 bits set - 4514873 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       4746.0, actual   2740 (0.58x)
Testing collisions (high 26-39 bits) - Worst is 37 bits: 150/148 (1.01x)
Testing collisions (high 12-bit) - Expected    4514873.0, actual 4510777 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    4514873.0, actual 4514617 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       4746.0, actual   2342 (0.49x)
Testing collisions (low  26-39 bits) - Worst is 35 bits: 298/593 (0.50x)
Testing collisions (low  12-bit) - Expected    4514873.0, actual 4510777 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    4514873.0, actual 4514617 (1.00x) (-256)
Testing distribution - Worst bias is the 19-bit window at bit 44 - 0.047%

Keyset 'Sparse' - 40-bit keys with up to 6 bits set - 4598479 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       4923.4, actual   2677 (0.54x)
Testing collisions (high 26-39 bits) - Worst is 39 bits: 41/38 (1.07x)
Testing collisions (high 12-bit) - Expected    4598479.0, actual 4594383 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    4598479.0, actual 4598223 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       4923.4, actual   2454 (0.50x)
Testing collisions (low  26-39 bits) - Worst is 33 bits: 1242/2461 (0.50x)
Testing collisions (low  12-bit) - Expected    4598479.0, actual 4594383 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    4598479.0, actual 4598223 (1.00x) (-256)
Testing distribution - Worst bias is the 19-bit window at bit 18 - 0.059%

Keyset 'Sparse' - 48-bit keys with up to 6 bits set - 14196869 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      46927.3, actual  26364 (0.56x)
Testing collisions (high 28-43 bits) - Worst is 35 bits: 5819/5865 (0.99x)
Testing collisions (high 12-bit) - Expected   14196869.0, actual 14192773 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   14196869.0, actual 14196613 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      46927.3, actual  23592 (0.50x)
Testing collisions (low  28-43 bits) - Worst is 43 bits: 13/22 (0.57x)
Testing collisions (low  12-bit) - Expected   14196869.0, actual 14192773 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   14196869.0, actual 14196613 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 25 - 0.023%

Keyset 'Sparse' - 56-bit keys with up to 5 bits set - 4216423 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       4139.3, actual   2396 (0.58x)
Testing collisions (high 26-39 bits) - Worst is 35 bits: 546/517 (1.06x)
Testing collisions (high 12-bit) - Expected    4216423.0, actual 4212327 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    4216423.0, actual 4216167 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       4139.3, actual   2068 (0.50x)
Testing collisions (low  26-39 bits) - Worst is 39 bits: 19/32 (0.59x)
Testing collisions (low  12-bit) - Expected    4216423.0, actual 4212327 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    4216423.0, actual 4216167 (1.00x) (-256)
Testing distribution - Worst bias is the 19-bit window at bit 31 - 0.101%

Keyset 'Sparse' - 64-bit keys with up to 5 bits set - 8303633 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16053.7, actual   9036 (0.56x)
Testing collisions (high 27-41 bits) - Worst is 35 bits: 1918/2006 (0.96x)
Testing collisions (high 12-bit) - Expected    8303633.0, actual 8299537 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8303633.0, actual 8303377 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16053.7, actual   7998 (0.50x)
Testing collisions (low  27-41 bits) - Worst is 37 bits: 286/501 (0.57x)
Testing collisions (low  12-bit) - Expected    8303633.0, actual 8299537 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8303633.0, actual 8303377 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 27 - 0.040%

Keyset 'Sparse' - 72-bit keys with up to 5 bits set - 15082603 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      52965.5, actual  29691 (0.56x)
Testing collisions (high 28-43 bits) - Worst is 42 bits: 54/51 (1.04x)
Testing collisions (high 12-bit) - Expected   15082603.0, actual 15078507 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   15082603.0, actual 15082347 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      52965.5, actual  26382 (0.50x)
Testing collisions (low  28-43 bits) - Worst is 43 bits: 18/25 (0.70x)
Testing collisions (low  12-bit) - Expected   15082603.0, actual 15078507 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   15082603.0, actual 15082347 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 37 - 0.023%

Keyset 'Sparse' - 96-bit keys with up to 4 bits set - 3469497 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       2802.7, actual   1578 (0.56x)
Testing collisions (high 26-39 bits) - Worst is 35 bits: 343/350 (0.98x)
Testing collisions (high 12-bit) - Expected    3469497.0, actual 3465401 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    3469497.0, actual 3469241 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       2802.7, actual   1385 (0.49x)
Testing collisions (low  26-39 bits) - Worst is 37 bits: 51/87 (0.58x)
Testing collisions (low  12-bit) - Expected    3469497.0, actual 3465401 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    3469497.0, actual 3469241 (1.00x) (-256)
Testing distribution - Worst bias is the 19-bit window at bit 34 - 0.036%

Keyset 'Sparse' - 160-bit keys with up to 4 bits set - 26977161 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected     169446.5, actual  94980 (0.56x)
Testing collisions (high 29-45 bits) - Worst is 44 bits: 61/41 (1.47x)
Testing collisions (high 12-bit) - Expected   26977161.0, actual 26973065 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   26977161.0, actual 26976905 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected     169446.5, actual  83971 (0.50x)
Testing collisions (low  29-45 bits) - Worst is 43 bits: 50/82 (0.60x)
Testing collisions (low  12-bit) - Expected   26977161.0, actual 26973065 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   26977161.0, actual 26976905 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 60 - 0.011%

Keyset 'Sparse' - 256-bit keys with up to 3 bits set - 2796417 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1820.7, actual   1018 (0.56x)
Testing collisions (high 25-38 bits) - Worst is 38 bits: 36/28 (1.27x)
Testing collisions (high 12-bit) - Expected    2796417.0, actual 2792321 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2796417.0, actual 2796161 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1820.7, actual    895 (0.49x)
Testing collisions (low  25-38 bits) - Worst is 37 bits: 42/56 (0.74x)
Testing collisions (low  12-bit) - Expected    2796417.0, actual 2792321 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2796417.0, actual 2796161 (1.00x) (-256)
Testing distribution - Worst bias is the 19-bit window at bit 19 - 0.130%

Keyset 'Sparse' - 512-bit keys with up to 3 bits set - 22370049 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      1 (36862.59x) !!!!!
Testing collisions (high 32-bit) - Expected     116512.9, actual  64872 (0.56x)
Testing collisions (high 28-44 bits) - Worst is 44 bits: 33/28 (1.16x)
Testing collisions (high 12-bit) - Expected   22370049.0, actual 22365953 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   22370049.0, actual 22369793 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected     116512.9, actual  58528 (0.50x)
Testing collisions (low  28-44 bits) - Worst is 39 bits: 476/910 (0.52x)
Testing collisions (low  12-bit) - Expected   22370049.0, actual 22365953 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   22370049.0, actual 22369793 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit  5 - 0.015%

Keyset 'Sparse' - 1024-bit keys with up to 2 bits set - 524801 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected         64.1, actual     30 (0.47x)
Testing collisions (high 23-33 bits) - Worst is 31 bits: 73/128 (0.57x)
Testing collisions (high 12-bit) - Expected     524801.0, actual 520705 (0.99x) (-4096)
Testing collisions (high  8-bit) - Expected     524801.0, actual 524545 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected         64.1, actual     40 (0.62x)
Testing collisions (low  23-33 bits) - Worst is 33 bits: 25/32 (0.78x)
Testing collisions (low  12-bit) - Expected     524801.0, actual 520705 (0.99x) (-4096)
Testing collisions (low   8-bit) - Expected     524801.0, actual 524545 (1.00x) (-256)
Testing distribution - Worst bias is the 16-bit window at bit 53 - 0.142%

Keyset 'Sparse' - 2048-bit keys with up to 2 bits set - 2098177 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1025.0, actual    563 (0.55x)
Testing collisions (high 25-37 bits) - Worst is 35 bits: 119/128 (0.93x)
Testing collisions (high 12-bit) - Expected    2098177.0, actual 2094081 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2098177.0, actual 2097921 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1025.0, actual    521 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 34/64 (0.53x)
Testing collisions (low  12-bit) - Expected    2098177.0, actual 2094081 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2098177.0, actual 2097921 (1.00x) (-256)
Testing distribution - Worst bias is the 18-bit window at bit 47 - 0.062%

*********FAIL*********

[[[ Keyset 'Permutation' Tests ]]]

Combination Lowbits Tests:
Keyset 'Combination' - up to 7 blocks from a set of 8 - 2396744 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1337.5, actual    782 (0.58x)
Testing collisions (high 25-38 bits) - Worst is 37 bits: 47/41 (1.12x)
Testing collisions (high 12-bit) - Expected    2396744.0, actual 2392648 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2396744.0, actual 2396488 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1337.5, actual    694 (0.52x)
Testing collisions (low  25-38 bits) - Worst is 37 bits: 25/41 (0.60x)
Testing collisions (low  12-bit) - Expected    2396744.0, actual 2392648 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2396744.0, actual 2396488 (1.00x) (-256)
Testing distribution - Worst bias is the 17-bit window at bit 20 - 0.051%


Combination Highbits Tests
Keyset 'Combination' - up to 7 blocks from a set of 8 - 2396744 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1337.5, actual    774 (0.58x)
Testing collisions (high 25-38 bits) - Worst is 38 bits: 23/20 (1.10x)
Testing collisions (high 12-bit) - Expected    2396744.0, actual 2392648 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2396744.0, actual 2396488 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1337.5, actual    672 (0.50x)
Testing collisions (low  25-38 bits) - Worst is 36 bits: 45/83 (0.54x)
Testing collisions (low  12-bit) - Expected    2396744.0, actual 2392648 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2396744.0, actual 2396488 (1.00x) (-256)
Testing distribution - Worst bias is the 18-bit window at bit 39 - 0.055%


Combination Hi-Lo Tests:
Keyset 'Combination' - up to 6 blocks from a set of 15 - 12204240 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      34678.6, actual  19486 (0.56x)
Testing collisions (high 27-42 bits) - Worst is 39 bits: 298/270 (1.10x)
Testing collisions (high 12-bit) - Expected   12204240.0, actual 12200144 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   12204240.0, actual 12203984 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      34678.6, actual  17374 (0.50x)
Testing collisions (low  27-42 bits) - Worst is 37 bits: 612/1083 (0.56x)
Testing collisions (low  12-bit) - Expected   12204240.0, actual 12200144 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   12204240.0, actual 12203984 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 17 - 0.032%


Combination 0x8000000 Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9214 (0.56x)
Testing collisions (high 27-41 bits) - Worst is 40 bits: 79/63 (1.23x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8237 (0.50x)
Testing collisions (low  27-41 bits) - Worst is 41 bits: 24/31 (0.75x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 20 - 0.055%


Combination 0x0000001 Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9110 (0.56x)
Testing collisions (high 27-41 bits) - Worst is 41 bits: 35/31 (1.09x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8234 (0.50x)
Testing collisions (low  27-41 bits) - Worst is 40 bits: 39/63 (0.61x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit  3 - 0.029%


Combination 0x800000000000000 Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9392 (0.57x)
Testing collisions (high 27-41 bits) - Worst is 39 bits: 148/127 (1.16x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8049 (0.49x)
Testing collisions (low  27-41 bits) - Worst is 39 bits: 76/127 (0.59x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 63 - 0.054%


Combination 0x000000000000001 Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9248 (0.56x)
Testing collisions (high 27-41 bits) - Worst is 38 bits: 263/255 (1.03x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8300 (0.51x)
Testing collisions (low  27-41 bits) - Worst is 37 bits: 283/511 (0.55x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 36 - 0.049%


Combination 16-bytes [0-1] Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9193 (0.56x)
Testing collisions (high 27-41 bits) - Worst is 41 bits: 37/31 (1.16x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8114 (0.50x)
Testing collisions (low  27-41 bits) - Worst is 40 bits: 34/63 (0.53x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 23 - 0.036%


Combination 16-bytes [0-last] Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9276 (0.57x)
Testing collisions (high 27-41 bits) - Worst is 40 bits: 70/63 (1.09x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8119 (0.50x)
Testing collisions (low  27-41 bits) - Worst is 38 bits: 130/255 (0.51x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 57 - 0.041%


Combination 32-bytes [0-1] Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9072 (0.55x)
Testing collisions (high 27-41 bits) - Worst is 39 bits: 143/127 (1.12x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8406 (0.51x)
Testing collisions (low  27-41 bits) - Worst is 36 bits: 549/1023 (0.54x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 54 - 0.042%


Combination 32-bytes [0-last] Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9083 (0.55x)
Testing collisions (high 27-41 bits) - Worst is 40 bits: 65/63 (1.02x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8072 (0.49x)
Testing collisions (low  27-41 bits) - Worst is 37 bits: 275/511 (0.54x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit  5 - 0.040%


Combination 64-bytes [0-1] Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9180 (0.56x)
Testing collisions (high 27-41 bits) - Worst is 40 bits: 69/63 (1.08x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8138 (0.50x)
Testing collisions (low  27-41 bits) - Worst is 41 bits: 17/31 (0.53x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 37 - 0.043%


Combination 64-bytes [0-last] Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9182 (0.56x)
Testing collisions (high 27-41 bits) - Worst is 40 bits: 73/63 (1.14x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8106 (0.49x)
Testing collisions (low  27-41 bits) - Worst is 35 bits: 1067/2047 (0.52x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 21 - 0.039%


Combination 128-bytes [0-1] Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9319 (0.57x)
Testing collisions (high 27-41 bits) - Worst is 35 bits: 2116/2047 (1.03x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8033 (0.49x)
Testing collisions (low  27-41 bits) - Worst is 41 bits: 19/31 (0.59x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit  3 - 0.024%


Combination 128-bytes [0-last] Tests:
Keyset 'Combination' - up to 22 blocks from a set of 2 - 8388606 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      16384.0, actual   9125 (0.56x)
Testing collisions (high 27-41 bits) - Worst is 36 bits: 1054/1023 (1.03x)
Testing collisions (high 12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      16384.0, actual   8296 (0.51x)
Testing collisions (low  27-41 bits) - Worst is 41 bits: 21/31 (0.66x)
Testing collisions (low  12-bit) - Expected    8388606.0, actual 8384510 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    8388606.0, actual 8388350 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 36 - 0.041%


[[[ Keyset 'Window' Tests ]]]

Keyset 'Window' - 136-bit key,  20-bit window - 136 tests, 1048576 keys per test
Window at   0 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at   1 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at   2 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at   3 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at   4 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at   5 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at   6 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at   7 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at   8 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at   9 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  10 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  11 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  12 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  13 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  14 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  15 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  16 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  17 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  18 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  19 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  20 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  21 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  22 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  23 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  24 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  25 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  26 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  27 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  28 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  29 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  30 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  31 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  32 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  33 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  34 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  35 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  36 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  37 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  38 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  39 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  40 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  41 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  42 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  43 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  44 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  45 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  46 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  47 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  48 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  49 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  50 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  51 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  52 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  53 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  54 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  55 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  56 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  57 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  58 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  59 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  60 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  61 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  62 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  63 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  64 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  65 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  66 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  67 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  68 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  69 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  70 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  71 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  72 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  73 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  74 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  75 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  76 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  77 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  78 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  79 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  80 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  81 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  82 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  83 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  84 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  85 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  86 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  87 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  88 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  89 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  90 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  91 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  92 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  93 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  94 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  95 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  96 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  97 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  98 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at  99 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 100 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 101 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 102 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 103 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 104 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 105 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 106 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 107 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 108 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 109 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 110 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 111 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 112 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 113 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 114 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 115 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 116 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 117 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 118 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 119 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 120 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 121 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 122 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 123 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 124 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 125 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 126 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 127 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 128 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 129 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 130 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 131 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 132 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 133 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 134 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 135 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Window at 136 - Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)

[[[ Keyset 'Cyclic' Tests ]]]

Keyset 'Cyclic' - 8 cycles of 8 bytes - 1000000 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected        232.8, actual    142 (0.61x)
Testing collisions (high 24-35 bits) - Worst is 35 bits: 36/29 (1.24x)
Testing collisions (high 12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected        232.8, actual    103 (0.44x)
Testing collisions (low  24-35 bits) - Worst is 29 bits: 953/1862 (0.51x)
Testing collisions (low  12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing distribution - Worst bias is the 17-bit window at bit 42 - 0.133%

Keyset 'Cyclic' - 8 cycles of 9 bytes - 1000000 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected        232.8, actual    144 (0.62x)
Testing collisions (high 24-35 bits) - Worst is 35 bits: 31/29 (1.07x)
Testing collisions (high 12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected        232.8, actual     96 (0.41x)
Testing collisions (low  24-35 bits) - Worst is 35 bits: 16/29 (0.55x)
Testing collisions (low  12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing distribution - Worst bias is the 17-bit window at bit 29 - 0.096%

Keyset 'Cyclic' - 8 cycles of 10 bytes - 1000000 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected        232.8, actual    122 (0.52x)
Testing collisions (high 24-35 bits) - Worst is 35 bits: 28/29 (0.96x)
Testing collisions (high 12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected        232.8, actual     98 (0.42x)
Testing collisions (low  24-35 bits) - Worst is 27 bits: 3793/7450 (0.51x)
Testing collisions (low  12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing distribution - Worst bias is the 16-bit window at bit 22 - 0.087%

Keyset 'Cyclic' - 8 cycles of 11 bytes - 1000000 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected        232.8, actual    138 (0.59x)
Testing collisions (high 24-35 bits) - Worst is 35 bits: 23/29 (0.79x)
Testing collisions (high 12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected        232.8, actual    108 (0.46x)
Testing collisions (low  24-35 bits) - Worst is 29 bits: 926/1862 (0.50x)
Testing collisions (low  12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing distribution - Worst bias is the 17-bit window at bit 28 - 0.074%

Keyset 'Cyclic' - 8 cycles of 12 bytes - 1000000 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected        232.8, actual    121 (0.52x)
Testing collisions (high 24-35 bits) - Worst is 35 bits: 21/29 (0.72x)
Testing collisions (high 12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected        232.8, actual    141 (0.61x)
Testing collisions (low  24-35 bits) - Worst is 35 bits: 26/29 (0.89x)
Testing collisions (low  12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing distribution - Worst bias is the 17-bit window at bit 47 - 0.113%

Keyset 'Cyclic' - 8 cycles of 16 bytes - 1000000 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected        232.8, actual    136 (0.58x)
Testing collisions (high 24-35 bits) - Worst is 35 bits: 24/29 (0.82x)
Testing collisions (high 12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected        232.8, actual    139 (0.60x)
Testing collisions (low  24-35 bits) - Worst is 32 bits: 139/232 (0.60x)
Testing collisions (low  12-bit) - Expected    1000000.0, actual 995904 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    1000000.0, actual 999744 (1.00x) (-256)
Testing distribution - Worst bias is the 17-bit window at bit  2 - 0.131%


[[[ Keyset 'TwoBytes' Tests ]]]

Keyset 'TwoBytes' - up-to-4-byte keys, 652545 total keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected         99.1, actual     70 (0.71x)
Testing collisions (high 23-34 bits) - Worst is 34 bits: 22/24 (0.89x)
Testing collisions (high 12-bit) - Expected     652545.0, actual 648449 (0.99x) (-4096)
Testing collisions (high  8-bit) - Expected     652545.0, actual 652289 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected         99.1, actual     50 (0.50x)
Testing collisions (low  23-34 bits) - Worst is 33 bits: 28/49 (0.56x)
Testing collisions (low  12-bit) - Expected     652545.0, actual 648449 (0.99x) (-4096)
Testing collisions (low   8-bit) - Expected     652545.0, actual 652289 (1.00x) (-256)
Testing distribution - Worst bias is the 16-bit window at bit 31 - 0.180%

Keyset 'TwoBytes' - up-to-8-byte keys, 5471025 total keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       6969.1, actual   3827 (0.55x)
Testing collisions (high 26-40 bits) - Worst is 35 bits: 912/871 (1.05x)
Testing collisions (high 12-bit) - Expected    5471025.0, actual 5466929 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    5471025.0, actual 5470769 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       6969.1, actual   3503 (0.50x)
Testing collisions (low  26-40 bits) - Worst is 30 bits: 14018/27876 (0.50x)
Testing collisions (low  12-bit) - Expected    5471025.0, actual 5466929 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    5471025.0, actual 5470769 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 18 - 0.049%

Keyset 'TwoBytes' - up-to-12-byte keys, 18616785 total keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      80695.5, actual  44916 (0.56x)
Testing collisions (high 28-43 bits) - Worst is 43 bits: 41/39 (1.04x)
Testing collisions (high 12-bit) - Expected   18616785.0, actual 18612689 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   18616785.0, actual 18616529 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      80695.5, actual  40675 (0.50x)
Testing collisions (low  28-43 bits) - Worst is 34 bits: 10267/20173 (0.51x)
Testing collisions (low  12-bit) - Expected   18616785.0, actual 18612689 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   18616785.0, actual 18616529 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 36 - 0.017%

Keyset 'TwoBytes' - up-to-16-byte keys, 44251425 total keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected     455926.3, actual 253850 (0.56x)
Testing collisions (high 29-46 bits) - Worst is 44 bits: 126/111 (1.13x)
Testing collisions (high 12-bit) - Expected   44251425.0, actual 44247329 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   44251425.0, actual 44251169 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected     455926.3, actual 227312 (0.50x)
Testing collisions (low  29-46 bits) - Worst is 45 bits: 31/55 (0.56x)
Testing collisions (low  12-bit) - Expected   44251425.0, actual 44247329 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   44251425.0, actual 44251169 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 31 - 0.007%

Keyset 'TwoBytes' - up-to-20-byte keys, 86536545 total keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected    1743569.4, actual 970333 (0.56x)
Testing collisions (high 30-48 bits) - Worst is 44 bits: 450/425 (1.06x)
Testing collisions (high 12-bit) - Expected   86536545.0, actual 86532449 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   86536545.0, actual 86536289 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected    1743569.4, actual 866393 (0.50x)
Testing collisions (low  30-48 bits) - Worst is 47 bits: 35/53 (0.66x)
Testing collisions (low  12-bit) - Expected   86536545.0, actual 86532449 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   86536545.0, actual 86536289 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 41 - 0.005%


[[[ 'MomentChi2' Tests ]]]

Running 1st unseeded MomentChi2 for the low 32bits/step 3 ... 38918759.259075 - 410470.251306
Running 2nd   seeded MomentChi2 for the low 32bits/step 3 ... 38919174.477652 - 410475.518226
KeySeedMomentChi2:  0.21001 PASS

[[[ Keyset 'Text' Tests ]]]

Keyset 'Text' - keys of form "Foo[XXXX]Bar" - 14776336 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      50836.3, actual  28458 (0.56x)
Testing collisions (high 28-43 bits) - Worst is 43 bits: 32/24 (1.29x)
Testing collisions (high 12-bit) - Expected   14776336.0, actual 14772240 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   14776336.0, actual 14776080 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      50836.3, actual  25490 (0.50x)
Testing collisions (low  28-43 bits) - Worst is 43 bits: 15/24 (0.60x)
Testing collisions (low  12-bit) - Expected   14776336.0, actual 14772240 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   14776336.0, actual 14776080 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 27 - 0.021%

Keyset 'Text' - keys of form "FooBar[XXXX]" - 14776336 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      50836.3, actual  28638 (0.56x)
Testing collisions (high 28-43 bits) - Worst is 40 bits: 217/198 (1.09x)
Testing collisions (high 12-bit) - Expected   14776336.0, actual 14772240 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   14776336.0, actual 14776080 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      50836.3, actual  25330 (0.50x)
Testing collisions (low  28-43 bits) - Worst is 37 bits: 801/1588 (0.50x)
Testing collisions (low  12-bit) - Expected   14776336.0, actual 14772240 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   14776336.0, actual 14776080 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 27 - 0.025%

Keyset 'Text' - keys of form "[XXXX]FooBar" - 14776336 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected      50836.3, actual  28535 (0.56x)
Testing collisions (high 28-43 bits) - Worst is 42 bits: 58/49 (1.17x)
Testing collisions (high 12-bit) - Expected   14776336.0, actual 14772240 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected   14776336.0, actual 14776080 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected      50836.3, actual  25367 (0.50x)
Testing collisions (low  28-43 bits) - Worst is 41 bits: 56/99 (0.56x)
Testing collisions (low  12-bit) - Expected   14776336.0, actual 14772240 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected   14776336.0, actual 14776080 (1.00x) (-256)
Testing distribution - Worst bias is the 20-bit window at bit 40 - 0.025%


[[[ Keyset 'Zeroes' Tests ]]]

Keyset 'Zeroes' - 204800 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected          9.8, actual      3 (0.31x)
Testing collisions (high 21-30 bits) - Worst is 26 bits: 319/624 (0.51x)
Testing collisions (high 12-bit) - Expected     204800.0, actual 200704 (0.98x)
Testing collisions (high  8-bit) - Expected     204800.0, actual 204544 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected          9.8, actual      2 (0.20x)
Testing collisions (low  21-30 bits) - Worst is 23 bits: 2469/4999 (0.49x)
Testing collisions (low  12-bit) - Expected     204800.0, actual 200704 (0.98x)
Testing collisions (low   8-bit) - Expected     204800.0, actual 204544 (1.00x) (-256)
Testing distribution - Worst bias is the 15-bit window at bit 56 - 0.289%


[[[ Keyset 'Seed' Tests ]]]

Keyset 'Seed' - 5000000 keys
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       5820.8, actual   3271 (0.56x)
Testing collisions (high 26-40 bits) - Worst is 35 bits: 788/727 (1.08x)
Testing collisions (high 12-bit) - Expected    5000000.0, actual 4995904 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    5000000.0, actual 4999744 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       5820.8, actual   2844 (0.49x)
Testing collisions (low  26-40 bits) - Worst is 39 bits: 27/45 (0.59x)
Testing collisions (low  12-bit) - Expected    5000000.0, actual 4995904 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    5000000.0, actual 4999744 (1.00x) (-256)
Testing distribution - Worst bias is the 19-bit window at bit 58 - 0.030%


[[[ Diff 'Differential' Tests ]]]

Testing 8303632 up-to-5-bit differentials in 64-bit keys -> 64 bit hashes.
1000 reps, 8303632000 total tests, expecting 0.00 random collisions..........
0 total collisions, of which 0 single collisions were ignored

Testing 11017632 up-to-4-bit differentials in 128-bit keys -> 64 bit hashes.
1000 reps, 11017632000 total tests, expecting 0.00 random collisions..........
0 total collisions, of which 0 single collisions were ignored

Testing 2796416 up-to-3-bit differentials in 256-bit keys -> 64 bit hashes.
1000 reps, 2796416000 total tests, expecting 0.00 random collisions..........
0 total collisions, of which 0 single collisions were ignored


[[[ DiffDist 'Differential Distribution' Tests ]]]

Testing bit 0
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    493 (0.48x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 35/63 (0.55x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    491 (0.48x)
Testing collisions (low  25-37 bits) - Worst is 30 bits: 2068/4095 (0.50x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 1
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    486 (0.47x)
Testing collisions (high 25-37 bits) - Worst is 28 bits: 8180/16383 (0.50x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    486 (0.47x)
Testing collisions (low  25-37 bits) - Worst is 29 bits: 4078/8191 (0.50x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 2
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    538 (0.53x)
Testing collisions (high 25-37 bits) - Worst is 35 bits: 76/127 (0.59x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    512 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 30 bits: 2090/4095 (0.51x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 3
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    546 (0.53x)
Testing collisions (high 25-37 bits) - Worst is 33 bits: 281/511 (0.55x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    521 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 24/31 (0.75x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 4
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    486 (0.47x)
Testing collisions (high 25-37 bits) - Worst is 34 bits: 129/255 (0.50x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    516 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 33 bits: 264/511 (0.52x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 5
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    528 (0.52x)
Testing collisions (high 25-37 bits) - Worst is 32 bits: 528/1023 (0.52x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    506 (0.49x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 37/63 (0.58x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 6
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    508 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 20/31 (0.63x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    494 (0.48x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 18/31 (0.56x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 7
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    518 (0.51x)
Testing collisions (high 25-37 bits) - Worst is 31 bits: 1068/2047 (0.52x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    481 (0.47x)
Testing collisions (low  25-37 bits) - Worst is 29 bits: 4068/8191 (0.50x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 8
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    498 (0.49x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 43/63 (0.67x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    525 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 20/31 (0.63x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 9
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    503 (0.49x)
Testing collisions (high 25-37 bits) - Worst is 31 bits: 1025/2047 (0.50x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    493 (0.48x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 41/63 (0.64x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 10
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    508 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 29 bits: 4177/8191 (0.51x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    509 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 33 bits: 275/511 (0.54x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 11
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    506 (0.49x)
Testing collisions (high 25-37 bits) - Worst is 34 bits: 128/255 (0.50x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    505 (0.49x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 19/31 (0.59x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 12
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    480 (0.47x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 38/63 (0.59x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    509 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 35 bits: 76/127 (0.59x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 13
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    527 (0.51x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 21/31 (0.66x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    479 (0.47x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 17/31 (0.53x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 14
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    538 (0.53x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 23/31 (0.72x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    500 (0.49x)
Testing collisions (low  25-37 bits) - Worst is 29 bits: 4109/8191 (0.50x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 15
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    511 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 19/31 (0.59x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    511 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 19/31 (0.59x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 16
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    517 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 19/31 (0.59x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    552 (0.54x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 44/63 (0.69x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 17
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    472 (0.46x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 34/63 (0.53x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    517 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 33 bits: 262/511 (0.51x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 18
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    518 (0.51x)
Testing collisions (high 25-37 bits) - Worst is 29 bits: 4182/8191 (0.51x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    503 (0.49x)
Testing collisions (low  25-37 bits) - Worst is 30 bits: 2070/4095 (0.51x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 19
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    513 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 30 bits: 2084/4095 (0.51x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    518 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 36/63 (0.56x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 20
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    535 (0.52x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 21/31 (0.66x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    523 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 32 bits: 523/1023 (0.51x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 21
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    471 (0.46x)
Testing collisions (high 25-37 bits) - Worst is 28 bits: 8222/16383 (0.50x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    516 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 35 bits: 74/127 (0.58x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 22
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    551 (0.54x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 36/63 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    515 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 35 bits: 89/127 (0.70x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 23
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    498 (0.49x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 17/31 (0.53x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    501 (0.49x)
Testing collisions (low  25-37 bits) - Worst is 31 bits: 1061/2047 (0.52x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 24
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    498 (0.49x)
Testing collisions (high 25-37 bits) - Worst is 33 bits: 270/511 (0.53x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    540 (0.53x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 23/31 (0.72x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 25
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    498 (0.49x)
Testing collisions (high 25-37 bits) - Worst is 34 bits: 136/255 (0.53x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    528 (0.52x)
Testing collisions (low  25-37 bits) - Worst is 32 bits: 528/1023 (0.52x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 26
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    507 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 18/31 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    519 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 41/63 (0.64x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 27
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    531 (0.52x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 21/31 (0.66x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    496 (0.48x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 36/63 (0.56x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 28
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    452 (0.44x)
Testing collisions (high 25-37 bits) - Worst is 27 bits: 16307/32767 (0.50x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    476 (0.46x)
Testing collisions (low  25-37 bits) - Worst is 30 bits: 2048/4095 (0.50x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 29
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    534 (0.52x)
Testing collisions (high 25-37 bits) - Worst is 34 bits: 138/255 (0.54x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    522 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 32 bits: 522/1023 (0.51x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 30
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    545 (0.53x)
Testing collisions (high 25-37 bits) - Worst is 35 bits: 72/127 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    471 (0.46x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 17/31 (0.53x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 31
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    499 (0.49x)
Testing collisions (high 25-37 bits) - Worst is 30 bits: 2128/4095 (0.52x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    518 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 34 bits: 134/255 (0.52x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 32
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    504 (0.49x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 18/31 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    515 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 30 bits: 2110/4095 (0.52x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 33
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    528 (0.52x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 36/63 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    497 (0.49x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 25/31 (0.78x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 34
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    512 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 35 bits: 67/127 (0.52x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    492 (0.48x)
Testing collisions (low  25-37 bits) - Worst is 35 bits: 79/127 (0.62x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 35
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    496 (0.48x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 35/63 (0.55x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    516 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 41/63 (0.64x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 36
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    553 (0.54x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 18/31 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    547 (0.53x)
Testing collisions (low  25-37 bits) - Worst is 34 bits: 160/255 (0.63x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 37
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    550 (0.54x)
Testing collisions (high 25-37 bits) - Worst is 35 bits: 86/127 (0.67x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    514 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 31 bits: 1049/2047 (0.51x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 38
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    539 (0.53x)
Testing collisions (high 25-37 bits) - Worst is 33 bits: 278/511 (0.54x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    506 (0.49x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 34/63 (0.53x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 39
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    476 (0.46x)
Testing collisions (high 25-37 bits) - Worst is 35 bits: 68/127 (0.53x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    481 (0.47x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 17/31 (0.53x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 40
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    517 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 19/31 (0.59x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    521 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 19/31 (0.59x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 41
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    502 (0.49x)
Testing collisions (high 25-37 bits) - Worst is 28 bits: 8285/16383 (0.51x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    517 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 34 bits: 138/255 (0.54x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 42
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    483 (0.47x)
Testing collisions (high 25-37 bits) - Worst is 28 bits: 8248/16383 (0.50x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    533 (0.52x)
Testing collisions (low  25-37 bits) - Worst is 35 bits: 72/127 (0.56x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 43
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    523 (0.51x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 38/63 (0.59x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    511 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 18/31 (0.56x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 44
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    520 (0.51x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 21/31 (0.66x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    543 (0.53x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 39/63 (0.61x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 45
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    472 (0.46x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 21/31 (0.66x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    510 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 34 bits: 139/255 (0.54x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 46
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    538 (0.53x)
Testing collisions (high 25-37 bits) - Worst is 34 bits: 136/255 (0.53x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    529 (0.52x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 23/31 (0.72x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 47
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    513 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 21/31 (0.66x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    503 (0.49x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 24/31 (0.75x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 48
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    511 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 18/31 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    520 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 25/31 (0.78x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 49
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    555 (0.54x)
Testing collisions (high 25-37 bits) - Worst is 35 bits: 71/127 (0.55x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    539 (0.53x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 38/63 (0.59x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 50
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    517 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 42/63 (0.66x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    541 (0.53x)
Testing collisions (low  25-37 bits) - Worst is 32 bits: 541/1023 (0.53x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 51
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    501 (0.49x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 19/31 (0.59x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    493 (0.48x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 42/63 (0.66x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 52
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    485 (0.47x)
Testing collisions (high 25-37 bits) - Worst is 35 bits: 66/127 (0.52x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    495 (0.48x)
Testing collisions (low  25-37 bits) - Worst is 35 bits: 72/127 (0.56x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 53
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    493 (0.48x)
Testing collisions (high 25-37 bits) - Worst is 34 bits: 144/255 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    520 (0.51x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 19/31 (0.59x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 54
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    564 (0.55x)
Testing collisions (high 25-37 bits) - Worst is 32 bits: 564/1023 (0.55x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    515 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 34 bits: 137/255 (0.54x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 55
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    523 (0.51x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 44/63 (0.69x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    507 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 31 bits: 1015/2047 (0.50x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 56
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    510 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 36/63 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    528 (0.52x)
Testing collisions (low  25-37 bits) - Worst is 34 bits: 140/255 (0.55x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 57
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    489 (0.48x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 18/31 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    545 (0.53x)
Testing collisions (low  25-37 bits) - Worst is 36 bits: 38/63 (0.59x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 58
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    474 (0.46x)
Testing collisions (high 25-37 bits) - Worst is 27 bits: 16002/32767 (0.49x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    506 (0.49x)
Testing collisions (low  25-37 bits) - Worst is 33 bits: 259/511 (0.51x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 59
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    513 (0.50x)
Testing collisions (high 25-37 bits) - Worst is 31 bits: 1071/2047 (0.52x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    484 (0.47x)
Testing collisions (low  25-37 bits) - Worst is 37 bits: 24/31 (0.75x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 60
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    524 (0.51x)
Testing collisions (high 25-37 bits) - Worst is 35 bits: 79/127 (0.62x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    545 (0.53x)
Testing collisions (low  25-37 bits) - Worst is 32 bits: 545/1023 (0.53x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 61
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    526 (0.51x)
Testing collisions (high 25-37 bits) - Worst is 33 bits: 278/511 (0.54x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    528 (0.52x)
Testing collisions (low  25-37 bits) - Worst is 33 bits: 272/511 (0.53x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 62
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    522 (0.51x)
Testing collisions (high 25-37 bits) - Worst is 37 bits: 19/31 (0.59x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    587 (0.57x)
Testing collisions (low  25-37 bits) - Worst is 33 bits: 298/511 (0.58x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)

Testing bit 63
Testing collisions ( 64-bit)     - Expected          0.0, actual      0 (0.00x)
Testing collisions (high 32-bit) - Expected       1024.0, actual    537 (0.52x)
Testing collisions (high 25-37 bits) - Worst is 36 bits: 36/63 (0.56x)
Testing collisions (high 12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (high  8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)
Testing collisions (low  32-bit) - Expected       1024.0, actual    512 (0.50x)
Testing collisions (low  25-37 bits) - Worst is 33 bits: 265/511 (0.52x)
Testing collisions (low  12-bit) - Expected    2097152.0, actual 2093056 (1.00x) (-4096)
Testing collisions (low   8-bit) - Expected    2097152.0, actual 2096896 (1.00x) (-256)



Input vcode 0x00000001, Output vcode 0x00000001, Result vcode 0x00000001
Verification value is 0x00000001 - Testing took 906.909590 seconds
-------------------------------------------------------------------------------
```