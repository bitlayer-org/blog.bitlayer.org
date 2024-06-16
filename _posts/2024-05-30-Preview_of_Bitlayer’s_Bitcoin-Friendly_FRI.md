---
layout: post
title:  "Preview of Bitlayer’s Bitcoin-Friendly FRI"
author: wangyao
categories: [ ZKP, STARK, BF-STARK ]
image: assets/images/common/common.jpg
---

## What have we done?

We are the first team to implement FRI (Fast Reed-Solomon Interactive Oracle of Proximity) verification in Bitcoin Script without using OP_CAT. To achieve this, we split the script into smaller segments to circumvent Bitcoin’s length limit and “glued” these segments together using BitCommitment, a gadget invented by BitVM. Our innovation was leveraging Bitcoin’s native Merkle tree, Taptree, for Merkle Commitment in the FRI protocol. This approach **avoids using OP_CAT and allows gas-free verification of Merkle paths**, as it is handled by Bitcoin’s consensus instead of the program.

## Why does it matter?

FRI is the core component of STARK proof systems. In essence, FRI allows a prover to demonstrate that they hold a function close to a polynomial over an evaluation domain. It is utilized in popular proof systems like Plonky2 and Plonky3. To implement STARK on Bitcoin, verifying FRI is essential.

We believe that zero-knowledge proof-based rollups are the only secure method to scale Bitcoin. Verifying ZKP proofs is foundational for building such systems. However, this is a challenging task due to the inherent nature of the Bitcoin network.

## Programming in Bitcoin and the implications for verifying ZKP proof on Bitcoin

Bitcoin’s programming language, Bitcoin Script, is deliberately designed to be Turing incomplete to avoid potential security risks. It offers basic primitives such as signatures, timelocks, and hashlocks. Additionally, Bitcoin Script programs cannot call other Bitcoin Script programs. Instead, you can attach a certain amount of Bitcoin to a script, and anyone who can run the script can claim the funds.

In essence, you cannot compose complex Bitcoin programs; rather, you can create more sophisticated ways to transfer money. There are two additional limitations for Bitcoin Script programs:

1. The length of a Bitcoin script cannot exceed 400KB, as most Bitcoin miners will refuse to include larger scripts in new blocks.
2. The stack depth of the Bitcoin VM is limited to 1000.

For example, implementing multiplication in Babybear Field’s degree 4 extension, a basic operation in verifying FRI, takes up about 14KB. This means you can only perform 28 multiplications in a single script. Therefore, even though it is theoretically possible to write a monolithic program to verify a STARK proof, the script length would become impractically large for Bitcoin.

## BitVM Model

Robin Linus developed a verification model called BitVM. While BitVM1 is more well-known, most teams are now working on BitVM2 due to its greater efficiency. The basic model is a challenge-response game between a prover and a verifier.

Two key primitives in the BitVM model are Taptree and BitCommitment:

- Taptree: A Bitcoin-native Merkle tree where the leaves are Bitcoin scripts. A UTXO can be published with the root hash of a Taptree. If someone provides a Bitcoin script that can run on Bitcoin along with a Merkle path for the script, they can claim the Bitcoin in the UTXO.
- BitCommitment: Introduced in the BitVM white paper, BitCommitment allows state propagation among Bitcoin scripts. For example, to propagate a binary value b, the prover provides two hashes corresponding to b = 0 and b = 1 and a time-locked UTXO with a certain amount of Bitcoin. The prover can only retrieve their coin by providing one preimage of the hash within the time limit. For more details, refer to BitVM white paper.

### Example from BitVM2

We quote the example from BitVM2. Suppose the prover wants to convince the verifier that the output of a very complicated function $f(x)=y$ and the execution of the function contains 42 steps. (In this case, $f$ is the STARK verifier.)

The prover creates a Taptree with 43 scripts to challenge any computation of $f_1, f_2, …, f_{42}$ . The prover commits x, y , and the intermediate results $z_1, z_2, …, z_{42}$ all at once. If the verifier finds any inconsistency in the results, they can run one of the scripts to spend the money at the Taptree root if $f_i(z_{i-1}) \neq z_i$.

## Our FRI Implimentation

In this chapter, we use the terms FRI prover and FRI verifier, and game responder and game challenger to distinguish between roles in the FRI and challenge-response game protocols.

### FRI Prover and Verifier

Our work is based on Plonky3 FRI. The FRI prover is similar to the Plonky3 FRI prover, with the primary difference being the use of Taptree for polynomial commitment. Note that Taptree requires the hash of the left node to be less than the hash of the right node, necessitating an index reference from the Merkle tree to the Taptree and recording the Merkle index in the leaf.

Our Bitcoin FRI verifier has two implementations: one in Rust and the other in Bitcoin Script. The Rust implementation is adopted from Plonky3 without modification. The Bitcoin Script implementation verifies the FRI folding and the Fiat-Shamir Transformation performed by the prover. It does not need to verify the Merkle tree inclusion. Additionally, the verifier splits the script into many short scripts (segments) and incorporates BitCommitment for the required intermediate states at the beginning of each script.

For the Fiat-Shamir Transformation, we use Blake3 hash. We also employ a more efficient BitCommitment implementation from BitVM called Winternitz Signature.

### Workflow

**Game Responder**

1. Calls the FRI prover program to commit an encoded polynomial and its foldings into Taptrees.
2. Calls the FRI verifier (the rust version) program to go through proof verification process.
3. Set the bounty into Verifier’s Taptree, which contains two subtrees:
- The FRI verifier (the Bitcoin script version, apparently) tree.
- The commitment Taptree with the encoded polynomial and its foldings.
4. Reveals the preimage of all BitCommitments on chain.

**Game Challenger**

1. Verifies the FRI proof offchain.
2. If the FRI proof is incorrect, plug all preimages of BitCommitment into the leaves of the Verifier’s tree.
3. Runs the leaf script of the Verifier’s Taptree to find an error.
4. Start a challenge at the Verifier’s Taptree’s leaf corresponding to the place where error is found.

(There is a thrid called Operator in the BitVM’s Model The Game Operator’s role is to pre-sign the Game Responder’s unlock transaction, ensuring the Verifier Taptree is constructed correctly.)

## Caveats

An unsolved problem is how to enforce the game responder to publish the contents of the Taptree leaves without transaction introspection. Currently, BitVM2 proposes an n-member committee to supervise the challenge-response game and pre-check each Taptree published by the game responder. Removing this committee or replacing it with a decentralized organization is a critical task for Bitcoin rollup builders, but no effective solution has been found so far.

## References

- Mastering Bitcoin V3 by Andreas M. Antonopoulos : [https://github.com/bitcoinbook/bitcoinbook](https://github.com/bitcoinbook/bitcoinbook)
- BitVM white paper by Robin Linus: [https://bitvm.org/bitvm.pdf](https://bitvm.org/bitvm.pdf)
- BitVM2 (unfinished writing) by Robin Linus: [https://bitvm.org/bitvm2.html](https://bitvm.org/bitvm2.html)
- A summary on the FRI low degree test by U Haböck [https://eprint.iacr.org/2022/1216.pdf](https://eprint.iacr.org/2022/1216.pdf)
- BitVM github: [https://github.com/BitVM/BitVM](https://github.com/BitVM/BitVM)
- Rust-Bitcoin: [https://github.com/rust-bitcoin/rust-bitcoin](https://github.com/rust-bitcoin/rust-bitcoin)
- Plonky3: [https://github.com/Plonky3/Plonky3](https://github.com/Plonky3/Plonky3)
