---
title: Some of my HSM-signed transactions were rejected
lang: en
date: 2026-11-25 22:00:00 +0900
---

I was building a signing path for Ethereum. My code builds the transaction, an HSM signs it, and my code sends it to a hosted node. The private key never leaves the HSM.

The transaction was rejected.

## my first guess: I got the math wrong

I assumed I had applied the signing calculation wrong somewhere in my code, so I went back through it. While I was checking, I kept running the same code again.

Some of the runs went through.

That changed the question. It was the same code and the same key, and it only failed some of the time. If my calculation had been wrong, it should have failed every time. So the problem had to be in something that is different on every run, and the signature is different on every run.

## two valid signatures

An ECDSA signature is a pair `(r, s)`. If `(r, s)` is valid for a message and a key, then `(r, n - s)` is valid for the same message and key, where `n` is the order of the curve. Every signature has this twin.

Ethereum stopped accepting one of the two in the Homestead fork. [EIP-2](https://eips.ethereum.org/EIPS/eip-2) says that transaction signatures whose `s` is greater than `secp256k1n/2` are invalid. The reason it gives is malleability: anyone could take a transaction, replace `s` with `secp256k1n - s`, flip `v`, and the result would still be a valid signature for a transaction with a different hash.

Bitcoin has the same rule under the name LOW_S ([BIP 146](https://github.com/bitcoin/bips/blob/master/bip-0146.mediawiki)).

## why it only happens with an HSM

With a software key you normally never see this, because the signing library fixes it for you. libsecp256k1, for example, checks whether `s` is high right after computing it, negates it if so, and flips the recovery id ([ecdsa_impl.h](https://github.com/bitcoin-core/secp256k1/blob/master/src/ecdsa_impl.h)).

An HSM does not do that. PKCS#11 defines an ECDSA signature as the concatenation of `r` and `s`, both the same length, most significant byte first ([PKCS #11 Current Mechanisms v2.40, section 2.3.1](https://docs.oasis-open.org/pkcs11/pkcs11-curr/v2.40/os/pkcs11-curr-v2.40-os.html)). Nothing in it says which half `s` should be in. So some of the signatures that come back have a high `s`. When mine came back low, the transaction went through. When it came back high, it was rejected. That was the whole difference between my good runs and my bad runs.

## what the rejection looks like

The message depends on who catches it. To see it, I signed the same transaction twice with a throwaway key that holds no funds, once with low `s` and once with the high-`s` twin, and sent both to a public Sepolia node running go-ethereum.

With low `s`, the node answered that the account has insufficient funds, which means the signature was accepted. With high `s`, it answered `invalid sender: invalid transaction v, r, s values`.

Libraries reject it before it is sent, if you let them see the signature:

| Library | Message |
|---|---|
| @ethereumjs/tx | `Invalid Signature: s-values greater than secp256k1n/2 are considered invalid` |
| ethers v6, reading `.s` of a signature | `non-canonical s; use ._s` |

In go-ethereum the check is one line in `ValidateSignatureValues`: with Homestead rules on, it returns false if `s` is greater than half the curve order ([crypto.go](https://github.com/ethereum/go-ethereum/blob/master/crypto/crypto.go)). The sender cannot be recovered from the signature, so the pool reports an invalid sender ([validation.go](https://github.com/ethereum/go-ethereum/blob/master/core/txpool/validation.go)).

None of these says "your `s` is too high" in a way I would have recognised at the time.

## the fix

```ts
const N = 0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141n;
const HALF_N = N / 2n;

function toLowS(s: bigint): bigint {
  return s > HALF_N ? N - s : s;
}
```

`N` is the order of secp256k1. The comparison is `s > N/2` with integer division, the same test go-ethereum uses.

After this, the rejections caused by `s` stopped.

## what I had not fixed yet

I was not done. A signed Ethereum transaction also carries a recovery bit (`v` in legacy transactions, `signature_y_parity` since [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)), and PKCS#11 does not return one. I got that wrong several times too. Replacing `s` with `n - s` flips it, which is what EIP-2 means by flipping `v`. That is a separate problem, so it has [its own post]({% post_url 2026-11-25-the-one-bit-my-hsm-did-not-give-me %}).

## what this post does not cover

- I have only done this on Ethereum.
- The node response above is what I got in November 2026. Messages can change between versions and providers.
- Source links point to the main branch.
