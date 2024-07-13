---
layout: post
title: "BitVM Bridge Becomes Practical"
author: xavier
categories: [BitVM, Bridge]
image: assets/images/20240705/bitvm_bridge.png
---

Disclaimer: Some ideas and specifications in this article are currently theoretical. Consult the BitVM community if you find anything uncertain.

It's been a while since our [last BitVM experiment article](https://blog.bitlayer.org/Experiment_of_BitVM_White_Paper/), and lots of changes have happened in the BitVM community. After several iterations, the BitVM bridge protocol now becomes practical and we may see Robin's BitVM bridge come out sometime this year.

# From BitVM1 to BitVM2

Although the BitVM1 solution demonstrated the possibility of generic computing on Bitcoin, it was considered not feasible for production.

1. In BitVM1, computation programs are compiled into RISC-V instructions to trace the execution. Verifiers must identify one of potentially billions of RISC-V instructions that executes incorrectly in order to challenge. BitVM's verification process is segmented into multiple rounds of challenge-response: the verifier selects a specific RISC-V instruction to challenge, the prover reveals the corresponding input and output, and the onchain verifier checks the correctness of the execution by replaying the instruction. This process continues using binary search to locate the target RISC-V instruction. In the worst-case, this interactive challenge can extend to dozens of rounds, resulting in prolonged final confirmation times.
2. The BitVM1 protocol supports only one single verifier, and non-cooperation of the verifier will halt the entire protocol.

To address the limitations of BitVM1, Robin recently proposed BitVM2. This version divides complex computational programs into sub-functions instead of RISC-V programs. The division is done according to Bitcoin block/transaction size limits. For example, a SNARK validator can be divided into about 43 sub-functions, each written in Bitcoin script. In the challenge game, the prover reveals the inputs and outputs of all sub-functions at once. The verifier then checks each sub-function to identify errors and punish the prover. While this increases the size of script to replay on Bitcoin, it reduces the multi-round interactive challenge-response process to one round, significantly mitigating the heavy on-chain footprint. Additionally, BitVM2 uses connector outputs to link transactions, allowing any verifier to initiate a challenge before the prover's withdrawal time lock expires, thereby enhancing the security and stability of the system.

With these innovations, Robin introduces a trustless cross-chain bridge solution for secure BTC transfers between Bitcoin and a side system. Below is the latest BitVM bridge protocol (a transaction graph posted in the BitVM Builders Group on 7/1/2024).

[![permissionless_bitvm_bridge](/assets/images/20240705//permissionless_bitvm_bridge.jpg)](/assets/images/20240705//permissionless_bitvm_bridge.jpg)

# An In-depth Analysis of BitVM Bridge

## BitVM Bridge Overview

The BitVM bridge facilitates BTC transfers between the Bitcoin mainnet and a side system. Users can lock BTC in a multisig address on Bitcoin and mint an equivalent amount of wBTC on a side system, what we're calling the Peg-in operation. Then wBTC holders can initiate a withdrawal, burn their wBTC, and request BTC from liquidity providers predefined in this BitVM bridge protocol. The liquidity provider then provides a zero-knowledge proof to retrieve the fronted BTC from the multisig address.

The Bitlayer research team has tried hard to fill in the missing pieces to redrawn the graph as below:

[![permissionless_bitvm_bridge_bitlayer](/assets/images/20240705//permissionless_bitvm_bridge_bitlayer.jpg)](/assets/images/20240705//permissionless_bitvm_bridge_bitlayer.jpg)

In the next chapters, we give an in-depth analysis of the BitVM bridge protocol. We hope this article can help readers understand the overall protocol process and the design rationale behind it.

## System Roles

There are five roles in the BitVM bridge.

### Federation: contains several cosigners

The BitVM bridge borrowed the idea from classic layer 2 solutions such as State Channels and uses pre-signed transactions to define the protocol all participants must obey. A federation of n members takes the role of  enforcing protocol compliance through an n-of-n multisig and temporarily holds bridged funds in one n-of-n multisig address on Bitcoin.

The federation's cosigners convene for the BitVM bridge protocol launch and create the n-of-n multisig address. Concurrently, they each gather necessary information, then build all transactions in the graph and presign some of them. Once the members disclose the transaction graph, they must delete their private keys. Hereafter, the protocol operates only requiring participants to complete and broadcast these pre-signed transactions.

The security assumption for the BitVM federation hinges on at least one member deleting their private key as mandated, known as the 1-of-n security model.

### Alice is the Depositing User

Alice is a Bitcoin user who holds a large amount of BTC and wants to move the fund to a side system. Alice decides to deposit her BTC through the BitVM bridge. Once the federation finishes the setup, Alice checks that all pre-signed transactions adhere to the BitVM bridge protocol. She then transfers her BTC to the n-of-n multisig address. She also mints the corresponding amount of wBTC in the side system.

### Bob is the Withdrawing User

Bob is a normal user in the side system who wants to withdraw a certain amount of wBTC to Bitcoin. Bob sends a withdrawal request of a certain amount of BTC (along with an expected fee rate) to a group of liquidity providers. After one liquidity provider agrees to fulfil the request, Bob burns the same amount of wBTC in the side system, and then waits for the liquidity provider to serve the withdrawal request.

### Operator is the Liquidity Provider

Operators are a group of Bitcoin users who also hold a significant amount of BTC and want to profit from the fund. In the BitVM bridge, all transactions must be predefined during the setup phase. Since the recipients of future withdrawal transactions are unknown at this stage, the BitVM bridge employs a "front-and-reimbursement" model. Liquidity providers front Bob's withdrawal request with their own liquidity. Notice that the amount Bob received will be a little less than Bob's request, and the difference is the service fee that goes to the operator. The operator then reclaims his capital from the n-of-n multisig address through a reimbursement process.

### Verifier can be Anyone

Because the operator may start an invalid reimbursement to steal fund from the n-of-n multisig address, the BitVM bridge uses fault proof as its core security mechnism. After the operator initiates the reimbursement process, if no fraud proof is submitted within the dispute time window, the operator can retrieve the corresponding BTC from the BitVM bridge multisig address. Conversely, if the verifiers identify the reimbursement as illegal within this window, they can challenge the operator. If the operator fails to provide valid proof within a specified timelock, the verifier will confiscate the BTC locked by the operator as punishment.

## Transaction process

### BitVM Setup

When Alice decides to deposit her BTC to the side system, the cross-chain protocol initiates:
1. Through an off-chain protocol, several cosigners are convened to form a federation. These cosigners can be selected in a decentralized manner or invited based on their community reputation. Once the federation members are determined, they will create an n-of-n multisig address to hold bridged funds.
2. Alice constructs a Peg-in Transaction to transfer 100 BTC to the n-of-n multisig address, aiming to mint the corresponding amount of wBTC in the side system. Alice does not sign this transaction at this stage but sends the transaction information to the federation.
3. Operators are convened through an off-chain protocol. Each operator constructs its own Kickoff 1 Transaction, Kickoff 2 Transaction, and Start Time Transaction (see the Key Transaction Types section for detailed explanations). Concurrently, the Challenge Transaction, Kickoff Timeout Transaction, and the second output UTXO of the Kickoff 1 Transaction referenced by the Start Timeout Transaction are pre-signed (pre-signed connector UTXOs are available for any verifier to challenge). Upon completion, every operator sends these transactions/signatures to the federation.
4. The federation verifies the legitimacy of Alice's Peg-in Transaction and other transaction information submitted by participating operators. It then constructs and pre-signs the subsequent Assert Transaction, Disprove Transaction, Disprove Chain Transaction, Take1 Transaction, and Take2 Transaction.
5. After the collection, verification, and pre-signing of all transactions are completed, the federation will publicize these transactions, signatures, and other basic information required for building transactions.
6. Finally, the cosigner will be asked to delete the private key. As long as one honest cosigner deletes the key, it becomes impossible for anyone to illegally withdraw funds from the multi-signature address.

### Alice deposits BTC from the Bitcoin mainnet

1. Alice will verify if all transactions are built according to the BitVM bridge protocol and include the corresponding signatures. Then she will sign and broadcast the Peg-in Transaction specified earlier.
2. The side system audits the legality of the entire BitVM Bridge protocol and the Peg-in Transaction, then mints 100 wBTC to Alice.

### Bob withdraws BTC to the Bitcoin mainnet

1. When Bob needs to withdraw BTC to the Bitcoin mainnet, he must first initiate a withdraw transaction on the side system to destroy 100 wBTC. Simultaneously, he constructs a Peg-out Transaction on the Bitcoin mainnet. This Peg-out Transaction includes two inputs: one with 0 BTC, recording Bob's withdraw transaction on the side system, and another awaiting the operator to add UTXO for fronting. The output is 99 BTC transferred to Bob, indicating that Bob is willing to pay 1 BTC as a service fee. Bob will pre-sign the first input of this transaction and bind it to the withdraw transaction on the side system. By pre-signing, Bob essentially creates an open transaction, "selling" his output to one of the operators. Operators can compete for this transaction without further interaction, and the winner can earn profits through the bridge protocol.
2. The operator checks the side system for a withdrawal record anchored to the Peg-out Transaction. After confirming the service fee was satisfactory, the operator filled in the input with 99 BTC and broadcasted the transaction to the Bitcoin mainnet. At this point, Bob completes the withdrawal process.

### Operator reimbursement

1. **Kickoff 1 Transaction**: After the operator fronts that capital, they will sign and broadcast the Kickoff 1 Transaction built during the Setup phase to open the reimbursement path. At this time, the operator will lock 2 BTC as a guarantee for honest reimbursement. Any identified malicious behavior will not only prevent the operator from obtaining the payment cost and profit but also result in the loss of the 2 BTC used as a guarantee.
2. **Start Time Transaction**: Within 6 hours after the confirmation of the Kickoff 1 Transaction, the operator needs to send a Start Time Transaction to reveal the start time of Bob's withdrawal request. The start time should be slightly earlier than when Bob initiated the Withdraw Transaction in the side system. The Start Time Transaction will also verify that the current BTC time is greater than the start time.
   1. **Start Time Timeout Transaction**: If the operator does not send a Start Time Transaction within 6 hours to consume the third output of the Kickoff 1 Transaction, any verifier can construct a Start Time Timeout Transaction to punish the operator and terminate the reimbursement process.
3. **Kickoff 2 Transaction**: Two weeks after the confirmation of the Kickoff 1 Transaction, the operator will construct and sign the Kickoff 2 Transaction, revealing the consensus state of the side system between the start time and two weeks later. The goal is to anchor a side system state in Bitcoin for subsequent challenge verification. The operator must front within two weeks after Bob issues the Withdraw transaction, and this Peg-out Transaction should also be processed in the side system within the same period.
   1. **Kickoff Timeout Transaction**: The Kickoff 2 Transaction must be announced as soon as possible two weeks after the confirmation of the Kickoff 1 Transaction on the Bitcoin mainnet. Otherwise, any verifier can construct a Kickoff Timeout Transaction to punish the operator and terminate the reimbursement process after two weeks and one day.
   2. **Disprove Chain Transaction**: If the side system state revealed by the operator in the Kickoff 2 Transaction is illegal, any verifier can construct a Disprove Chain Transaction to reveal the true side system state. For example, if the side system is a POW chain, the verifier can submit a fork with a larger proof of work within the time range of the start time to two weeks later to initiate a Disprove Chain Transaction. This will punish the operator and terminate the reimbursement process.
4. **Take1 Transaction**: The operator will wait for the verifier to verify the submitted side system status and reimbursement behavior. This includes checking whether there is a corresponding Bob withdraw transaction in the side system and whether the reimbursement amount and receiving account correspond one-to-one. If there is no verifier challenge within three days after the Kickoff 2 Transaction confirmation, the operator will broadcast the Take1 Transaction to withdraw 100 BTC (including 99 BTC and 1 BTC handling fee) from the n-of-n multi-signed address and withdraw the 2 BTC previously used for guarantee.
5. **Challenge Transaction**: If the verifier detects that there is no withdrawal transaction and Peg-out Transaction that complies with the protocol within the start time to two weeks later, the verifier will pay 1 BTC to construct and broadcast a Challenge Transaction that consumes the first output of Kickoff 1 Transaction to initiate a challenge. This 1 BTC is used to cover the cost of the operator's response to the challenge and to prevent malicious challenges. As a result, the operator will be unable to retrieve assets through the Take1 Transaction.
   1. **Assert Transaction**: When the operator recognizes that the reimbursement behavior is challenged, it will initiate an Assert Transaction three days after the confirmation of the Kickoff 2 Transaction (reserving time for the Disprove Chain). This transaction will reveal the zero-knowledge proof of the correct reimbursement and the intermediate variables used by the proof subfunction. The zero-knowledge proof verifies whether there is a corresponding withdraw transaction in all blocks of the side system exposed by the operator within two weeks and correctly handles the Peg-out Transaction on the Bitcoin mainnet. One possible implementation is that the side system combines Bob's withdraw transaction with the corresponding Peg-out Transaction on the Bitcoin mainnet and binds it to the Kickoff 1 Transaction issued by the operator. This also solves the double-spending problem, as the operator cannot reuse the same Kickoff Transaction to apply for reimbursement for the same Peg-out Transaction.
      1. **Disprove Transaction**: Any verifier can synchronize the Assert Transaction from the Bitcoin mainnet to obtain the intermediate variables used to verify the zero-knowledge proof sub-function. The verifier can then judge the execution correctness of the zero-knowledge proof verification program locally. If the verifier recognizes an execution error in the sub-function, it can provide the input and output unlocking margin submitted by the operator to punish the operator and terminate the reimbursement process.
      2. **Take2 Transaction**: If no verifier initiates a Disprove Transaction within two weeks after the Assert Transaction is confirmed, the operator will broadcast a Take2 Transaction, withdrawing 100 BTC from the n-of-n multi-signed address and 2 BTC previously used for collateral.
   2. **Assert Timeout Transaction**: In practice, there should be a Timeout Slash Transaction. This is to avoid the scenario where the operator does not issue an Assert Transaction and keeps 2 BTC locked in the n-of-n address forever. Robin responded that this transaction would be added in the next update.

## Key transaction types

### Take1/2 Transaction

Take1/Take2 Transactions are the only way for the federation cosigner to extract BTC from the multi-signed address after deleting the private key. Operators are motivated to construct and broadcast the pre-transaction of Take1/Take2 Transactions so they can retrieve the BTC paid for Bob, the service fee, and the 2 BTC previously used for collateral.

### Kickoff 1 Transaction

The Kickoff 1 Transaction is the initiation transaction of the entire reimbursement process. The transaction includes an input of 2 BTC and three outputs:
1. The address of P2WPKH or P2TR (ensuring that the transaction hash remains unchanged) can be unlocked by the operator. There are two scenarios for using this UTXO:
   1. The operator will pre-sign this UTXO so that it can be used in the Take1 Transaction. By using the `SIGHASH_ALL` signature type, all inputs and outputs of Take1 are locked and cannot be changed. As shown in Path &#9312; in the figure.
   2. The operator will pre-sign this UTXO so that it can be used in the Challenge Transaction. By using the `SIGHASH_SINGLE` signature type, the first output is locked to transfer 1 BTC to the operator, allowing the verifier to add other inputs and outputs to extract the remaining BTC. As shown in Path &#9313; in the figure.
2. A P2TR address, which can be unlocked by the operator, includes 2 BTC locked by the operator. There are three scenarios for using this UTXO:
    1. The operator can reveal the state of the side system by opening Bitcommitment (refer to the article:  [https://bitvm.org/bitvm.pdf](https://bitvm.org/bitvm.pdf)) two weeks after the Kickoff 1 Transaction confirmation. A possible implementation is that the script checks that the submitted block header is within the specified time, the blocks are continuous, and they can be traced back to the genesis block of the side system. As shown in Path &#9314; in the figure.
    2. The operator will pre-sign to unlock the UTXO so that any verifier can use it in the Kickoff Timeout Transaction after two weeks and one day of the Kickoff 1 Transaction confirmation. By using the `SIGHASH_SINGLE` type to lock the first output and destroy 1.9 BTC (presumably to prevent the operator from challenging himself), the verifier is allowed to add other inputs and outputs to extract the remaining BTC. As shown in Path &#9315; in the figure.
    3. The operator will pre-sign the Taptree Time Lock leaf node so that any verifier can use the third output of the Kickoff 1 Transaction in the Start Timeout Transaction. By using the `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` signature type, the two inputs of the transaction and the first output of 1.9 BTC are locked. As shown in Path &#9316; in the figure.
3. A P2TR address, which can be unlocked by the operator. There are two scenarios for using this UTXO:
    1. The operator needs to reveal the start time of the reimbursement process by opening Bitcommitment within six hours after the Kickoff 1 confirmation. Meanwhile, the script will verify whether the current Bitcoin mainnet block timestamp is greater than the start time. As shown in Path &#9317; in the figure.
    2. The operator will pre-sign the Taptree unlock condition so that any verifier can use it in the Start Timeout Transaction together with the second output of Kickoff 1, six hours after the Kickoff 1 confirmation. By using the `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` signature type, the two inputs of the transaction and the first output of 1.9 BTC are locked, allowing the verifier to add other inputs and outputs to extract the remaining BTC. As shown in Path &#9318; in the figure.

The format and content of Kickoff 1 Transactions are determined during the BitVM setup. The federation needs to verify whether Kickoff 1 Transactions lock 2 BTC and generate three corresponding outputs. The federation constructs Take1 Transactions based on the first output of Kickoff 1 Transactions.

### Start Time Transaction

The Start Time Transaction reveals the start time of the withdrawal request by opening Bitcommitment. The format and content of this transaction are determined during the BitVM setup. The federation needs to verify whether the Bitcommitment promised for the start time in the Start Time Transaction is the same as that promised for the Kickoff 2 Transaction and the Disprove Transaction.

### Kickoff 2 Transaction

The operator reveals the status of the side system two weeks after the confirmation of Kickoff 1. According to the normal process, the side system is highly likely to have completed the synchronization and confirmation of Bitcoin mainnet Peg-out Transactions. Therefore, the status submitted by the side system two weeks later can be used to check and verify the existence and correctness of the withdrawal and Peg-out during this period.

The Kickoff 2 Transaction includes two outputs:
1. The P2TR address that the operator can unlock: The operator can unlock it for building the Take1 Transaction three days after the Kickoff 2 Transaction confirmation.
2. The P2TR address, providing n-of-n signature unlocking, includes 2 BTC. There are two scenarios for using this UTXO:
    1. The federation will pre-sign this UTXO so that it can be used in the Take1 Transaction. By using the `SIGHASH_ALL` signature type to lock Take1, all inputs and outputs cannot be changed. As shown in Path &#9319; in the figure.
    2. The federation will pre-sign this UTXO so that it can be used in the Assert Transactions. By using the `SIGHASH_ALL` signature type to lock Assert, all inputs and outputs cannot be changed. Assert requires opening the BitCommitment corresponding to all intermediate variables needed to verify the zero-knowledge proof. As shown in Path &#9320; in the figure.
    3. The federation will pre-sign this UTXO so that it can be used in the Disprove Chain Transaction. By using the `SIGHASH_SINGLE` type to lock the first output and destroy 1 BTC, the verifier can add other inputs and outputs to extract the remaining BTC. In this case, the verifier pays a higher proof cost but can obtain more profits compared to the Timeout slash case. As shown in Path &#9321; in the figure.

The format and content of Kickoff 2 Transactions are determined during BitVM Setup. The federation needs to verify whether Kickoff 2 Transactions use the second output of Kickoff 1 Transactions and generate two new outputs. Based on the output of Kickoff 2 Transactions, the federation will construct Take1 Transactions.

### Disprove Chain

After the Kickoff 2 Transaction is issued, if the verifier finds that the consensus state $ S_B $ of the side system exposed by the operator in the Kickoff 2 Transaction is not the longest chain, they can obtain a challenge reward of 1 BTC. This is achieved by submitting a correct side system consensus state $ S_B' $ that satisfies $ S_B'.weight > S_B.weight $ within the time range of (start_time, start_time + 2 weeks).

### Challenge Transaction

If there is any objection to the operator's reimbursement behavior, any verifier can challenge by providing 1 BTC and unlocking the second output of the Kickoff 1 Transaction. Since this UTXO is essential for Take1 redemption, the operator must prove their honest behavior through an Assert Transaction when challenged, and then proceed with the subsequent Take2 redemption. To ensure efficiency and prevent cost-free challenges for each reimbursement, this transaction requires a 1 BTC challenge fee, which is paid to the operator and can be used for subsequent Assert Transaction fees.

Since anyone can initiate a Challenge Transaction, it is not fully determined during BitVM setup. The federation only needs to verify if the operator has unlocked the second output of the Kickoff 1 Transaction using the `SIGHASH_SINGLE` signature type, while ensuring one of the Challenge Transaction outputs transfers 1 BTC to the operator.

### Assert Transaction

After the verifier initiates a Challenge Transaction, the honest operator is motivated to construct an Assert Transaction to retrieve funds through Take2. The Assert Transaction includes an input where the operator must reveal the Bitcommitment of the intermediate variables of all sub-functions involved in zero-knowledge verification, while generating two new outputs:
1. The P2TR address that the operator can unlock: The operator can unlock it to build the Take1 Transactions two weeks after Assert confirmation.
2. P2TR address, providing n-of-n signature unlocking, including 2 BTC. There are two scenarios for using this UTXO:
   1. The federation will pre-sign this UTXO so it can be used in the Take2 Transaction. Using the `SIGHASH_ALL` signature type to lock Take1 ensures that all inputs and outputs cannot be changed. As shown in Path &#9322; in the figure.
   2. The federation will pre-sign this UTXO so it can be used in the Disprove Transaction. Using the `SIGHASH_SINGLE` signature type to lock the first output and destroy 1 BTC allows the verifier to add other inputs and outputs to extract the remaining BTC. As shown in Path &#9323; in the figure.

The format and content of the Assert Transaction are determined during BitVM setup. The federation needs to verify whether the Kickoff 2 Transaction uses the second output of the Kickoff 1 Transaction and generates two new outputs. At the same time, the federation also needs to verify whether the Bitcommitment revealed by the Assert Transaction matches the Bitcommitment used in the intermediate variables of the zero-knowledge proof verification sub-function. The federation will construct the Take1 Transaction based on the output of Kickoff 2 Transaction.

### Disprove Transaction

Within two weeks of the Assert Transaction, if someone discovers that the intermediate variables revealed by the operator in the Assert Transaction cannot satisfy the state transition relationship, they can verify the failure of the sub-function execution on-chain by providing the input and output of the corresponding sub-function. This allows them to unlock the tapleaf of Taptree 2 and obtain a challenge reward of 1 BTC (destroying 1 BTC).

## Security and liveness analysis

### Security

The correct execution of the BitVM bridge protocol transaction process is guaranteed by the federation's n-of-n multi-signature. The honest cosigners in the federation will immediately delete their private keys after pre-signing. Therefore, as long as one member is honest, it can ensure that the federation cannot conspire to steal the funds in the n-of-n address.

### Liveness

The BitVM bridge liveness issue can be viewed from the following two aspects:
1. The liveness of the federation: The pre-signing process requires the participation of all members. According to Robin's expectation of having 1000 federation members, ensuring that all 1000 members are online and pre-sign correctly is a challenge that needs to be addressed.
2. The liveness of the bridge protocol: After the federation completes the pre-signing and dissemination, the bridge protocol will be finalized as long as the Kickoff Transaction is issued. Therefore, the liveness problem mainly hinges on whether the Kickoff Transaction can be issued smoothly. The necessary prerequisite for the operator to send the Kickoff Transaction is that Alice's Peg-in Transaction is successfully finalized. If Alice chooses not to send the Peg-in Transaction after the federation's pre-signing and publicity, it will cause the protocol to fail to initiate, resulting in wasted effort by the federation and potential attacks .

## Capital efficiency

1. Each operator needs to prepare a 2-BTC UTXO in reserve before the setup phase. Although this UTXO is not locked, if the operator consumes it outside of the protocol, the Kickoff 1 Transaction will not be valid any more, and the operator can't front any withdrawal request.
2. In each front-and-reimbursement process, the operator needs to front n BTC (99 BTC in this scenario) and unlock the UTXO used for guarantee (2 BTC in this scenario). The operator can only retrieve the funds after the Take1 or Take2 Transaction is confirmed. Retrieving the funds through the Take1 Transaction path takes at least 2 weeks and 3 days, while the Take2 Transaction path takes at least 4 weeks and 3 days.
3. In this article, the amount of each Peg-in and Peg-out is 100 BTC. However, Robin mentioned in a previous interview with Bankless that in practical scenarios, the assets locked by Peg-in may be reimbursed multiple times. For example, the operator can use a smaller granularity such as 10 BTC for a single reimbursement. After the operator completes a 10 BTC payment and successfully reimburses it (2 weeks and 3 days or 4 weeks and 3 days), the operator can continue to use the reimbursed UTXO for subsequent Peg-out Transactions. However, in this case, the number of transactions that need to be pre-signed during setup will also increase.

# Summary

Robin's ingenious design has introduced a more secure and decentralized cross-chain bridge solution for Bitcoin. The community is actively collaborating to further optimize costs and enhance user experience, ensuring the protocol continues to evolve. We believe that in the near future, the BitVM bridge will become a crucial component of the Bitcoin ecosystem.

The BitVM paradigm can also be used in other scenarios such like rollups. Bitlayer is trying to push the first rollup protocol based on the BitVM paradigm to the market. Stay tuned!

# References

- [BitVM 2: Permissionless Verification on Bitcoin](https://bitvm.org/bitvm2.html)
- [Is This the Future of Bitcoin?](https://www.youtube.com/watch?v=8HqDykVPypM)
- [Experiment of BitVM White Paper](https://blog.bitlayer.org/Experiment_of_BitVM_White_Paper/)
- [BITVM 2: OPENING UP THE PLAYING FIELD](https://bitcoinmagazine.com/technical/bitvm-2-opening-up-the-playing-field)
- [BitVM: Compute Anything on Bitcoin](https://bitvm.org/bitvm.pdf)
