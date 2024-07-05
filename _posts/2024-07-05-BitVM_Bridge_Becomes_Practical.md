---
layout: post
title: "BitVM Bridge Becomes Practical"
author: xavier
categories: [BitVM, Bridge]
image: assets/images/20240611/op_cat.png
---

Disclaimer: some words in this article are just speculations. Consult BitVM community if you find anything doubtful.

It's been a while since our last BitVM article, and lots of changes have happened in the BitVM community. After several iterations, the BitVM bridge protocol now becomes practical and we may see Robin's BitVM bridge come out sometime this year.

# From BitVM1 to BitVM2

Although the previous BitVM solution expanded the computing power of Bitcoin, it was only in the theoretical verification stage and still has a long way to go before it can be used for production.

1. BitVM compiles complex computational programs into binary circuits composed of NAND gates implemented by Bitcoin scripts. Verifiers may need to find one of the billions of NAND gates that execute incorrectly to initiate punishment against the prover. Due to the block size and transaction capacity limitations of the Bitcoin network, provers cannot reveal the input and output of all circuits in a single transaction. Therefore, BitVM's verification process is decomposed into multiple rounds of challenge-response: the verifier selects the challenged NAND gate, the prover reveals the corresponding input and output, and the verifier verifies whether the execution is correct and continues to search for the target NAND gate through binary search. This interaction challenge can reach dozens of rounds in the worst-case scenario, resulting in prolonged final confirmation time and low protocol operation efficiency.
2. The BitVM protocol only includes a single prover and a single validator, and its support for complex scenarios is insufficient.

In order to solve the limitations of the previous BitVM solution, Robin recently proposed a more advanced version, BitVM2: splitting complex calculation programs into several sub-functions based on block/transaction size (for example, the Snark zero-knowledge proof validator can be split into about 43 sub-functions written in Bitcoin scripts), and using sub-functions to replace each NAND gate. Although this approach increases the size of a single validation script, it significantly reduces the number of intermediate input and output states that provers need to reveal. When a challenge occurs, the prover can submit the input and output of all sub-functions at once. The verifier can verify each sub-function based on this information to find a subfunction that executes incorrectly, and then punish the prover. This design reduces the original multi-round interactive challenge response to two rounds. In addition, BitVM2 uses connector output to connect the two transactions before and after, allowing any verifier to construct a transaction before the prover's withdrawal Time Lock is opened, consuming connector output to force entry into the challenge process. This design improves the security and stability of the system.

With the support of the above innovations, Robin provides a trustless cross-chain bridge solution to achieve the secure transfer of BTC between Bitcoin and the side system. See below for the latest BitVM bridge protocol (a transaction graph posted in BitVM Builders Group on 7/1/2024). The author continues to track the official BitVM documentation and community discussions, attempting to provide an in-depth interpretation of the latest BitVM protocol, hoping to help readers understand the overall protocol process and the design thinking behind it.

[![permissionless_bitvm_bridge](/assets/images/20240705//permissionless_bitvm_bridge.jpg)](/assets/images/20240611//permissionless_bitvm_bridge.jpg)

# An In-depth Analysis of BitVM Bridge

## BitVM Bridge Overview

The BitVM Bridge defines the process for transferring BTC between the Bitcoin mainnet and the side system. Users of the Bitcoin network can initiate the BitVM cross-chain protocol to lock a certain amount of BTC in a multisig address on the Bitcoin mainnet, and subsequently mint an equivalent amount of wBTC (Wrapped BTC) on the side system to complete the Peg-in process. The wBTC holder on the side system can also initiate a withdrawal transaction to burn their wBTC holdings and then request an advance of BTC from the liquidity providers specified in the BitVM cross-chain protocol. The liquidity provider then provides a zero-knowledge proof related to the advance, allowing them to retrieve their advanced BTC from the multisig address along with cross-chain transaction fees.

Bitlayer research team has tried hard to fill in the missing pieces and redrawn the graph as below:

[![permissionless_bitvm_bridge_bitlayer](/assets/images/20240705//permissionless_bitvm_bridge_bitlayer.jpg)](/assets/images/20240611//permissionless_bitvm_bridge_bitlayer.jpg)

In the next chapters, we give an in-depth analysis of BitVM bridge protocol.

## Role division

There are five roles in the entire BitVM Bridge.

### Federation: contains several cosigners

The BitVM bridge refers to the classic Layer 2 scheme such as State Channel on Bitcoin, and adopts the method of pre-signing transactions to standardize the format and content of cross-chain bridge related transactions. BitVM sets up a Federation containing n members. On the one hand, it forces participants to follow the protocol through n-of-n multi-signature of n members, and on the other hand, it uses n-of-n multi-signed addresses to temporarily host funds on Bitcoin.

The cosigners in the Federation are summoned for the BitVM Bridge setup ceremony at the protocol launch, generating a multi-signed address controlled by n-of-n signatures to lock Peg-in funds. At the same time, the cosigners collect corresponding information and construct withdrawal transactions and required pre-transactions to unlock funds from this address. After the cosigners disclose the basic information required to generate the transaction and the pre-signed transaction, all cosigners will be required to delete the private key. Subsequent protocol operations only require other participants in the network to supplement information on these pre-signed transactions and broadcast the transaction.

The security assumption of the BitVM Federation is that at least one Federation member has deleted the private key as required, which is the so-called 1-of-n security assumption.

### Deposit User: Alice

After the Federation completes the setup, Alice will check whether all pre-signed results are built according to the BitVM Bridge protocol, and then transfer the BTC that needs to cross the chain to the multi-signed address controlled by the n-of-n signature published by the Federation. Finally, mint out the corresponding number of wBTC assets on the side system.

### Withdrawal User: Bob

Bob initiates a withdraw transaction in the side system , destroys a certain number of wBTC, and provides liquidity providers with information about the withdraw transaction and the fees they are willing to pay, waiting for interested liquidity providers to complete the withdrawal.

### Liquidity Provider: Operator

For security reasons, in the design of BitVM Bridge, all transactions used by cross-chain protocols are determined during the Setup phase. However, it is impossible to determine who the future withdrawal will be during the Setup phase, and it is also impossible to transfer the BTC locked with n-of-n multi-signed addresses to Bob during the pre-signature phase. Therefore, BitVM Bridge adopts a "Advance-Reimbursement" design, which ensures the certainty of transactions by using the liquidity provider Operator role to advance BTC for withdrawal users and then find n-of-n multi-signed addresses for reimbursement (which also stipulates that the amount of reimbursement per transaction is fixed).

During Setup, the Operator needs to submit information to the Federation, which generates and pre-signs withdrawal transactions for each Operator. As a liquidity provider in the network, the Operator pays for Bob when he initiates a withdrawal request. After the Operator honestly completes a payment for Bob's withdrawal transaction, it can retrieve its own payment costs and profits from the multi-signed address.

### Verifier：Anyone

BitVM Bridge uses fault proof: after the Operator initiates the reimbursement process, if no fraud proof is submitted within the dispute time window, the Operator can successfully retrieve the corresponding number of BTC from the Bitvm Bridge multi-signature address. Conversely, if the Verifier detects the illegal reimbursement behavior of the Operator within the dispute time window, the Verifier can challenge the reimbursement transaction initiated by the Operator. Once the Operator cannot provide valid and true reimbursement proof within the specified time, the Verifier will take away the BTC locked by the Operator for initiating reimbursement as a punishment for malicious reimbursement by the Operator.

## Transaction process

### BitVM Setup

When Alice wants to deposit BTC through BitVM Bridge to the side system, the cross-chain protocol starts:
1. Through a certain off-chain protocol, several cosigners are convened to form a Federation. These cosigners can be selected in a decentralized way, or OGs with certain reputation in the community can be invited to participate. After the Federation members are determined, the corresponding multi-signature address for the deposit lock is also determined at the same time.
2. Alice constructs Peg-in Transaction. The content is to transfer 100 BTC to the Federation multi-signed address, hoping to mint the same number of wBTC on the side system (note that Alice will not sign this transaction at this time), and then Alice will send this transaction information to the Federation.
3. Several liquidity providers are convened through some kind of off-chain protocol, which will pay for the withdrawal of users on the subsequent side system and charge a certain fee. Each Operator will construct its own Kick off 1 Transaction , Kickoff 2 Transaction, and Start Time Transaction (see the Key Transaction Types section for a detailed explanation of the transaction types mentioned in this section and below) At the same time, the Challenge Transaction, Kickoff Timeout Transaction, and the second output UTXO of the Kick off 1 Transaction referenced by the Start Timeout Transaction are pre-signed (pre-signed connector UTXO is available for any verifier to challenge). After completion, the Operator sends these transactions/signatures and the basic information required to construct the transaction to the Federation.
4. The Federation verifies the legitimacy of the Peg-in Transaction , all transaction information and signature content/and expressions submitted by participating Operators, and then constructs subsequent Assert Transaction, Disprove Transaction, Disprove Chain Transaction, and Take 1 Transaction, Take 2 Transaction and pre-signs them.
5. After the Federation completes the collection, verification, and pre-signing of all transactions, the Federation will publicize these transactions, signatures, and other basic information for building transactions.
6. Finally, the cosigner will be asked to delete the private key, so as long as an honest cosigner deletes the key, it is impossible to steal the BTC in the multi-signature address through the collusion of all Federation members.

### Alice deposits BTC from the Bitcoin mainnet

1. After the Federation completes the setup, Alice will verify whether all transactions are built according to the BitVM Bridge protocol based on the information published by the Federation, and the pre-signed signature is also included. After confirming the accuracy, she will sign and broadcast the Peg-in Transaction specified in the previous setup.
2. Alice submits the confirmation proof of Peg-in Transaction on the side system. After the side system audits the correctness of the entire BitVM Bridge protocol operation, it mints out 100 wBTC to complete the cross-chain (wBTC will freely circulate on the side system).

### Bob withdraws BTC to the Bitcoin mainnet

1. As the holder of wBTC on the side system, when Bob needs to withdraw BTC from the side system to the Bitcoin mainnet during the schedule, first Bob needs to initiate a withdraw transaction on the side system to destroy 100 wBTC, and at the same time construct a Peg-out transaction on the Bitcoin mainnet (this Peg-out transaction contains two inputs, one input contains 0 BTC, recording Bob's Withdraw transaction on the side system, and the other is waiting for the liquidity provider Operator to add UTXO for advance payment), and the output is 99 BTC transferred to Bob (this transaction also indicates that Bob is willing to pay 1 BTC is used for cross-chain transaction fees. Bob will sign the first input of this transaction and bind it to the withdraw transaction on the side system. The meaning of pre-signing here is that Bob essentially executes an open transaction, "selling" their output to one of the operators. After Bob pre-signs, the operator can compete for this transaction from Bob without interactivity, and the winner can earn profits through the bridge protocol.
2. The operator checked that there was a withdrawal record anchored to the Peg-out transaction in the side system. After evaluating the transaction fee satisfactorily, the operator filled in the input of 99 BTC and broadcasted the transaction to the Bitcoin mainnet. At this point, Bob completed the withdrawal process.

### Operator reimbursement

1. **Kickoff1 Transaction**: After the Operator completes the payment, sign and broadcast the Kickoff 1 Transaction that has been built during the Setup phase to open the reimbursement path. The Operator will lock 2 BTC at this time to ensure that there will be no malicious reimbursement (any identified malicious behavior not only makes the Operator unable to obtain the cost and profit of the payment, but also causes it to lose 2 BTC used for guarantee).
2. **Start Time Transaction**: Operator needs to send a Start Time Transaction within 6 hours after the confirmation of Kickoff1 Transaction to reveal the start time of Bob's withdrawal request (the start time should be slightly earlier than the time when Bob initiated the Withdraw Transaction in the side system), and the Start Time Transaction will also check whether the current BTC time is greater than the start time.
   1. **Start Time Timeout Transaction**: If the Operator does not send a Start Time Transaction within 6 hours to consume the third output of Kickoff1 Transaction, any Verifier can construct a Start Time Timeout Transaction to punish the Operator and terminate the reimbursement process.
3. **Kickoff2 Transaction**: The operator will construct and sign the Kickoff2 Transaction two weeks after the confirmation of the Kickoff1 Transaction, revealing the consensus state of the side system between (start_time, start_time + 2 weeks) (the goal is to anchor a side system state in Bitcoin for subsequent challenge verification: there is an implicit constraint here, the operator must complete the Peg-out advance within two weeks after Bob issues the Withdraw transaction, and this advance transaction should also be processed in the side system within two weeks)
   1. **Kickoff Timeout Transaction**: The Kickoff2 Transaction must be announced as soon as possible two weeks after the confirmation of the Kickoff1 Transaction on the Bitcoin mainnet, otherwise any Verifier can construct the Kickoff Timeout Transaction to punish the Operator and terminate the reimbursement process after two weeks and one day.
   2. **Disprove Chain Transaction**: If the side system state revealed by the Operator in the Kickoff2 transaction is illegal, then any Verifier can construct a Disprove Chain Transaction to reveal the true side system state (one possible implementation is: if the side system is a POW chain, the Verifier can submit a fork with a larger proof of work within the time range of (start_time, start_time + 2 weeks) to initiate a Disprove Chain Transaction) to punish the Operator and terminate the reimbursement process.
4. **Take1 Transaction**: The Operator will wait for the Verifier to verify the submitted side system status and reimbursement behavior after announcing the Kickoff2 Transaction (whether there is a corresponding Bob withdraw transaction in the side system, whether the reimbursement amount and receiving account correspond one-to-one). If there is no Verifier challenge within three days, the Operator will broadcast Take1 Transaction, withdraw 100 BTC (including 99 BTC and 1 BTC handling fee) from the n-of-n multi-signed address, and withdraw 2 BTC previously used for guarantee.
5. **Challenge Transaction**: If the Verifer detects that there is no withdrawal + advance behavior that complies with the protocol within (start_time, start_time + 2 weeks), the Verifier will pay 1 BTC (to prevent malicious challenges, 1 BTC is used to cover the cost of Operator's response to the challenge) to construct and broadcast a Challenge Transaction that consumes the first output of Kickoff1 to initiate a challenge, making it impossible for the Operator to retrieve assets through Take 1 Transaction
   1. **Assert Transaction**: When the Operator recognizes that the reimbursement behavior is challenged, it will initiate an Assert Transaction three days after the confirmation of the Kickoff2 Transaction (reserving the Disprove Chain time) to reveal the zero-knowledge proof of the correct reimbursement and the intermediate variables used by the proof subfunction (guessing that the zero-knowledge proof here verifies whether there is a corresponding Withdraw Transaction on all blocks in the side system exposed by the Operator within two weeks and correctly handles the Peg-out transactions on the Bitcoin mainnet: one possible implementation is that the side system combines Bob's Withdraw Transaction with the corresponding Peg-out transaction on the Bitcoin mainnet and the Kick off1 issued by the Operator Transaction binding, which also solves the double spending problem, because once the Operator cannot reuse the same Kickoff transaction to apply for reimbursement for the same advance behavior)
      1. **Disprove Transaction**: Any Verifier can synchronize Assert Transaction from the Bitcoin mainnet to obtain intermediate variables used to verify the zero-knowledge proof sub-function, and then judge the execution correctness of the zero-knowledge proof verification program locally. Once the Verifier recognizes a sub-function execution error, it can provide the input and output unlocking margin submitted by the Operator to punish the Operator and terminate the reimbursement process.
      2. **Take2 Transaction**: If no Verifier initiates a Disprove transaction within two weeks after the Assert Transaction is confirmed, the Operator will broadcast a Take 2 Transaction, withdrawing 100 BTC from the n-of-n multi-signed address and 2 BTC previously used for collateral.
   2. **Assert Timeout Transaction**: In practice, there should be a Timeout Slash transaction As shown in Path the KickOff2 transaction, so as to avoid the challenge that the Operator does not issue an Assert transaction and keeps 2 BTC locked in the n-of-n address forever (Robin responded that this transaction will be added in the next update).

## Key transaction types

### Take 1/2 Transaction

Take1/Take2 Transaction is the only way for the Federation cosigner to extract BTC from the multi-signed address after deleting the private key. Operators have the motivation to construct and broadcast the pre-transaction of Take1/Take2 Transaction, so that they can retrieve the BTC paid for Bob, the handling fee, and the 2 BTC previously used for guarantee.

### Kickoff 1 Transaction

The Kickoff1 transaction is the initiation transaction of the entire reimbursement process. The transaction includes an input of 2 BTC and three outputs:
1. The address of p2wpkh or p2tr (ensuring that the transaction hash remains unchanged) can be unlocked by the Operator. There are two scenarios for using this UTXO, which are:
   1. Operator will pre-sign this UTXO so that it can be used in the Take1 transaction: use `SIGHASH_ALL` signature Type to lock all inputs and outputs of Take1 cannot be changed. As shown in Path &#9312; in the figure.
   2. Operator will pre-sign this UTXO so that it can be used in Challenge transaction: Use the `SIGHASH_SINGLE` signature Type to lock the first output to transfer 1 BTC to the Operator, allowing the Verifier to add other inputs and outputs to extract the remaining BTC. As shown in Path &#9313; in the figure.
2. P2tr address, which can be unlocked by Operator, including 2 BTC locked by Operator. There are three scenarios for using this UTXO, which are:
    1. Operator can reveal the state of the side system by opening Bitcomitment (Refer to the article: [https://bitvm.org/bitvm.pdf](https://bitvm.org/bitvm.pdf)) two weeks after Kickoff1 Transaction confirmation (possible implementation is: the script checks that the submitted block header is within the specified time and the blocks are continuous, and can be traced back to the genesis block of the side system). As shown in Path &#9314; in the figure.
    2. Operator will pre-sign to unlock the UTXO so that any Verifier can be used in the Kickoff Timeout transaction after two weeks and one day of Kickoff1 Transaction confirmation: Use `SIGHASH_SINGLE` Type to lock the first output and destroy 1.9 BTC (guess to avoid the Operator challenging himself), allowing the Verifier to add other inputs and outputs to extract the remaining BTC. As shown in Path &#9315; in the figure.
    3. Operator will pre-sign the Taptree Time Lock leaf node so that any Verifier can be used with the third output of Kickoff1 Transaction in the Start Timeout transaction: Use `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` signature Type to lock the two inputs of the transaction and the first output of 1.9 BTC. As shown in Path &#9316; in the figure.
3. P2tr address, which can be unlocked by Operator. There are two scenarios for using this UTXO, which are:
    1. Operator needs to reveal the start time of the reimbursement process by opening Bitcomitment within six hours after confirming on Kickoff1. At the same time, the script will verify whether the current Bitcoin mainnet block timestamp is greater than the start time. As shown in Path &#9317; in the figure.
    2. Operator will pre-sign the Taptree unlock condition so that any Verifier can be used in the Start Timeout transaction together with the second output of Kickoff1 six hours after Kickoff1 confirmation: Use the `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` signature Type to lock the two inputs of the transaction and the output of the first 1.9 BTC, allowing the Verifier to add other inputs and outputs to extract the remaining BTC. As shown in Path &#9318; in the figure.

The format and content of Kickoff1 transactions are determined during BitVM Setup. Federation needs to verify whether Kickoff1 transactions lock 2 BTC and generate corresponding three outputs. Federation constructs Take1 transactions based on the first output of Kickoff1

### Start Time Transaction

Start Time Transaction reveals the start time of the withdrawal request by opening Bitcomitment. The format and content of this transaction are determined during BitVM Setup. Federation needs to verify whether the Bitcomitment promised for start time in Start Time Transaction is the same as that promised for Kickoff2 Transaction and Disprove Transaction.

### Kickoff2 Transaction

Operator revealed the status of the side system two weeks after confirmation on Kickoff1. According to the normal process, the side system is highly likely to have completed the synchronization and confirmation of Bitcoin mainnet Peg-out transactions. Therefore, the status of submitting the side system two weeks later can be used to check/verify the existence and correctness of the withdrawal and advance payment process during this period.

The Kickoff2 transaction includes two outputs:
1. The p2tr address that the operator can unlock: The operator can unlock it for building Take1 transactions three days after Kickoff2 confirmation
2. P2tr address, providing n-of-n signature unlocking, including 2 BTC. There are two scenarios for using this UTXO, which are:
    1. Federation will pre-sign this UTXO so that it can be used in Take1 transactions: using `SIGHASH_ALL` signature Type to lock Take1, all inputs and outputs cannot be changed. As shown in Path &#9319; in the figure.
    2. Federation will pre-sign this UTXO so that it can be used in Assert transactions: using the `SIGHASH_ALL` signature Type to lock Assert, all inputs and outputs cannot be changed. Assert requires opening the BitCommitment As shown in Path all intermediate states required to verify zero-knowledge proofs. As shown in Path &#9320; in the figure.
    3. Federation will pre-sign this UTXO so that it can be used in Disprove Chain transactions: Use the `SIGHASH_SINGLE` Type to lock the first output and destroy 1 BTC, allowing the Verifier to add other inputs and outputs to extract the remaining BTC (in this case, the Verifier pays more proof cost, so it can obtain more profits than the Timeout slash case). As shown in Path &#9321; in the figure.

The format and content of Kickoff2 transactions are determined during BitVM Setup. Federation needs to verify whether Kickoff2 transactions use the second output of Kickoff1 and generate two new outputs. Federation will construct Take1 transactions based on the output of Kickoff2.

### Disprove Chain

After the Kickoff2 transaction is issued, once the Verifier finds that the consensus state SB of the Side system exposed by the Operator in the Kickoff2 transaction is not the longest chain , it can obtain a challenge reward of 1 BTC by submitting a correct Side system consensus state SB 'that satisfies SB'weight > SB.weight within the time range of (start_time, start_time + 2weeks) (destroy 1 BTC).

### Challenge Transaction

If there is any objection to the Operator's reimbursement behavior, any Verifier can provide 1 BTC to challenge by unlocking the second output of the Kickoff1 Transaction. Because it is a necessary UTXO for Take 1 redemption, when someone challenges, the Operator must prove its honest behavior through the Assert transaction, and then proceed with the subsequent Take 2 redemption. In order to ensure efficiency and avoid being challenged without cost for each reimbursement, this transaction requires 1 BTC as the challenge cost to be paid to the Operator, which can be used to pay the subsequent Assert transaction fees.

Since anyone can initiate a Challenge Transaction, the Challenge Transaction is not fully determined during BitVM Setup. The Federation only needs to check if the Operator has unlocked the second output of Kickoff1 Transaction by `SIGHASH_SINGLE` signature Type while limiting one of the outputs of the Challenge Transaction to 1 BTC transferred to the Operator.

### Assert Transaction

After the Verifier initiates a Challenge Transaction, the honest Operator is motivated to construct an Assert to attempt to retrieve funds through Take2 in order to obtain the advance payment of BTC, transaction fees, and the previously locked 2 BTC. The Assert Transaction contains an input in which the Operator needs to open the Bitcommitment of the intermediate state of all sub-functions involved in zero-knowledge verification, while generating two new outputs:
1. The p2tr address that the operator can unlock: The operator can unlock it for building Take1 transactions two weeks after Assert confirmation
2. P2tr address, providing n-of-n signature unlocking, including 2 BTC. There are two scenarios for using this UTXO, which are:
   1. Federation will pre-sign this UTXO so that so that it can be used in Take2 transactions: using `SIGHASH_ALL` signature Type to lock Take1, all inputs and outputs cannot be changed. As shown in Path &#9322; in the figure.
   2. Federation will pre-sign this UTXO so that so that it can be used in Disprove transactions: Use the `SIGHASH_SINGLE` signature Type to lock the first output and destroy 1 BTC, allowing the Verifier to add other inputs and outputs to extract the remaining BTC. As shown in Path &#9323; in the figure.

The format and content of the Assert transaction are determined during BitVM Setup. Federation needs to verify whether the Kickoff2 transaction uses the second output of Kickoff1 transaction and generates two new outputs. At the same time, Federation also needs to verify whether the Bitcommitment opened by Assert Transaction is the Bitcommitment used in the intermediate state of the zero-knowledge proof verification sub-function. Federation will construct the Take1 transaction based on the output of Kickoff2.

### Disprove Transaction

Within two weeks of the Assert transaction, if someone discovers that the intermediate state revealed by the Operator in the Assert transaction cannot satisfy the state transition relationship, they can verify the failure of the sub-function execution on the chain by providing the input and output of the corresponding sub-function, thereby unlocking the tapleaf of Taptree 2 and obtaining a challenge reward of 1 BTC (destroying 1 BTC).

## Safety and activity analysis

### Security

The correct execution of the BitVM Bridge protocol transaction process is guaranteed by the federation n-of-n multi-signature. The honest cosigner in the federation will immediately abandon their private keys after the pre-signing is completed. Therefore, as long as one of the nodes is honest, it can ensure that the federation cannot conspire to steal the money in the n-of-n address.

### Activity

The BitVM Bridge activity issue can be viewed from the following two aspects:
- The activity of Federation: The pre-signing of Federation requires everyone to participate. According to Robin's expectation of 1000 Federation members, how to ensure that 1000 members are online and pre-signed correctly is a problem to be solved.
- The activity of the bridge protocol: After the federation completes the pre-signing and publicity, as long as the Kick Off transaction is issued, the bridge protocol will definitely be finalized. Therefore, the activity problem mainly depends on whether the Kick Off transaction can be issued smoothly. The necessary prerequisite for the operator to send the Kick Off transaction is that Alice's Peg-in transaction is successfully finalized. Therefore, if Alice chooses not to send the Peg-in Transaction after the federation pre-signing and publicity, it will cause the protocol to fail to be initiated , resulting in the waste of work before the federation, and there may be potential attacks .

## Financial efficiency

1. Each Operator needs to provide a UTXO (2 BTC in this scenario) for guarantee in the Setup phase. Although this UTXO is not locked, once the Operator uses it for other transactions, the Kickoff1 Transaction will not be successfully put on the chain, and the advance payment reimbursement process cannot be started
2. In each advance reimbursement process, the operator needs to advance m BTC (99 BTC in the figure) and unlock UTXO used for guarantee (2 BTC in this scenario) , and can only get the money back after Take1/Take2 Transaction confirmation . Among them, it takes at least 2 weeks + 3 days to retrieve the funds through Take1 Transaction path , and through Take2 Transaction path it takes at least 4 weeks + 3 days to retrieve the funds.
3. In this article, the amount of each Peg In and Peg Out is 100 BTC. However, Robin mentioned in a previous interview with Bankless that in practical scenarios, the assets locked by Peg-in may be reimbursed multiple times. For example, the Operator can use a smaller granularity such as 10 BTC for a single reimbursement. After the Operator completes a 10 BTC payment and successfully reimburses it (2 weeks + 3 days or 4 weeks + 3 days), the Operator can continue to use the reimbursed UTXO for subsequent Peg-out transactions. However, in this case, the number of transactions that need to be pre-signed each time you set up will also increase.

# Summary

Robin's genius design has brought a more secure and decentralized cross-chain bridge solution to Bitcoin. The community closely cooperates to further optimize costs and improve friendliness, making the protocol continue to evolve . I believe that in the near future, BitVM Bridge will definitely become a crucial basic component in the Bitcoin ecosystem.

Bitlayer is trying to design the first rollup protocol based on BitVM paradigm. Stay tuned!

# References

- [BitVM 2: Permissionless Verification on Bitcoin](https://bitvm.org/bitvm2.html)
- [Is This the Future of Bitcoin?](https://www.youtube.com/watch?v=8HqDykVPypM)
- [Experiment of BitVM White Paper](https://blog.bitlayer.org/Experiment_of_BitVM_White_Paper/)
- [BITVM 2: OPENING UP THE PLAYING FIELD](https://bitcoinmagazine.com/technical/bitvm-2-opening-up-the-playing-field)
- [BitVM: Compute Anything on Bitcoin](https://bitvm.org/bitvm.pdf)
