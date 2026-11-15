---
title: Why a signature needs low-s
lang: en
---

Most of my work is signing paths where the private key never leaves an HSM. In one of them my own code does everything around the key: it builds the Ethereum transaction, sends a hash to the HSM, and broadcasts what comes back. Nothing sits between the HSM and the chain in that path, and that is where low-s became my problem. An HSM returns a perfectly valid signature that the chain rejects about half the time.

## Two valid signatures for one message

An ECDSA signature is a pair `(r, s)`. For any valid pair, `(r, n - s)` is also valid for the same message and the same key, where `n` is the order of the curve. Nothing is broken here. It follows directly from how verification works.

For a blockchain this is a problem, because the signature is part of the transaction. If both forms are accepted, anyone can take a signed transaction, replace `s` with `n - s`, and get a transaction that does the same thing under a different hash. Bitcoin and Ethereum both closed this by accepting only the lower half: Ethereum has required `s <= n/2` since EIP-2.

## What the HSM gives you

The HSM signs a 32-byte digest, so I RLP-encode the transaction, hash it with Keccak-256 outside the HSM, and send only the digest for raw ECDSA signing. Through PKCS#11 the signature comes back as `r` followed by `s`. The HSM makes no promise about which half `s` falls in, so roughly one signature in two has a high `s`. The HSM is right and the chain is right. The code in between has to reconcile them.

```ts
const N = 0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141n;
const HALF_N = N >> 1n;

function normalize(r: bigint, s: bigint) {
  return s > HALF_N ? { r, s: N - s, flipped: true } : { r, s, flipped: false };
}
```

## The part that is easy to miss

An Ethereum signature also carries `yParity`, one bit that lets the node recover the public key from the signature. Replacing `s` with `n - s` flips that bit. PKCS#11 does not return it at all, so I compute it myself: recover the address for both values and keep the one that matches the signer. I do that after normalizing `s`, not before.

It is only a few lines of code. But in that path nobody else was going to write them, and a transaction that fails half the time is not something you want to discover in front of a customer.
