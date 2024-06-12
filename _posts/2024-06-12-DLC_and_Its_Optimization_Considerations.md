---
layout: post
title:  "DLC and Its Optimization Considerations"
author: lynndell
categories: [ DLC, BitVM, OP-DLC ]
image: assets/images/20240612/OP_DLC_BitVM.png
---

# 1. Introduction
The Discreet Log Contract (DLC) is a contract execution framework based on an Oracle, proposed by Tadge Dryja of the Massachusetts Institute of Technology in 2018. DLC allows two parties to execute conditional payments based on predefined conditions. The parties determine possible outcomes, pre-sign them, and use these pre-signatures to execute payments when Oracle signs off on the result. Therefore, DLC enables new decentralized financial applications while ensuring the security of Bitcoin deposits.

Compared to the Lightning Network, DLC has the following significant advantages:
- **Privacy**: DLC offers better privacy protection than the Lightning Network. Contract details are only shared between the parties involved and are not stored on the blockchain. In contrast, Lightning Network transactions are routed through public channels and nodes, making their information public and transparent.
- **Complexity and Flexibility of Financial Contracts**: DLC can directly create and execute complex financial contracts on the Bitcoin network, such as derivatives, insurance, and betting, whereas the Lightning Network is primarily used for rapid, small-value payments and cannot support complex applications.
- **Reduced Counterparty Risk**: DLC funds are locked in multi-signature contracts and are only released when the result of a predefined event occurs, reducing the risk of any party not adhering to the contract. Although the Lightning Network reduces the need for trust, it still carries certain counterparty risks in channel management and liquidity provision.
- **No Need to Manage Payment Channels**: DLC operations do not require the creation or maintenance of payment channels, which are central to the Lightning Network and involve complex and resource-intensive management.
- **Scalability for Specific Use Cases**: While the Lightning Network somewhat increases Bitcoin's transaction throughput, DLC offers better scalability for complex contracts on Bitcoin.

Although DLCs hold significant advantages in the Bitcoin ecosystem, there are still some risks and issues, such as:
- **Key Risk**: There is a risk of leakage or loss of Oracle's private keys and the committed nonces, leading to the loss of user assets.
- **Centralized Trust Risk**: Centralization in Oracles can easily lead to denial-of-service attacks.
- **Decentralized Key Derivation Issue**: If the Oracle is decentralized, the Oracle nodes only possess shares of the private key. However, decentralized Oracle nodes cannot directly use BIP32 for key derivation based on these private key shares.
- **Collusion Risk**: If Oracle nodes collude among themselves or with one party, the trust issue with Oracles remains unresolved. A reliable monitoring mechanism is needed to minimize trust in Oracles.
- **Fixed Denomination Change Problem**: Conditional signatures require a deterministic, enumerable set of events before building the contract to construct the transaction. Hence, using DLC for asset reallocation has a minimum amount restriction, leading to the issue of fixed denomination change.

To address these, this paper proposes several solutions and optimization ideas to mitigate the risks and issues associated with DLCs, thereby enhancing the security of the Bitcoin ecosystem.
# 2. DLC Principle
Alice and Bob enter into a betting agreement on whether the hash value of the (n+k)-th block is odd or even. If it is odd, Alice wins the game and can withdraw the assets within time t; if it is even, Bob wins the game and can withdraw the assets within time t. Using DLC, the (n+k)-th block information is transmitted through an Oracle to construct a conditional signature, ensuring the correct winner gets all the assets.

**Initialization**: The generator of the elliptic curve is G, and its order is q.

**Key generation**: The Oracle, Alice, and Bob independently generate their own private and public keys.
- The Oracle's private key is z, and its public key is Z, satisfying Z=z⋅G;
- Alice's private key is x, and her public key is X, satisfying X=x⋅G;
- Bob's private key is y, and his public key is Y, satisfying Y=y⋅G.

**Funding transaction**: Alice and Bob create a funding transaction together, locking 1 BTC each in a 2-of-2 multi-signature output (one public key X belongs to Alice, and the other Y to Bob).

**Contract Execution Transactions (CET)**: Alice and Bob create two CETs to spend the funding transaction.
The Oracle computes the commitment

$$R:=k\cdot G$$

and then calculates S and S′

$$S:=R-hash(OddNumber,R)\cdot Z,$$

$$S':=R-hash(EvenNumber,R)\cdot Z$$

and broadcasts (R, S, S′).

Alice and Bob each compute the corresponding new public key

$$PK^{Alice}:=X+ S,$$

$$PK^{Bob}:=Y+ S'.$$

**Settlement**: After the (n+k)-th block appears, the Oracle generates the corresponding s or s′ based on the hash value of that block.
- If the hash value of the (n+k)-th block is odd, the Oracle computes and broadcasts 

$$s:=k-hash(OddNumber,R)\cdot z$$

- If the hash value of the (n+k)-th block is even, the Oracle computes and broadcasts

$$s':=k-hash(EvenNumber,R)\cdot z$$

**Withdrawal**: Either Alice or Bob can withdraw the assets based on the s or s′ broadcasted by the Oracle.
- If the Oracle broadcasts s, Alice can compute the new private key sk^{Alice} and withdraw the locked 2 BTC

$$sk^{Alice}:= x + s.$$

- If the Oracle broadcasts s′, Bob can compute the new private key sk^{Bob} and withdraw the locked 2 BTC

$$sk^{Bob}:= y + s'.$$

Analysis: The new private key sk^{Alice} calculated by Alice and the new public key PK^{Alice} satisfies the discrete logarithm relationship

$$sk^{Alice}\cdot G= (x+s)\cdot G=X+S=PK^{Alice}$$

Thus, Alice's withdrawal will be successful.

Similarly, the new private key sk^{Bob} calculated by Bob and the new public key PK^{Bob} satisfy the discrete logarithm relationship

$$sk^{Bob}\cdot G= (y+s')\cdot G=Y+S'=PK^{Bob}$$

Thus, Bob's withdrawal will be successful.

Furthermore, if the Oracle broadcasts s, it is useful for Alice, but not for Bob because Bob cannot compute the corresponding new private key sk^{Bob}. Similarly, if the Oracle broadcasts s′, it is useful for Bob, but not for Alice, because Alice cannot compute the corresponding new private key sk^{Alice}. Finally, the description above omits the time lock. A time lock must be added to allow one party to compute the new private key and withdraw within time t. Otherwise, if it exceeds time t, the other party can use the original private key to withdraw the assets.
# 3. DLC Optimizations
## 3.1 Key Management
In the DLC protocol, the Oracle's private key and the committed nonce are crucial. Leakage or loss of the Oracle's private key and the committed nonce can lead to the following four security issues:

**(1) Oracle Loses Private Key z**

If the Oracle loses its private key, the DLC cannot settle, necessitating the execution of a DLC refund contract. Therefore, the DLC protocol includes a refund transaction to prevent the consequences of the Oracle losing its private key.

**(2) Oracle's Private Key z Leakage**

If the Oracle's private key is leaked, all DLCs based on that private key face the risk of fraudulent settlement. An attacker who steals the private key can sign any desired message, gaining complete control over the outcomes of all future contracts. Moreover, the attacker is not limited to issuing a single signed message but can also publish conflicting messages, such as signing that the hash value of the (n+k)-th block is both odd and even.

**(3) Oracle's Leakage or Reuse of Nonce k**

If the Oracle leaks the nonce k, then at the settlement phase, regardless of whether the Oracle broadcasts s or s′, an attacker can calculate the Oracle's private key z as follows:

$$z:=(k-s)/hash(OddNumber,R)$$

$$z:=(k-s')/hash(EvenNumber,R)$$

If the Oracle reuses the nonce k, then after two settlements, an attacker can solve the system of equations based on the Oracle's broadcast signatures to deduce the Oracle's private key z in one of four possible scenarios,

case 1:

$$s_1=k-hash(OddNumber_1,R)\cdot z$$

$$s_2=k-hash(OddNumber_2,R)\cdot z$$

case 2:

$$s_1'=k-hash(EvenNumber_1,R)\cdot z$$

$$s_2'=k-hash(EvenNumber_2,R)\cdot z$$

case 3:

$$s_1=k-hash(OddNumber_1,R)\cdot z$$

$$s_2'=k-hash(EvenNumber_2,R)\cdot z$$

case 4:

$$s_1'=k-hash(EvenNumber_1,R)\cdot z$$

$$s_2=k-hash(OddNumber_2,R)\cdot z$$

**(4) Oracle Loses Nonce k**

If the Oracle loses the nonce k, the corresponding DLC cannot settle, necessitating the execution of a DLC refund contract.

Therefore, to enhance the security of the Oracle's private key, it is advisable to use BIP32 to derive child or grandchild keys for signing. Additionally, to increase the security of the nonce, the hash value k:=hash(z, counter) should be used as the nonce k, to prevent repetition or loss of the nonce.

## 3.2 Decentralized Oracle

In DLC, the role of the Oracle is crucial as it provides key external data that determines the outcome of the contract. To enhance the security of these contracts, decentralized Oracles are required. Unlike centralized Oracles, decentralized Oracles distribute the responsibility of providing accurate and tamper-proof data across multiple independent nodes, reducing the risk associated with a single point of failure and decreasing the likelihood of manipulation or targeted attacks. With a decentralized Oracle, DLCs can achieve a higher degree of trustlessness and reliability, ensuring that contract execution relies entirely on the objectivity of predetermined conditions.

Schnorr threshold signatures can be used to implement decentralized Oracles. Schnorr threshold signatures offer the following advantages:
- **Enhanced Security**: By distributing the management of keys, threshold signatures reduce the risk of single points of failure. Even if some participants' keys are compromised or attacked, the entire system remains secure as long as the breach does not exceed a predefined threshold.
- **Distributed Control**: Threshold signatures enable distributed control over key management, eliminating a single entity to hold all signing powers, thereby reducing the risks associated with power concentration.
- **Improved Availability**: Signatures can be completed as long as a certain number of Oracle nodes agree, increasing the system's flexibility and availability. Even if some nodes are unavailable, the overall system's reliability is not affected.
- **Flexibility and Scalability**: The threshold signature protocol can set different thresholds as needed to meet various security requirements and scenarios. Additionally, it is suitable for large-scale networks, offering good scalability.
- **Accountability**: Each Oracle node generates a signature share based on its private key share, and other participants can verify the correctness of this signature share using the corresponding public key share, enabling accountability. If correct, these signature shares are accumulated to produce a complete signature.

Therefore, the Schnorr threshold signature protocol has significant advantages in decentralized Oracles in terms of improving security, reliability, flexibility, scalability, and accountability.
## 3.3 Coupling of Decentralization and Key Management
In key management technology, an Oracle possesses a complete key z and, using BIP32 along with increments ω, can derive a multitude of child keys z+ω^{(1)} and grandchild keys z+ω^{(1)}+ω^{(2)}. For different events, the Oracle can use various grandchild private keys z+ω^{(1)}+ω^{(2)} to generate corresponding signatures σ for the respective events msg.

In the decentralized Oracle scenario, there are n participants, and a threshold signature requires t+1 participants, where $$t<n$$. Each of the n Oracle nodes possesses a private key share z_i,i=1,...,n. These n private key shares z_i correspond to a complete private key z, but the complete private key z never appears. Under the condition that the complete private key z does not appear, t+1 Oracle nodes use their private key shares z_i,i=1,...,t+1 to generate signature shares σ_i′ for the message msg′, and these signature shares σ_i′ combine into a complete signature σ′. Validators using the complete public key Z can verify the correctness of the message-signature pair (msg′,σ′). Since it requires t+1 Oracle nodes to collectively generate the threshold signature, it provides high security.

However, in a decentralized Oracle scenario, the complete private key z does not appear, and therefore direct key derivation using BIP32 is not possible. In other words, decentralized Oracle technology and key management technology cannot be directly integrated.

The paper "[Distributed Key Derivation for Multi-Party Management of Blockchain Digital Assets](https://ieeexplore.ieee.org/document/10476163)" **proposes a distributed key derivation scheme in threshold signature scenarios.** The core idea is based on **Lagrange interpolation polynomials**, where the private key share z_i and the complete private key z satisfy the following interpolation relationship:

$$z_i = \sum\limits_{j = 1}^n {f_j(i)} =z+\sum\limits_{j = 1}^n {a_{0,j}+a_{1,j}i+...+a_{t,j}i^t}$$

Adding the increment ω to both sides of the equation yields:

$$z_i +\omega =z+\omega +\sum\limits_{j = 1}^n {a_{0,j}+a_{1,j}i+...+a_{t,j}i^t}$$

This equation shows that the private key share z_i plus the increment ω still satisfies the interpolation relationship with the complete private key z plus ω. In other words, the child private key share z_i+ω and the child key z+ω satisfy the interpolation relationship. Therefore, each participant can use their private key share z_i plus the increment ω to derive the child private key share z_i+ω, used to generate child signature shares, and validate them using the corresponding child public key Z+ω⋅G.

However, one must consider hardened and non-hardened BIP32. Hardened BIP32 takes the private key, chain code, and path as input, performs SHA512, and outputs the increment and child chain code. Non-hardened BIP32, on the other hand, takes the public key, chain code, and path as input, performs SHA512, and outputs the increment and child chain code. In a threshold signature scenario, the private key does not exist, so only non-hardened BIP32 can be used. Or, using a homomorphic hash function, hardened BIP32 can be applied. However, a homomorphic hash function differs from SHA512 and is not compatible with the original BIP32.

## 3.4 OP-DLC: Oracle Trust-minimized
In DLC, the contract between Alice and Bob is executed based on the Oracle's signed result, thus requiring a certain level of trust in the Oracle. Therefore, the correct behavior of the Oracle is a major premise for the operation of DLC.

To reduce trust in the Oracle, research has been conducted on executing DLC based on the results of n Oracles, decreasing reliance on a single Oracle.

- The "n-of-n" model involves using n Oracles to sign the contract and executing the contract based on the results of all Oracles. This model requires all Oracles to be online for signing. If any Oracle goes offline or there is a disagreement on the results, it affects the execution of the DLC contract. The trust assumption here is that all Oracles are honest.
- The "k-of-n" model involves using n Oracles to sign the contract, executing the contract based on the results of any k Oracles. If more than k Oracles conspire, it affects the fair execution of the contract. Moreover, the number of CETs required in the "k-of-n" model is C_n^k times that of a single Oracle or the "n-of-n" model. The trust assumption in this model is that at least k out of n Oracles are honest.

Simply increasing the number of Oracles does not achieve de-trust of the Oracles because, after malicious actions by an Oracle, the aggrieved party in the contract has no on-chain recourse.

**Therefore, we propose OP-DLC, which incorporates an optimistic challenge mechanism into DLC.** Before participating in setting up the DLC, n Oracles need to pledge and build a permissionless on-chain OP game in advance, committing to not act maliciously. If any Oracle acts maliciously, then Alice, Bob, any other honest Oracle, or any other third-party honest observer can initiate a challenge. If the challenger wins the game, the on-chain system penalizes the malicious Oracle by forfeiting its deposit. Additionally, OP-DLC can also adopt the "k-of-n" model for signing, where the k value can even be 1. Therefore, the trust assumption is reduced to just needing one honest participant in the network to initiate an OP challenge and penalize a malicious Oracle node.

When settling OP-DLC based on Layer 2 computation results:
- If an Oracle signs with incorrect results, causing Alice to suffer a loss, Alice can use the correct Layer 2 calculation results to challenge the Oracle's advance-pledged permissionless on-chain OP game. Winning the game, Alice can penalize the malicious Oracle and compensate for her loss.
- Similarly, Bob, other honest Oracle nodes, and third-party honest observers can also initiate challenges. However, to prevent malicious challenges, the challenger must also pledge.

Therefore, OP-DLC facilitates mutual supervision among oracle nodes, minimizing the trust placed in oracles.This mechanism only requires one honest participant and has a 99% fault tolerance, effectively addressing the risk of Oracle collusion.
## 3.5 OP-DLC + BitVM Dual Bridge
When DLC is used for cross-chain bridges, fund distribution must occur at the settlement of the DLC contract:
- It necessitates pre-setting through CETs, meaning the fund settlement granularity of DLC is limited, such as 0.1 BTC in the Bison network. This raises an issue: Layer 2 asset interactions for users should not be restricted by the fund granularity of DLC CETs.
- When Alice wants to settle her Layer 2 assets, it forces the settlement of user Bob’s Layer 2 assets to Layer 1 as well. This raises an issue: each Layer 2 user should have the freedom to choose their deposits and withdrawals independently of other users' actions.
- Alice and Bob negotiate spending. The issue here is that it requires both parties to be willing to cooperate.

**Therefore, to address the aforementioned issue, we propose the OP-DLC + BitVM dual bridge. This solution enables users to deposit and withdraw through BitVM's permissionless bridge as well as through the OP-DLC mechanism, achieving change at any granularity and enhancing liquidity.**

In the OP-DLC, the Oracle is the BitVM federation, with Alice as a regular user and Bob as the BitVM federation. When setting up OP-DLC, the CETs built allow for immediate spending of Alice's output on Layer 1, while Bob's output includes a "DLC game Alice can challenge" with a timelock period. When Alice wants to withdraw:
- If the BitVM federation, acting as the Oracle, signs correctly, Alice can withdraw on Layer 1. However, Bob can withdraw on Layer 1 after the timelock period.
- If the BitVM federation, acting as the Oracle, cheats, causing a loss to Alice, she can challenge Bob's UTXO. If the challenge is successful, Bob’s amount can be forfeited. Note: another member of the BitVM federation can also initiate the challenge, but Alice, being the aggrieved party, is most motivated to do so.
- If the BitVM federation, acting as the Oracle, cheats, causing a loss to Bob, an honest member of the BitVM federation can challenge the "BitVM game" to penalize the cheating Oracle node.

Moreover, if user Alice wants to withdraw from Layer 2 but the preset CETs in the OP-DLC contract do not match the amount, Alice can choose the following methods:
- Withdraw through BitVM, with the BitVM operator fronting the amount on Layer 1. The BitVM bridge assumes there is at least one honest participant in the BitVM federation.
- Withdraw through a specific CET in OP-DLC, with the remaining change fronted by the BitVM operator on Layer 1. Withdrawing through OP-DLC will close the DLC channel, but the remaining funds in the DLC channel will move to the BitVM Layer 1 pool, without forcing other Layer 2 users to withdraw. The OP-DLC bridge assumes there is at least one honest participant in the channel.
- Alice and Bob negotiate spending without the involvement of an Oracle, requiring cooperation from Bob.

Therefore, the OP-DLC + BitVM dual bridge offers the following advantages:
- BitVM solves the DLC channel's change problem, reduces the number of CETs required, and is unaffected by CET fund granularity;
- By combining the OP-DLC bridge with the BitVM bridge, it provides users with multiple channels for deposits and withdrawals, improving user experience;
- Setting the BitVM consortium as both Bob and the oracle, and utilizing the OP mechanism, minimizes trust in the oracle;
- Integrating the excess withdrawal from the DLC channel into the BitVM bridge pool enhances fund utilization.

# 4. Conclusion
DLC emerged before the activation of Segwit v1 (Taproot) and has already been integrated with the Lightning Network, allowing the extension of DLC to update and execute continuous contracts within the same DLC channel. With technologies like Taproot and BitVM, more complex off-chain contract verifications and settlements can be implemented within DLC. Additionally, by integrating the OP challenge mechanism, it is possible to minimize trust in Oracles.

# References

1. [Specification for Discreet Log Contracts](https://github.com/discreetlogcontracts/dlcspecs)
2. [Discreet Log Contracts](https://adiabat.github.io/dlc.pdf)
3. [Scaling DLC Part1: Off-chain Discreet Log Contracts](https://medium.com/crypto-garage/scaling-dlc-part1-off-chain-discreet-log-contracts-8bb9a57c86b)
4. [Scaling DLC Part2: Free option problem with DLC](https://medium.com/crypto-garage/scaling-dlc-part2-free-option-problem-with-dlc-ff939311954c)
5. [Scaling DLC Part3: How to avoid free option problem with DLC](https://medium.com/crypto-garage/scaling-dlcs-part3-how-to-avoid-free-option-problem-with-dlcs-c4592a69559e)
6. [Lightning Network](https://static1.squarespace.com/static/6148a75532281820459770d1/t/61af971f7ee2b432f1733aee/1638897446181/lightning-network-paper.pdf)
7. [DLC on Lightning](https://medium.com/crypto-garage/dlc-on-lightning-cb5d191f6e64)
8. [DLC Private Key Management Part 1](https://suredbits.com/dlc-private-key-management-part-1/)
9. [DLC Private Key Management Part 2: The Oracle’s private keys](https://suredbits.com/dlc-private-key-management-part-2-the-oracles-private-keys/)
10. [DLC Key Management Pt 3: Oracle Public Key Distribution](https://suredbits.com/dlc-key-management-pt-3-oracle-public-key-distribution/)
11. [BitVM: Compute Anything on Bitcoin](https://bitvm.org/bitvm.pdf)
12. [BitVM 2: Permissionless Verification on Bitcoin](https://bitvm.org/bitvm2.html)
13. [BitVM Off-chain Bitcoin Contracts](https://brink.dev/assets/files/2024-01-16-eng-bitvm-slides.pdf)
14. [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
15. [Schnorr signature](https://en.wikipedia.org/wiki/Schnorr_signature)
16. [FROST: Flexible Round-Optimized Schnorr Threshold Signatures](https://link.springer.com/chapter/10.1007/978-3-030-81652-0_2)
17. [A Survey of ECDSA Threshold Signing](https://eprint.iacr.org/2020/1390)
18. [Distributed Key Derivation for Multi-Party Management of Blockchain Digital Assets](https://ieeexplore.ieee.org/document/10476163)
19. [Segregated Witness](https://en.wikipedia.org/wiki/SegWit)
20. [Optimistic Rollup](https://ethereum.org/en/developers/docs/scaling/optimistic-rollups/)
21. [Taproot](https://bitcoinops.org/en/topics/taproot/)