    ZIP: unassigned
    Title: Air drops, Proof-of-Balance, and Stake-weighted Polling
    Owners: Daira-Emma Hopwood <daira@jacaranda.org>
            Jack Grigg <thestr4d@gmail.com>
    Status: Draft
    Category: Informational
    Created: 2023-12-07
    License: MIT
    Discussions-To: <https://github.com/zcash/zips/issues/1229>


# Terminology

The key word "MUST" in this document is to be interpreted as described in
BCP 14 [^BCP14] when, and only when, it appears in all capitals.

"Pool snapshot" refers to a snapshot of the state of balances in a shielded pool
as of the end of a specified block.

"Claim" refers to a proof by a ZEC holder (TODO: support ZSAs) that they held a
set of notes summing to a committed value at the time of a given pool snapshot.


# Abstract

This ZIP specifies a mechanism that effectively takes a snapshot of the state of
shielded balances in the Orchard pool and, from NU6.3, the Ironwood pool
[^zip-0229], and allows holders to prove that they had
at least a given balance as of the snapshot, in such a way that they cannot claim
the same balance more than once. The privacy of holders is retained, in the sense
that claims cannot be linked to their past or future spends.

Possible applications include private air drops, private proof-of-balance, and
private stake-weighted polling.


# Motivation

TODO: Explain why this can't be done more simply, and how the problem is
isomorphic to air-drops and stake-weighted polling.

> Do we need to be able to construct a note in another token corresponding to the
> claimed value? Can that be done just with the existing ZSA spec + a public value
> commitment?


# Requirements

Who is eligible to vote / claim an airdrop?
* Most likely approach: snapshot the Zcash chain at some height. Eligible notes
  exist in the commitment tree at that height, but don’t exist in the nullifier
  set at that height.

Does the Zcash side need to prove spend authority?
* Yes, it does (otherwise if people gave out their viewing keys to others, those
  others could vote / claim the airdrop instead).
* UX effect: anyone who moved their funds to a different spend authority after
  the snapshot and then lost their old spending keys become unable to vote /
  claim the airdrop.


# Specification

Sketch:

A "nullifier non-membership tree" is a Merkle tree of sorted disjoint
(start, end) pairs representing the gaps between revealed nullifiers at a pool
snapshot. That is, the union of the regions start..=end is exactly the
complement of the set of revealed nullifiers at the snapshot.

The tree has depth $\mathsf{MerkleDepth^{excl}} := 31$. Its leaf at position $i$
is $\mathsf{MerkleCRH^{Orchard}}(\mathsf{MerkleDepth^{excl}}, \mathsf{start}_i, \mathsf{end}_i)$,
where $(\mathsf{start}_i, \mathsf{end}_i)$ is gap number $i$ in ascending order,
counting from $0$. Unused leaves have the value $\mathsf{Uncommitted^{Orchard}} = 2$,
which is not an output of $\mathsf{MerkleCRH^{Orchard}}$. Internal nodes are
computed as in § 4.9 'Merkle Path Validity' [^protocol-merklepath], using
$\mathsf{MerkleCRH^{Orchard}}$ with $\mathsf{MerkleDepth^{excl}}$ in place of
$\mathsf{MerkleDepth}$. Leaves and internal nodes are therefore hashed with
distinct layer arguments, all in the domain
$\{ 0 .. \mathsf{MerkleDepth^{Orchard}}-1 \}$ of $\mathsf{MerkleCRH^{Orchard}}$.

The tree holds at most $2^{31}$ gaps, so it can represent a pool snapshot with at
most $2^{31}-1$ revealed nullifiers. (A note commitment tree holds up to
$2^{\mathsf{MerkleDepth^{Orchard}}}$ notes, so a full pool would need depth $33$,
which would in turn need a leaf hash other than $\mathsf{MerkleCRH^{Orchard}}$.)

From NU6.3 the Orchard protocol has two pools, the Orchard pool and the Ironwood
pool, each with its own note commitment tree, anchor, and nullifier set
[^zip-0229]. A pool snapshot, and its nullifier non-membership tree, is of one of
these pools. The Note Claim statement below applies unchanged to a note in either
pool, with $\mathsf{rt^{cm}}$ and $\mathsf{rt^{excl}}$ taken from the pool snapshot
of the pool that holds the note.

An "alternate nullifier" is a value derived from a note, with similar
cryptographic properties to its standard nullifier, but including a "nullifier
domain" as input to the derivation. It is distinct from and unlinkable with the
standard nullifier, or any of the other alternate nullifiers for that note in
other nullifier domains. A note has exactly one alternate nullifier for each
nullifier domain.

An entity that wants to conduct an air-drop, stake-weighted poll, etc. does the
following:

* Choose a pool snapshot at a particular block height for each pool to be
  included. All of these pool snapshots MUST be as of the end of the same block;
  otherwise a holder could claim a note in the Orchard pool, move its value to the
  Ironwood pool before the later snapshot, and claim the same value again under a
  different alternate nullifier.
* Choose a previously unused nullifier domain $\mathsf{dom}$.
    * > TODO: how to ensure it is previously unused? What are the consequences if it isn't?
    * Maybe the domain is derived from the pool snapshot and a string identifying
      the air-drop/poll? Then a wallet that supports this protocol can display the
      string and the block height/date of the snapshot to the wallet user.
* Deterministically construct a nullifier non-membership tree as of each snapshot,
  with root $\mathsf{rt^{excl}}$. Anyone can check that this root is correct using
  public information.
* Publish $(\mathsf{rt^{cm}}, \mathsf{rt^{excl}})$ for each pool snapshot, where
  $\mathsf{rt^{cm}}$ is the root of that pool's note commitment tree as of the
  snapshot.
* Keep track of a set of alternate nullifiers revealed by the statement below and
  make sure that they don't repeat.

To participate, a holder proves the following informal statement:

"Given:
* a value commitment $\mathsf{cv}$;
* a nullifier domain $\mathsf{dom}$;
* an alternate nullifier $\mathsf{nf_{dom}}$;
* a pool snapshot $(\mathsf{rt^{cm}}, \mathsf{rt^{excl}})$;

I am the holder of a note $\mathbf{n}$ that is unspent at pool snapshot
$(\mathsf{rt^{cm}}, \mathsf{rt^{excl}})$, such that $\mathbf{n}$ has value
commitment $\mathsf{cv}$ and alternate nullifier $\mathsf{nf_{dom}}$ in
nullifier domain $\mathsf{dom}$."

### Another possible approach

Have the holder *actually* spend the claimed notes (e.g. to themself) with an
anchor at the pool snapshot. This has the disadvantage that they cannot
participate in concurrent polls/air-drops.

## Alternate nullifier derivation

Given a nullifier domain $\mathsf{dom}$ and an Orchard note
$\mathbf{n} = (\mathsf{d^{old}}, \mathsf{pk_d^{old}}, \mathsf{v^{old}}, \text{ρ}^{\mathsf{old}}, \text{ψ}^{\mathsf{old}}, \mathsf{rcm^{old}})$
with note commitment $\mathsf{cm^{old}}$, the alternate nullifier $\mathsf{nf_{dom}}$
for $\mathbf{n}$ is computed as:
$$\begin{array}{rcl}
\mathsf{nf_{dom}} &\!\!\!=\!\!\!& \mathsf{DeriveAlternateNullifier_{nk}}(\text{ρ}^{\mathsf{old}}, \text{ψ}^{\mathsf{old}}, \mathsf{cm^{old}}, \mathsf{dom}) \\
&\!\!\!=\!\!\!& \mathsf{Extract}_{\mathbb{P}}\big(\big[(\mathsf{PRF^{nfAlternate}_{nk}} (\text{ρ}^{\mathsf{old}}, \mathsf{dom}) + \text{ψ}^{\mathsf{old}}) \bmod q_{\mathbb{P}}\big]\, \mathcal{K}^\mathsf{Orchard} + \mathsf{cm^{old}}\big)
\end{array}$$

Here $\mathsf{Extract}_{\mathbb{P}}$ and $\mathcal{K}^\mathsf{Orchard}$ are as in
the derivation of Orchard nullifiers in the protocol specification [^protocol],
and $\mathsf{PRF^{nfAlternate}}$ is another instantiation of Poseidon: a 3-input
Constant-Input-Length sponge over the width-3 Poseidon permutation of
§ 5.4.1.10 'PoseidonHash Function' [^protocol-poseidonhash], with initial
capacity element $3 \cdot 2^{64}$, that absorbs $(\mathsf{nk}, \text{ρ}, \mathsf{dom})$
and one zero padding element in two permutation calls. ($\mathsf{PoseidonHash}$
itself is defined only for two inputs.) This lets $\mathsf{dom}$ be an arbitrary
field element, since it never enters the capacity element. Its security rests on
the generic sponge/PRF model: the informal argument for $\mathsf{PRF^{nfOrchard}}$
in § 5.4.1.10 depends on its 2-input layout and does not carry over. A
4-element to 4-element permutation would need only one call, but also new round
constants and analysis.

> TODO: security analysis, in particular for collision attacks and for linkability across domains.


<details>
<summary>

### Rationale for not using a similar nullifier derivation to ZSA split notes
</summary>

The nullifier derivation in draft ZIP 226 for ZSA split notes
[^zip-0226-split-notes] is:
$$\mathsf{nf} = \mathsf{Extract}_{\mathbb{P}}\big(\big[(\mathsf{PRF^{nfOrchard}_{nk}} (\text{ρ}^{\mathsf{old}}) + \text{ψ}^{\mathsf{nf}}) \bmod q_{\mathbb{P}}\big]\, \mathcal{K}^\mathsf{Orchard} + \mathsf{cm^{old}} + \mathcal{L}^\mathsf{Orchard}\big)$$
where $\text{ψ}^{\mathsf{nf}} = \mathsf{ToBase^{Orchard}}\big(\mathsf{PRF^{expand}_{rseed\_nf}}([\mathtt{0x0A}] \,||\, \underline{\text{ρ}^{\mathsf{old}}})\big)$
for a random $\mathsf{rseed\_nf}$.

This fails to do what we want in two ways:

* It is nondeterministic, since $\text{ψ}^{\mathsf{nf}}$ is derived from a random
  $\mathsf{rseed\_nf}$.
* If $\text{ψ}^{\mathsf{nf}}$ were required to be $\text{ψ}^{\mathsf{old}}$ and
  $\mathcal{L}^\mathsf{Orchard}$ were replaced by a hash-to-curve of
  $\mathsf{dom}$, then nullifiers for different domains (including the original
  ZEC domain) would be linkable, since they would differ by a predictable point.
  (There are two possible points corresponding to a given nullifier, but this is
  only a trivial obstacle to linking them.)
</details>

## Making a claim

A Claim consists of one or more Note Claims in the same nullifier domain
$\mathsf{dom}$. For each note that the holder wants to claim, they prove an
instance $\pi$ of the Note Claim statement below, and provide this proof together
with a spend authorization signature $\sigma$ (constructed as though they were
spending the note, but over the claim message instead of a transaction sighash).

The claim message is application-defined, but it MUST commit to the
application's payload for the Claim (for example a poll choice or an air-drop
recipient) and to the primary input of every Note Claim in the Claim. Every
spend authorization signature of a Claim is over the same claim message, which
binds the Note Claims to each other and to the payload.

A verifier MUST check, for each Note Claim of a Claim, that:

* $(\mathsf{rt^{cm}}, \mathsf{rt^{excl}})$ is a published pool snapshot, and
  $\mathsf{dom}$ is the nullifier domain of the air-drop/poll;
* $\pi$ is valid for its primary input;
* $\mathsf{SpendAuthSig^{Orchard}.Validate_{rk}}(\mathsf{msg}, \sigma) = 1$,
  where $\mathsf{msg}$ is the claim message;
* $\mathsf{nf_{dom}}$ has not already been revealed in $\mathsf{dom}$, including
  by another Note Claim of the same Claim.

The value commitment of a Claim is the sum $\mathsf{cv^{total}}$ of the
$\mathsf{cv}$ values of its Note Claims. To open it, the holder reveals the total
value $\mathsf{v^{total}}$ of the claimed notes and the sum
$\mathsf{rcv^{total}}$ of their $\mathsf{rcv}$ values, modulo $r_{\mathbb{P}}$;
the verifier checks that
$\mathsf{cv^{total}} = \mathsf{ValueCommit^{Orchard}_{rcv^{total}}}(\mathsf{v^{total}})$.
Alternatively, $\mathsf{cv^{total}}$ can be used in another zk proof without
being opened.

> Can we make it more efficient to claim that you hold multiple notes?

## Circuit

A valid instance of a Note Claim statement, $\pi$, assures that given a primary input:
* $\mathsf{rt^{cm}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$
* $\mathsf{rt^{excl}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$
* $\mathsf{cv} ⦂ \mathsf{ValueCommit^{Orchard}.Output}$
* $\mathsf{dom} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$
* $\mathsf{nf_{dom}} ⦂ \{0 .. q_{\mathbb{P}}-1 \}$
* $\mathsf{rk} ⦂ \mathsf{SpendAuthSig^{Orchard}.Public}$

the prover knows an auxiliary input:

* $\mathsf{path^{cm}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}^{[\mathsf{MerkleDepth^{Orchard}}]}$
* $\mathsf{pos^{cm}} ⦂ \{ 0 .. 2^{\mathsf{MerkleDepth^{Orchard}}}\!-1 \}$
* $\mathsf{g_d^{old}} ⦂ \mathbb{P}^*$
* $\mathsf{pk_d^{old}} ⦂ \mathbb{P}^*$
* $\mathsf{v^{old}} ⦂ \{ 0 .. 2^{\ell_{\mathsf{value}}}-1 \}$
* $\text{ρ}^{\mathsf{old}} ⦂ \mathbb{F}_{q_{\mathbb{P}}}$
* $\text{ψ}^{\mathsf{old}} ⦂ \mathbb{F}_{q_{\mathbb{P}}}$
* $\mathsf{rcm^{old}} ⦂ \{ 0 .. 2^{\ell^{\mathsf{Orchard}}_{\mathsf{scalar}}}-1 \}$
* $\mathsf{cm^{old}} ⦂ \mathbb{P}$
* $\mathsf{nf^{old}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$
* $\alpha ⦂ \{ 0 .. 2^{\ell^{\mathsf{Orchard}}_{\mathsf{scalar}}}-1 \}$
* $\mathsf{ak}^{\mathbb{P}} ⦂ \mathbb{P}^*$
* $\mathsf{nk} ⦂ \mathbb{F}_{q_{\mathbb{P}}}$
* $\mathsf{rivk} ⦂ \mathsf{Commit^{ivk}.Trapdoor}$
* $\mathsf{rcv} ⦂ \{ 0 .. 2^{\ell^{\mathsf{Orchard}}_{\mathsf{scalar}}}-1 \}$
* $\mathsf{path^{excl}} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}^{[\mathsf{MerkleDepth^{excl}}]}$
* $\mathsf{pos^{excl}} ⦂ \{ 0 .. 2^{\mathsf{MerkleDepth^{excl}}}\!-1 \}$
* $\mathsf{start} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$
* $\mathsf{end} ⦂ \{ 0 .. q_{\mathbb{P}}-1 \}$

such that the following conditions hold:

**Note commitment integrity** $\hspace{0.5em} \mathsf{NoteCommit^{Orchard}_{rcm^{old}}}(\mathsf{repr}_{\mathbb{P}}(\mathsf{g_d^{old}}), \mathsf{repr}_{\mathbb{P}}(\mathsf{pk_d^{old}}), \mathsf{v^{old}}, \text{ρ}^{\mathsf{old}}, \text{ψ}^{\mathsf{old}}) \in \{ \mathsf{cm^{old}}, \bot \}$.

**Merkle path validity for** $\mathsf{cm^{old}} \hspace{0.5em} (\mathsf{path^{cm}}, \mathsf{pos^{cm}})$ is a valid Merkle path of depth $\mathsf{MerkleDepth^{Orchard}}$, as defined in § 4.9 'Merkle Path Validity', from $\mathsf{Extract}_{\mathbb{P}}(\mathsf{cm^{old}})$ to the anchor $\mathsf{rt^{cm}}$.

**Value commitment integrity** $\hspace{0.5em} \mathsf{cv} = \mathsf{ValueCommit^{Orchard}_{rcv}}(\mathsf{v^{old}})$.

**Nullifier integrity** $\hspace{0.5em} \mathsf{nf^{old}} = \mathsf{DeriveNullifier_{nk}}(\text{ρ}^{\mathsf{old}}, \text{ψ}^{\mathsf{old}}, \mathsf{cm^{old}})$.

**Spend authority** $\hspace{0.5em} \mathsf{rk} = \mathsf{SpendAuthSig^{Orchard}.RandomizePublic}(\alpha, \mathsf{ak}^{\mathbb{P}})$.

**Diversified address integrity** $\hspace{0.5em} \mathsf{ivk} = \bot$ or $\mathsf{pk_d^{old}} = [\mathsf{ivk}]\, \mathsf{g_d^{old}}$ where $\mathsf{ivk} = \mathsf{Commit^{ivk}_{rivk}}(\mathsf{Extract}_{\mathbb{P}}(\mathsf{ak}^{\mathbb{P}}), \mathsf{nk})$.

**Merkle path validity for** $(\mathsf{start}, \mathsf{end}) \hspace{0.5em} (\mathsf{path^{excl}}, \mathsf{pos^{excl}})$ is a valid Merkle path of depth $\mathsf{MerkleDepth^{excl}}$, as defined in § 4.9 'Merkle Path Validity', from $\mathsf{excl}$ to the anchor $\mathsf{rt^{excl}}$, where $\mathsf{excl} = \mathsf{MerkleCRH^{Orchard}}(\mathsf{MerkleDepth^{excl}}, \mathsf{start}, \mathsf{end})$.

**Nullifier in excluded range** $\hspace{0.5em} \mathsf{start} \leq \mathsf{nf^{old}} \leq \mathsf{end}$.

**Alternate nullifier integrity** $\hspace{0.5em} \mathsf{nf_{dom}} = \mathsf{DeriveAlternateNullifier_{nk}}(\text{ρ}^{\mathsf{old}}, \text{ψ}^{\mathsf{old}}, \mathsf{cm^{old}}, \mathsf{dom})$.

## Circuit implementation

All of these but the last three checks are analogous to the corresponding parts
of an Action statement, with three differences: the Merkle path validity check
for $\mathsf{cm^{old}}$ is unconditional, with no exception for
$\mathsf{v^{old}} = 0$; $\mathsf{cv}$ commits to $\mathsf{v^{old}}$ alone; and
$\mathsf{nf^{old}}$ is an auxiliary input rather than a primary input, so that a
claim cannot be linked to a spend of the note.

**Merkle path validity for** $(\mathsf{start}, \mathsf{end})$ is almost identical
to the other Merkle path validity check.

Alternate nullifier integrity is probably very similar to **Nullifier integrity**.

**Nullifier in excluded range** is fairly straightforward. Nullifiers are
arbitrary field elements so be careful of overflow. Since we can check outside
the circuit that $\mathsf{start} \leq \mathsf{end}$, the check becomes
equivalent to $0 \leq \mathsf{nf^{old}} - \mathsf{start} \leq \mathsf{end} - \mathsf{start}$.

## Rationale

> TODO


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3] or later](protocol/protocol.pdf)

[^protocol-merklepath]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 4.9: Merkle Path Validity](protocol/protocol.pdf#merklepath)

[^protocol-poseidonhash]: [Zcash Protocol Specification, Version 2026.8.0 [NU6.3]. Section 5.4.1.10: PoseidonHash Function](protocol/protocol.pdf#poseidonhash)

[^zip-0226-split-notes]: [ZIP 226: Transfer and Burn of Zcash Shielded Assets — Split Notes](zip-0226.rst#split-notes)

[^zip-0229]: [ZIP 229: Version 6 Transaction Format](zip-0229.md)
