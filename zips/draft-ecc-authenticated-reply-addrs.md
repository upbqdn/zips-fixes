    ZIP: ???
    Title: Authenticated Reply Addresses
    Owners: Jack Grigg <thestr4d@gmail.com>
            Kris Nuttycombe <kris@nutty.land>
            Daira-Emma Hopwood <daira@jacaranda.org>
    Status: Draft
    Category: Standards / Wallet
    Created: 2023-11-12
    License: MIT
    Discussions-To: <https://github.com/zcash/zips/issues/1230>
    Pull-Request: <https://github.com/zcash/zips/pull/1061>


# Terminology

The key words "MUST" and "MUST NOT" in this document are to be interpreted as described in
BCP 14 [^BCP14] when, and only when, they appear in all capitals.


# Abstract

TODO


# Motivation

TODO


# Specification

## Authenticated reply address encoding

- Versioning.
    - TODO: decide how an authenticated reply address is framed in a memo. ZIP 302
      [^zip-0302] defines no type-length-value structure. A type could be assigned by a
      structured-memo revision of ZIP 302, or a revision of ZIP 302 could assign a lead
      byte from the ranges it reserves for future updates (`0xF6` followed by bytes that
      are not all zero, or `0xF7` to `0xFE`). The lead byte `0xFF` is not suitable,
      because ZIP 302 designates it for arbitrary data about which readers make no
      assumptions.
- The reply address.
    - This uses a binary encoding of a ZIP 316 [^zip-0316] Unified Address.
        - TODO: decide on specifics - https://github.com/zcash/librustzcash/pull/711#issuecomment-1377783264
    - This might be the same address for all inputs, or it might only cover a subset of them
      (in a collaborative multi-sender transaction).
- A non-empty list of tuples:
    - $\mathsf{receiverType}$: The receiver type for which this proof is made. This receiver
      type MUST match one of the receiver types in the reply address.
    - $\mathsf{addrProof}$: An address proof for that receiver type.

TODO: specify the encoding of each field, including the number of tuples, and the widths of
$\mathsf{receiverType}$ and of the fields of each address proof.

TODO: when an address proof for Orchard receivers is defined (building on an Orchard
extension of ZIP 304 [^zip-0304]), it must say whether a spent Action in either the Orchard
or the Ironwood pool may back an Orchard receiver, and how its index identifies
$\mathtt{vActionsOrchard}$ or $\mathtt{vActionsIronwood}$ [^zip-0229].

## Creating

## Verifying

- Verify that the transaction is valid (in particular, that all proofs and signatures are valid).
- Decrypt the transaction output to obtain the memo field.
    - TODO: specify which outputs are decrypted, and how a memo that contains an
      authenticated reply address is recognized.
- Decode the authenticated reply address.
    - This MUST validate the inner encodings of e.g. ZIP 316 for UAs (as applicable).
- Verify each address proof.

## Sapling address proof

### Encoding

- $\mathsf{index}$: The index of a Sapling Spend description in the $\mathtt{vSpendsSapling}$
  field of the transaction [^zip-0225] [^zip-0229].
- $\mathsf{nullifier_{addr}}$: A nullifier for a ZIP 304 fake note. [^zip-0304]
- $\mathsf{zkproof_{addr}}$: A Sapling spend proof.

### Proof-of-address functions

The following functions perform the proof steps of the ZIP 304 signature and verification
algorithms [^zip-0304-signature-algorithm] [^zip-0304-verification-algorithm], without the
`spendAuthSig` steps, and with $\alpha$ supplied by the caller instead of selected at random.

TODO: specify these functions in ZIP 304, so that ZIP 311 [^zip-0311] can also use them.

$\mathsf{Zip304CreateProof}((\mathsf{d}, \mathsf{pk_d}), (\mathsf{ask}, \mathsf{nsk}, \mathsf{ovk}), \alpha)$,
for a payment address $(\mathsf{d}, \mathsf{pk_d})$ and its expanded spending key
$(\mathsf{ask}, \mathsf{nsk}, \mathsf{ovk})$:

- Compute $\mathsf{ak}$, $\mathsf{g_d}$, $\mathsf{cm}$, $\mathsf{rt}$, $path$, $\mathsf{cv}$ and
  $\mathsf{nf}$ as in the ZIP 304 signature algorithm.
- Let $\mathsf{rk} = \mathsf{SpendAuthSig.RandomizePublic}(\alpha, \mathsf{ak})$.
- Let $zkproof$ be the byte sequence representation of a Sapling spend proof with primary
  input $(\mathsf{rt}, \mathsf{cv}, \mathsf{nf}, \mathsf{rk})$ and auxiliary input
  $(path, 0, \mathsf{g_d}, \mathsf{pk_d}, 1, 0, \mathsf{cm}, 0, \alpha, \mathsf{ak}, \mathsf{nsk})$.
- Return $(\mathsf{nf}, zkproof)$.

$\mathsf{Zip304VerifyProof}((\mathsf{d}, \mathsf{pk_d}), \mathsf{nf}, \mathsf{rk}, zkproof)$:

- Let $\mathsf{g_d} = \mathsf{DiversifyHash}(\mathsf{d})$. If $\mathsf{g_d} = \bot$, return false.
- Compute $\mathsf{cm}$, $\mathsf{rt}$ and $\mathsf{cv}$ as in the ZIP 304 verification
  algorithm.
- Decode and verify $zkproof$ as a Sapling spend proof with primary input
  $(\mathsf{rt}, \mathsf{cv}, \mathsf{nf}, \mathsf{rk})$. If verification fails, return false.
- Return true.

### Creating

Taking as input:

- The Sapling expanded spending key $(\mathsf{ask}, \mathsf{nsk}, \mathsf{ovk})$.
- $\alpha$: The randomness used to construct the Spend description.
- $\mathsf{index}$: The index of that Spend description in $\mathtt{vSpendsSapling}$.

The Sapling address proof is created as follows:

- Extract the Sapling receiver $(\mathsf{d}, \mathsf{pk_d})$ from the reply address.
- Compute $(\mathsf{nullifier_{addr}}, \mathsf{zkproof_{addr}}) = \mathsf{Zip304CreateProof}((\mathsf{d}, \mathsf{pk_d}), (\mathsf{ask}, \mathsf{nsk}, \mathsf{ovk}), \alpha)$.
- Return $(\mathsf{index}, \mathsf{nullifier_{addr}}, \mathsf{zkproof_{addr}})$.

### Verifying

- Extract the Sapling receiver $(\mathsf{d}, \mathsf{pk_d})$ from the reply address.
- If $\mathsf{index} \geq \mathtt{nSpendsSapling}$, return an error.
- Look up the Spend description at $\mathtt{vSpendsSapling}[\mathsf{index}]$.
- Extract $\mathsf{rk}$ from the Spend description.
- Call $\mathsf{Zip304VerifyProof}((\mathsf{d}, \mathsf{pk_d}), \mathsf{nullifier_{addr}}, \mathsf{rk}, \mathsf{zkproof_{addr}})$, and return an error if it returns false.

# Rationale

We do not generate and include a full ZIP 304 signature in the Sapling address proof; we
only rely on its proof-of-address steps, as $\mathsf{Zip304CreateProof}$ and
$\mathsf{Zip304VerifyProof}$. This is for two reasons:

- The ZIP 304 `spendAuthSig` proves knowledge of the spend authorizing key, but we already
  obtain that from the `spendAuthSig` on the Sapling spend description, so there is no
  additional security benefit from including it, and omitting it decreases the encoding
  size.
- We do not need to encode `rk` because it is already present in the transaction.

The individual address proofs commit to the entire reply address. For example, if the
reply address contains a Sapling and Orchard receiver, but the transaction only spends
Sapling notes, the sender is still confirming that the Orchard receiver is also an allowed
reply address receiver.

However, to have proper cross-linking, we need to ensure that all receivers in the UA have
proofs of spend authority, to prevent a sender from including a receiver that they do not
control (as a form of impersonation attack). There are a few ways this could be resolved:

- Always require the transaction to spend notes for all receivers.
    - Pro: This could be done with dummy notes for the receiver types that the sender
      doesn't want to spend from.
    - Con: This would prevent the sender from pre-emptively adding receivers for upcoming
      shielded pools that are not yet activated.
    - Con: For UAs with transparent receivers, this would be incompatible with shielded-only
      transactions.
    - TODO: for an Orchard receiver, decide whether a spend from either the Orchard or the
      Ironwood pool satisfies this requirement.
- Have an optional proof of spend authority in the address proof.
    - This is omitted when a spent note is present for that receiver type (as the spent note
      serves this purpose).
    - This is pretty much the exact opposite of ZIP 311 (where we require proofs of spend
      authority, and have optional proof-of-address).

In any case, address proofs for Orchard receivers exceed a single 512-byte memo field (a
one-Action Halo 2 proof is 4992 bytes [^protocol-txnencoding]). This requires the memo
bundles of ZIP 231 [^zip-0231], which is not deployed in NU7.


# Security and Privacy Considerations

The verification process for authenticated reply addresses requires that the full
transaction is validated, because it outsources proof-of-spend-authority to the spend
proofs and signatures. As a consequence, wallets that scan transactions via a light client
protocol MUST NOT show the reply address as authenticated until the full transaction has
been downloaded and validated.

# Reference implementation

TBD


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol-txnencoding]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 7.1: Transaction Encoding and Consensus](protocol/protocol.pdf#txnencoding)

[^zip-0225]: [ZIP 225: Version 5 Transaction Format](zip-0225.rst)

[^zip-0229]: [ZIP 229: Version 6 Transaction Format](zip-0229.md)

[^zip-0231]: [ZIP 231: Memo Bundles](zip-0231.md)

[^zip-0302]: [ZIP 302: Standardized Memo Field Format](zip-0302.rst)

[^zip-0304]: [ZIP 304: Sapling Address Signatures](zip-0304.rst)

[^zip-0304-signature-algorithm]: [ZIP 304: Sapling Address Signatures — Signature algorithm](zip-0304.rst#signature-algorithm)

[^zip-0304-verification-algorithm]: [ZIP 304: Sapling Address Signatures — Verification algorithm](zip-0304.rst#verification-algorithm)

[^zip-0311]: [ZIP 311: Zcash Payment Disclosures](zip-0311.rst)

[^zip-0316]: [ZIP 316: Unified Addresses and Unified Viewing Keys](zip-0316.rst)
