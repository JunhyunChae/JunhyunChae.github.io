---
title: The one bit my HSM did not give me
lang: en
date: 2026-11-25 23:00:00 +0900
---

I was building a signing path for Ethereum. My code builds the transaction, an HSM signs its hash, and my code sends it to a hosted node. The private key never leaves the HSM.

After I [fixed the high `s` values]({% post_url 2026-11-25-some-hsm-signed-transactions-were-rejected %}), there was one more field I had to fill in myself: the recovery bit. I got it wrong several times.

## what the bit is

An ECDSA signature is a pair `(r, s)`. During signing, the signer computes a point `R` on the curve, and `r` comes from its x-coordinate. For one x-coordinate there are two points on the curve, one with an even y and one with an odd y. To get the public key back from a signature, you need to know which of the two it was. That is the recovery bit.

[EIP-2930](https://eips.ethereum.org/EIPS/eip-2930) defines it as "the parity (0 for even, 1 for odd) of the y-value of a secp256k1 signature." [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559) transactions carry it as `signature_y_parity`. Legacy transactions put it inside `v`: 27 or 28 originally, and `{0,1} + CHAIN_ID * 2 + 35` with [EIP-155](https://eips.ethereum.org/EIPS/eip-155).

This bit matters more than it looks. An Ethereum transaction has no sender field. The EIP-1559 payload is `rlp([chain_id, nonce, ..., signature_y_parity, signature_r, signature_s])`, and there is no `from` in it. The node recovers the public key from the signature and derives the sender's address from that. So the recovery bit decides who sent the transaction.

## why the HSM does not return it

A software signing library knows `R`, because it computed it. libsecp256k1 sets the bit right there ([ecdsa_impl.h](https://github.com/bitcoin-core/secp256k1/blob/master/src/ecdsa_impl.h)):

```c
*recid = (overflow << 1) | secp256k1_fe_is_odd(&r.y);
```

Through PKCS#11, an ECDSA signature is just `r` and `s` concatenated ([PKCS #11 Current Mechanisms v2.40, section 2.3.1](https://docs.oasis-open.org/pkcs11/pkcs11-curr/v2.40/os/pkcs11-curr-v2.40-os.html)). The x-coordinate of `R` comes out as `r`. Its y-coordinate stays inside the HSM. The bit has to be worked out afterwards, outside the HSM.

## what a wrong bit looks like

A wrong bit is not rejected as a bad signature.

Flipping the bit picks the other point. That point is just as valid, so the signature still recovers a public key. It is simply a different key. I checked this with a throwaway key: one signed transaction, two versions that differ only in the bit.

```
signer          : 0xb4d4A07A2473304C958f90Ee9A213D862aa926f0  yParity = 0
from (right bit): 0xb4d4A07A2473304C958f90Ee9A213D862aa926f0
from (wrong bit): 0x1a3306A0d91a8499bA7273C5e865c83365011A7C
```

With the wrong bit, the transaction is "from" an address that has nothing to do with my key.

Then I sent both versions to two public nodes, in November 2026. My throwaway key has no money, so the right one fails too. What matters is how the wrong one fails.

| node | right bit | wrong bit |
|---|---|---|
| Geth v1.17.9 (Sepolia) | `insufficient funds for gas * price + value: balance 0, tx cost 630000000000000, overshot 630000000000000` | the same message |
| reth v2.11.0 (mainnet) | `Insufficient funds for gas * price + value: have 0 want 630000000000000` | the same message |

Neither node says anything about the signature. As far as the node knows, the signature is fine and the sender has no money.

The go-ethereum code shows why. The transaction pool takes the sender from the signature, then looks up that sender's nonce and balance ([validation.go](https://github.com/ethereum/go-ethereum/blob/master/core/txpool/validation.go)). With a wrong bit, the balance it looks up belongs to the other address. Reading that code, I would expect a funded key to get the same "insufficient funds" answer when the bit is wrong, which points at balances and gas instead of the signature. I have not run that case.

## how I compute it now

The HSM does not give me the y-coordinate, so I do not try to compute the parity directly. I try both values, recover the address that each one gives, and keep the one that matches the address of the HSM key. Written with ethers v6:

```ts
import { recoverAddress } from 'ethers';

function findYParity(digest: string, r: string, s: string, expected: string): 0 | 1 {
  for (const yParity of [0, 1] as const) {
    if (recoverAddress(digest, { r, s, yParity }).toLowerCase() === expected.toLowerCase()) {
      return yParity;
    }
  }
  throw new Error('neither recovery bit matches the HSM key');
}
```

`expected` is the address of the HSM key. If neither value matches, the function throws. In that case the problem is not the bit. The digest or the key is wrong.

The order matters. I run this after normalizing `s`, not before. Replacing `s` with `n - s` flips the bit, which is why libsecp256k1 does `*recid ^= high;` right after it negates a high `s`. If you pick the bit first and normalize `s` afterwards, you are back to a wrong bit, but only on the runs where `s` happened to be high.

I checked that with a software key. I signed 2,000 random digests with low-s normalization turned off (`lowS: false` in @noble/curves), so the signatures came back the way an HSM returns them. 991 of them had a high `s`. The bit picked before normalizing was wrong for exactly those 991 and right for the other 1,009.

## what this post does not cover

- I only tested EIP-1559 transactions. The legacy `v` values above are from the EIP-155 text, not from my own tests.
- I have not sent a wrong-bit transaction from a funded key.
- Recovery ids 2 and 3, the `overflow` part of the libsecp256k1 line, are not handled. The libsecp256k1 source notes that this condition is "cryptographically unreachable".
- The node responses are what I got in November 2026. They can change between versions and providers.
- Source links point to the main branch.
