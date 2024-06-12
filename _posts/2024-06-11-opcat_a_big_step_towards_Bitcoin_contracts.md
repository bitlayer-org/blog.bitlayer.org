---
layout: post
title:  "OP_CAT: A Big Step towards Bitcoin Contracts, From Vault to General Computing"
author: andrew
categories: [ op_cat, bitcoin, matt ]
image: assets/images/common/common.jpg
---


OP_CAT was originally part of the Bitcoin official opcodes, allowing string concatenations on the stack. OP_CAT can concatenate two elements in the stack and push the result back to the stack. OP_CAT opcodes can cause stack elements to grow exponentially, which can cause memory usage to grow exponentially with script size, [ultimately resulting in similar denial of service attack](https://nvd.nist.gov/vuln/detail/CVE-2010-5137). Therefore, Satoshi Nakamoto [removed OP_CAT ](https://github.com/bitcoin/bitcoin/blob/10164916f712bd3c92f0b3ac329ba2e1209746fe/src/script/interpreter.cpp#L456)out of caution on August 15, 2010.

> A simple script that pushes a 1-byte value into the stack and then repeats the script OP_DUP, OP_CAT 40 times will cause the stack value to exceed 1TB in size.

With the passage of time and the development of technology, this issue is no longer an obstacle. Under the Taproot architecture, the size of stack elements is strictly limited to 520 bytes, thus avoiding the above attack methods.

There has always been a big controversy in the Bitcoin community about the issue of re-enabling OP_CAT. On the one hand, OP_CAT can be applied in many scenarios, such as mixed coins, Lightning Network, Bitcoin Layer 2, etc. On the other hand, it is still unclear whether OP_CAT will damage the security of the Bitcoin network and the degree of damage. In addition, the discussion about OP_CAT is no longer limited to adding an opcode, but also includes the debate of whether to enable the "Bitcoin Script Toolbox", such as whether to enable OP_CSFS, OP_CTV and a series of opcodes after enabling OP_CAT. Fortunately, OP_CAT has been enabled on the Bitcoin Signet, which is used to test on a non-POW network, indicating that OP_CAT may be re-enabled on the mainnet in the near future.

This article attempts to interpret the latest progress of OP_CAT-based applications, starting from the two practical proposals, Vault and MATT, analyzing the corresponding prototype system from a technical perspective, exploring and looking forward to the huge empowerment of the Bitcoin ecosystem with OP_CAT in the future.

> Disclaimer: This article only discusses the impact of OP_CAT on smart contracts on Bitcoin, and does not represent a desire to implement all smart contract functions with only OP_CAT, nor does it represent a negation of other newly proposed opcodes. Although this article is positive about the re-enable of OP_CAT, there may still be unforeseen risks to OP_CAT. Bitcoin developers, communities, and miners should decide cautiously after sufficient discussion and testing.

# OP_CAT

In October 2023, Bitcoin Core developer Ethan Heilman and Botanix Labs Chief Software Engineer Armin Sabouri jointly [released a draft ](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-October/022049.html) Bitcoin Improvement Proposal (BIP) called "OP_CAT". The proposal has been tested on the Bitcoin Signet. The BIP redefines the opcode OP_SUCCESS126 and activates the opcode OP_CAT through a soft fork.

We will explain in detail the role of OP_CAT. When executing OP_CAT instructions:

1. Pop two values from the stack.

2. Concatenate the popped-up values together.

3. Then push the connected value to the top of the stack.

If there are fewer than two values on the stack, or the size of the concatenated value exceeds the maximum script element size of 520 bytes, the OP_CAT will fail.

```c++
case OP_CAT:
{
    if (stack.size() < 2) // If the number of elements on the stack less than 2，then fail
        return set_error(serror, SCRIPT_ERR_INVALID_STACK_OPERATION);
    valtype& vch1 = stacktop(-2); // Get the element offset 2
    valtype& vch2 = stacktop(-1); // Get the element offset 1, which is top element.
    // If the combined length of this two elements exceeds 520 bytes, then fail.
    if (vch1.size() + vch2.size() > MAX_SCRIPT_ELEMENT_SIZE) 
        return set_error(serror, SCRIPT_ERR_INVALID_STACK_OPERATION);
     // Set an element offset to 2 on the stack as a concatenation of this two elements.
     vch1.insert(vch1.end(), vch2.begin(), vch2.end());
    stack.pop_back(); // Pop the top element.
}
break;
````

From the above analysis, the implementation of OP_CAT is very simple and easy to understand, but the second half of this article will gradually show readers the power of OP_CAT.

# Implement Vault

In order to better demonstrate the magic of OP_CAT, [Rijindael](https://twitter.com/rot13maxi)  uses OP_CAT to achieve the deposit and withdrawal of the [purrfect vault](https://github.com/taproot-wizards/purrfect_vault). The project purrfect vault fully utilizes the ability of OP_CAT and the Schnorr-signed trick to achieve transaction introspection, ultimately achieving the purpose of checking historical transactions when spending UTXO. In this chapter, we will delve into the implementation of the purrfect vault to give readers a clear interpretation. In addition, it is worth noting again that the Schnorr-signed trick does not require other forks to be introduced to Bitcoin, so the entire implementation of the purrfect vault depends only on OP_CAT.

## Schnorr Signed Trick

To summarize in one sentence, the "Trick of Schnorr Signature" refers to the introspection of transactions that can be achieved by utilizing the structure of Schnorr signature itself and only using OP_CAT. As early as 2021, Andrew Poelstra observed and published an article explaining this. This article briefly explains the principle, and interested readers can refer to [the link to this article](https://medium.com/blockstream/cat-and-schnorr-tricks-i-faf1b59bd298).

First, briefly describe the key generation and signature algorithm of Schnorr signature.

- Key generation algorithm: randomly select a value on a finite field as a private key $x$ , and calculate $P= xG$  based on the elliptic curve on the finite field, where $G$ the elliptic curve generator. The $P$  is exactly the public key.

- Signature algorithm: temporarily select a value $k$  on the finite field, calculate $ R = kG $ , and $ s = k + xH(P \|\| R \|\| \text{data})$ , where $H$ the hash function (i.e., SHA256), $ \text{data} $ is the signed message, $ \|\| $ is the string connection operation. $ R\|\|s $  is the signature.

OP_CHECKSIG is the opcode used in Bitcoin scripts to verify signatures. OP_CHECKSIG checks the validity of the public key and signature on the stack with the transaction itself as a message. Consider the following Bitcoin script:

```javascript
// Inputs
<kG>
<s>
// Locking script
OP_2DUP OP_CAT // [R, s, R||s]
OP_ROT OP_DUP // [s, R||s, R, R]
<G> 
OP_EQUALVERIFY // [s, R||s, R]，check R=G, meaning k = 1
OP_CHECKSIG // R||s as a signature，R as public key (meaning x = 1), check whether signature is valid.

// [s]
````

Running the above script, we force $x = k = 1$ , then the signature algorithm becomes $s = 1+H(G \|\| G \|\| \text{data} )$, that is, $s-1 = H(G\|\|G\|\|data)$ , where $\text{data}$ is the bitcoin transaction itself. In this way, we find that $ s $ only related to the bitcoin transaction itself with the hash function ($ G $  is a constant), based on this we can verify this relationship by $ \text{data} $ inputting it into the stack, so as to check the structure of the transaction itself. But there is a problem here how to calculate the $ s-1 $ , where $ s $ is a signature on bitcoin and cannot be directly arithmetic. Fortunately, we can modify some reserved fields in the transaction and perform this hash operation multiple times so that the last byte of $ s $ is `0x01`, so we can construct two values on the stack: $ s $ and $ s-1 $ by OP_CAT. Specifically, consider the Bitcoin script `<s truncated the last byte> OP_DUP <0x01> OP_CAT OP_TOSTACK <0x00> <OP_CAT> OP_FROMSTACK`, which naturally appears on the stack after running $ s $ and $ s-1 $ .

Now let's take a look at what the trick of Schnorr signature can be done? The following pseudocode is a example to check the output amount of the spending transaction must be exactly 10. In the following pseudocode, we simplify the transaction structure of Bitcoin to highlight two important fields, including the output public key `output_publickey` and the output amount `output_amount`, while omitting fields such as version and lock time, and replacing them with `other_components`. We provide detailed comments, but interested readers can refer to the [Bitcoin script list ](https://en.bitcoin.it/wiki/Script)for comparison and understanding. Using OP_CAT to check Bitcoin transactions includes the following steps:

1. Let the transaction structure itself and the "tricked" signature of the transaction as the witness.

2. In the locking scripts, first check whether specific fields in the transaction structure meet the requirements.

3. Splicing and assembling transactions, using the transaction on the stack to check whether the assembly satisfies the equation $ s = 1+H(G\|\|G\|\|\text{data}) $ ;

4. Make a copy of the signature and use `OP_CHECKSIG` to check the validity of the signature on the transaction itself.

```javascript
// witness
<output_publickey> 
<output_amount>
<other_components>
<signature_truncated_last_byte>

// locking script
<pubkey> OP_TOALTSTACK // move <pubkey> to the bottom of alt stack.
1 OP_PICK // copy output_amount to top of stack.
10 OP_EQUALVERIFY // check output_amount is 10 or not.

OP_CAT
OP_CAT // combine output_publickey、output_amount and other_components to a transaction

OP_SHA256 //  let hashed transaction as "data"（simplify processing abount tag hash）
<G> <G> 2 OP_ROLL OP_CAT OP_CAT // concatenate to G||G||data
OP_SHA256 // calculate H(G||G||data)

OP_TOALTSTACK // move H(G||G||data) to alt stack

OP_DUP // copy <signature_truncated_last_byte> 
<00> OP_CAT //  s-1
OP_FROMSTACK // put H(G||G||data) on top of stack
OP_EQUALVERIFY // check the equation s-1 = H(G||G||data)

<01> OP_CAT // combine <signature_truncated_last_byte> to a valid <signature>
OP_FROMALTSTACK // move <pubkey> to top of stack
OP_CHECKISG // check <signature> with <pubkey>

````

By constructing the above script, we have achieved checking the output amount of the spending transaction in the locking script. It is not difficult to find that other fields of the transaction can be checked in a similar way, such as the public key of a certain output, the number of UTXOs of the transaction output, and so on. To simplify the subsequent description, a pseudo-opcode `op_pick_field ([field])` will be abstracted here. Note that this pseudo-opcode is not a formally defined opcode, but rather an expansion of a piece of code. The parameter of `op_pick_field` is some field of the transaction, which will be placed at the top of the stack after execution.

## UTXO with status

In the previous section, we saw that we can use OP_CAT to check the structure of the spending transaction in the locking script of a UTXO, so that some constraints can be assigned in advance. But can OP_CAT do more? Can we check the transaction that generates UTXO with the same idea of checking the spending transaction? In this section, we will describe a UTXO designed with state.

The following script is a simple example. This script hopes to spend only half the amount of UTXO at a time. It can be clearly seen from the following script.

1. First, the previous transaction was assembled in the stack.

2. Then execute the `OP_SHA256` to obtain the TXID of the previous transaction;

3. Finally, check whether the input of the current transaction is the same as the TXID of the previous transaction.

```javascript
// witness
<pretx_output_publickey> 
<pretx_output_amount>
<pretx_other_components>

// locking script
1 OP_PICK // copy pretx_output_amount to top of stack
OP_ALTSTACK // move pretx_output_amount to alt stack
0 OP_PICK // copy pretx_output_publickey to top of stack
OP_ALTSTACK // move pretx_output_publickey to alt stack

OP_CAT
OP_CAT // let pretx_output_publickey、pretx_output_amount and pretx_other_components 
       // be combined to a transaction.

OP_HASH256 // get TXID
push_u32(0) // push the intput index to stack
OP_SWAP OP_CAT // get full input information

op_pick_field([the first input id])
OP_EQUALVERIFY // check the input on stack is same with the input in the transaction

op_pick_field([the frist output amount])
OP_DUP OP_ADD // double the frist output amount
OP_FROMSTACK // move pretx_output_amount from alt stack to top of stack
OP_EQUALVERIFY

op_pick_field([the first output public key])
OP_FROMSTACK // move pretx_output_publickey from alt stack to top of stack
OP_EQUALVERIFY

````

By assembling the previous transaction on the stack, the various fields of the previous transaction are indirectly obtained, and these fields can be used to check certain fields of the previous and subsequent transactions with the abstracted `op_pick_field` opcodes in the previous section. For example, in the above locking script, there are two constraints. One is to double the first output amount of the current transaction and make it the same as the output amount of the previous transaction. The other is that the output of the previous transaction must be the same as the first output of the current transaction. This ensures that each transaction can only spend half of the amount of the previous UTXO.

Similar to the previous section, we abstracted another pseudo-opcode `op_pick_pretx_field (i, [field])`. Assuming the current transaction uses  UTXO $ u$  as the $ i_{\text{th}} $  input, then this pseudo-opcode can extract a field of the transaction $ u$ .

## Vault prototype

Before introducing the implementation of Vault, let's first define the specific functions and constraints of a Vault. A Vault involves four types of transactions: *Deposit*, *Trigger*, *Complete*, and *Cancel*. First, Vault allows one user to deposit some money into the Vault (*Deposit*), and then the user can trigger a withdrawal transaction (*Trigger*) and the final output address is specified in the Trigger transaction. Next, the user can choose to initiate a new transaction to complete the withdrawal operation (*Complete*), and the money will be sent to the address specified in the Trigger transaction. Alternatively, the user can choose to cancel the transaction and return to the Deposit state (*Cancel*), so that subsequent the user can still trigger the Trigger transaction again.

With the introduction of the previous two sections, you should be able to construct such a prototype by yourself at this point, but we will explain the whole process in detail. You can check if your understanding is correct. The design here refers to the documentation in the [purrfect vault ](https://github.com/taproot-wizards/purrfect_vault), which is not completely consistent. Interested readers should check it.

First, we define the process of Vault precisely according to the following figure. *Vault Taproot* is a Taproot output condition that defines three leaf scripts, and these three leaf scripts contain the unlocking conditions of Trigger transaction, Complete transaction and Cancel transaction respectively. To spend Taproot, one of the leaf scripts must be unlocked. The Deposit transaction itself is the startup transaction that opens the Vault, and only needs to specify the output as *Vault Taproot.*

[![Vault_without_script](/assets/images/20240611/Vault_without_script.jpg)](/assets/images/20240611/Vault_without_script.jpg)

We consider the constraints of different transactions and use the pseudo-opcodes provided in the previous sections to implement these constraints. Note that for simplicity, we describe the constraints on the number of inputs and outputs in the text, but these constraints are not implemented in the pseudo-code. However, referring to the previous sections, these constraints are easy to check by the way when constructing transactions in the stack.

**Constraints of Trigger Transactions**: We need to ensure that Trigger transactions (1) have two inputs and two outputs (2) the first input and the first output have the same amount (3) the address of the first input and the address of the first output are the same. The second input is used to pay gas fees, so we do not restrict the second input.

**Constraints of Complete Transactions**: Complete transactions are more complex than Trigger transactions and require checking the content of the previous transaction. Therefore, the constraints it requires include (1) having two inputs and one output (2) the previous transaction is indeed a Trigger transaction, which can be obtained through some markers, such as only Trigger transactions have two outputs among all transactions in Vault (3) the first output amount of the previous transaction is consistent with the unique output amount of the current transaction.

**Cancel transaction constraints**: Cancel transaction constraints are relatively simple, it only needs to ensure that (1) there are two inputs and one output (2) the amount of the first input is the same as the amount of the output.

Below is the Bitcoin script pseudocode that describes these constraints. This basically translates the text description above clearly, so no redundant comments are added. The input and output represent the fields of the corresponding UTXO.

```javascript
// trigger leaf
op_pick_field(amount of input1)
op_pick_field(amount of output1)
OP_EQUALVERIFY

op_pick_field(pk of input1)
op_pick_field(pk of output1)
OP_EQUALVERIFY

// compelete leaf
op_pick_pretx_field(0, amount of input1)
op_pick_field(amount of output)
OP_EQUALVERIFY

// cancel leaf
op_pick_field(amount of input1)
op_pick_field(amount of output)
OP_EQUALVERIFY
````

Below figure is just put the pseudocode on the leaf of *Vault Taproot*.

[![Vault_with_script](/assets/images/20240611//Vault_with_script.jpg)](/assets/images/20240611//Vault_with_script.jpg)

# Implement General Computing (MATT)

In the previous chapter, we introduced how to implement a Vault, for which we described how to implement transaction introspection, how to constrain transaction fields, and how to design a UTXO with state. Using this knowledge, in this chapter we will introduce [MATT (Merkleize All The Things) ](https://merkle.fun/), which is a proposal to implement a universal finite-state machine through the challenge-response paradigm. The specific details of this chapter mainly refer to the implementation of [pymatt ](https://github.com/Merkleize/pymatt)(still a demo), but it does a lot of simplification. It should be noted that the universal finite-state machine that only uses OP_CAT is still Work In Progress (CatVM).

Pymatt requires an additional opcode OP_CCV that is not enabled to assist with the implementation, but OP_CAT can fully emulate OP_CCV functionality, which we will explain in a moment. Therefore, the general-purpose fine-state machine MATT can be implemented only with OP_CAT. But, as the claimer at the beginning of this article says, this article only discusses the impact of OP_CAT on smart contracts on Bitcoin, and does not represent a desire to implement all smart contract functions with only OP_CAT, nor does it represent a negation of other newly proposed opcodes. 

## OP_CCV

In order to comply with the description method of MATT proposal, this section first introduces OP_CCV opcode, and then gives a OP_CAT implementation of OP_CCV. The full name of OP_CCV is OP_CHECKCONTRACTVERIFY, which can be generated by Taproot Output Key to transfer data between UTXO, the following specific explanation of how to do this.

The Output Key specified by a Bitcoin transaction UTXO using the Taproot feature is not a real public key, but is formed by combining the Internal key and Taptree through the "Taproot Tweak" process. Users can spend this UTXO by giving the corresponding signature of the Internal Key, or by unlocking a leaf node (TapLeaf) in the Taptree. Sometimes the Internal Key is not a valid public key, and the UTXO can only be spent by unlocking the Taptree. For more features of Taproot, interested readers can refer to the [interpretation](https://github.com/bitcoinops/taproot-workshop/blob/master/2.4-taptree.ipynb).

[![taproot_with_opccv](/assets/images/20240611//taproot_with_opccv.jpg#pic_center)](/assets/images/20240611//taproot_with_opccv.jpg#pic_center))

OP_CCV feature combines optional 32-byte component *data* outside the Internal key and Taptree to form the Output Key, as shown in the figure above.

```c++
// taproot
output_key = taproot_tweak(internal_key, taptree)

// taproot with OP_CCV
output_key = taproot_double_tweak(internal_key, taptree, data)
````

OP_CCV can specify the input or output number of the current transaction to verify the *data*. Note that the *data* in the UTXO of an input of a transaction is actually the *data* of an output UTXO of the previous transaction, which realizes the transfer of 32 bytes of data between transactions. OP_CCV actually contains a series of flags to define the checked fields, but for simplicity, we only describe two fields data and the index of the input or output.

```javascript
<data> // 32 byte data
<index> // In the formal definition of OP_CCV, the index represents input or output is defined by a flag.
        // This article simplify the flag，and descript it with words.
OP_CCV // Check the data is included in output_key.
````

How to use OP_CAT to simulate OP_CCV and achieve the purpose of passing 32 bytes? Thanks to Salvatore Ingala's insight, as we can see in the previous chapter on implementing Vault by OP_CAT, when spending UTXO, the Bitcoin script can not only check its own fields, but also check a field of the previous transaction. You can generate an additional UTXO with amount 0 and *data* as public key in the previous transaction, then the *data* can be checked by subsequent transaction. 

For the sake of simplicity and consistency with the MATT proposal, OP_CCV opcodes will be used later, rather than OP_CAT simulation, but remember that OP_CAT can always be used to simulate OP_CCV.

## Status commitment

Similar to Ethereum or other blockchain systems with state, the concept of State Root is widely rooted in people's minds. MATT also adopts this concept, which summarizes all data that needs to be stored and used into a 32-byte state root through Merkle commitment. Whether it is obtaining or modifying the state, it is completed through the Merkle proof of the state root. Here, it is emphasized that by using the state root, UTXO can directly pass any number of states, which means that applications based on it are no longer limited by the size of the transmitted state.

## State transition commitment

Currently, the standard size of a Bitcoin script is 400k (non-consensus rule but a common practice), but an application may far exceed this size. At this point, using the ability of taproot to split the application into smaller scripts may be a better choice. Specifically, assuming the purpose of the application is to calculate $ y = f(x)$ , where $ x$ and $ y$ are the state roots introduced in the previous section. This process can be split into different steps $ x_1 = f_0(x_0), x_2 = f_1(x_1) ..., x_n = f_{n-1}(x_{n-1}) $ , where $ n$ is the number of all steps and $ x_i, i\in [n]$ is all intermediate states, and $ x = x_0$ , $ y=x_n$ .

With such a splitting rule, MATT (1) takes all intermediate state roots $ x_0, ..., x_{n-1} $ and commits them as a state transition tree $ t $ (Merkle Tree) (2) takes all intermediate computation functions $ f_0, ..., f_{n-1} $ and aggregates them into an execution tree through the capabilities of Taproot, where each leaf node executes only one intermediate computation function.

To better define the state transition tree, first assume the existence of $ w$ such that $ n= 2^w $ . This assumption is trivial, because you can always fill in functions that do nothing $ f_{\text{dummy}}(x) = x  $ so that $ n$ such conditions are satisfied. The state transition tree is defined as a full binary tree of $ w $  layers. The state transition tree $ t $ on each node (including leaf nodes and branch nodes) is defined $  t(i, j) $ , where $ i $ and $ j $ are indexes of an intermediate state. When using this concept later, we always make sure $ t(i, j) $ must be some node on the state transition tree. So, simply put, $ t(i, j) $  is the root of the subtree composed of these intermediate states from $ x_i$ to $ x_j $ , and when $ i = j$ , $ t(i, j)$ represents a leaf node. 

## Binary challenge protocol

The first two sections respectively perform Merkle tree operations on state, state transition, and splitted functions, which are used three times in MATT, which is the source of [Merkleize All The Things](https://merkle.fun/). The following figure explains how these three Merkle trees implement arbitrary calculations through the challenge-response paradigm. We will first explain the meanings represented by each element in the figure below, and then describe the entire challenge process and corresponding constraints. We will implement these constraints in the next section using the opcodes defined by OP_CAT. Now let's first explain what the elements in this figure mean.

Each dashed box in the figure represents a Bitcoin transaction, the solid line outside the dashed box represents the correspondence between the Bitcoin transaction input and output, and the thick solid line inside the dashed box represents the UTXO cost relationship. For simplicity, each Bitcoin transaction in the figure has only one input and one output, which omits the gas fee; if only the OP_CAT is used, the output for state transmission is also omitted (see section "OP_CCV"). In each Bitcoin transaction, the upper right corner represents the signer of the transaction. The blue box represents the output and corresponding spending conditions of each Bitcoin transaction, where "or" represents the use of Taproot's "or" relationship to combine different Bitcoin scripts. In addition, to ensure the continuity of the challenge process, a timeout Bitcoin script that can be directly unlocked by the counterparty is needed, but omitted here for simplicity.

Although there are some variables expressed by mathematical formulas in the figure, roughly speaking, the script content contained in all blue outputs is fixed, that is, from the first transaction, it has been guaranteed that the subsequent series of transactions must comply with the transaction specifications in the figure below. For example, the unlock script Start_Challenge this output requires that the output of the transaction that costs it must be Alice_Reveal, which can be achieved through the introspection of the OP_CAT introduced in the previous section. In addition to formatting the transaction, this challenge also requires passing some state, namely the gray box representing the unlock script in each Bitcoin transaction, where the variables represent the input required for UTXO unlocking, and the first line of these variables represents the constrained *data*, which is the *data* passed between UTXOs.

[![matt](/assets/images/20240611//matt.jpg)](/assets/images/20240611//matt.jpg)


Now let's describe the process of the binary challenge protocol. Alice and Bob are two parties in the challenge-response process. Alice declares in advance $ f(x) = y $ and the state transition tree in the calculation process $ t(0, n-1) $ , that is, the root of the Merkle tree composed of $x_0, ...x_{n-1}$, and collateralizes somey money (*Delcare Tx*). But Bob, as the counterparty, believes that $ f(x) \neq y $ and initiate the challenge. Another result $y'$ and the state transition tree composed of another different state transition list $$x'_0, ..., x'_{n-1}$$, $t'(0, n-1)$ need to be provided (*Start Challenge Tx*). Next is a recursion process. For the state transition tree provided by Alice before $ t(i, j) $ (if it is the first round of recursion, then $ i = 0, j = n-1 $ ), Alice always reveals the state of the middle node of the state transition tree $ x_m $ , the root of the left subtree $ t(i, m) $ and the root of the right subtree $ t(m, j) $ (*Alice Reveal Tx*), and Bob observes these values and compares them with the state transition tree he constructed, providing different roots of the left subtree $ t'(i, m) $ (*Bob Reveal Left Tx*) or the root of the right subtree $ t'(m, j) $ (*Bob Reveal Right Tx*). Continue this recursion process until Bob finds $ m $ , such that $$ x_m = x'_m $$ and $$ x_{m+1} \neq x'_{m+1} $$ . At this point, Bob can prove by executing the corresponding function script $ f_m(x_m) \neq x'_{m+1}$ , and finally take Alice's collateralized Bitcoin (*Leaf Tx*).

Next, we will briefly describe the constraints required for several transactions in the figure, omitting the constraints on the transaction structure (according to the introduction in the previous chapter, it is not difficult to constrain the output address of the next transaction), mainly describing the unlocking conditions of the output of each Bitcoin transaction, and implementing it in the next section.

- *Declare Tx* is the first transaction signed by Alice, which declares that Bob can start the challenge protocol. `Start_Challenge` is the output of the *Declare Tx* transaction, where *data* in `Start_Challenge` is assembled by $$ x, y, t(0, n-1) $$ Merkle tree. Unlocking `Start_Challenge` requires the witness to provide not only *data*, but also the result that the challenger thinks is correct $$ y', t'(0, n-1) $$ . Here $$ t $$ and $$ t' $$ will be used in subsequent recursions, and the expressions are transformed to $$ t_{i, j} $$ and $$ t'_{i, j} $$ , where $$ i =0, j = n-1 $$ .

- *Start Challenge Tx* is a transaction signed by Bob, which initiates the challenge protocol and requires Alice to input an intermediate state $$ x_m $$ , where $$ m = (i+j)/2 $$ . `Alice_Reveal` is the UTXO output of the *Start Challenge Tx* transaction, where the *data* needs to be $$ x_{i}, x_{j}, x'_{j}, t_{i, j}, t'_{i, j} $$ assembled through the Merkle tree. `Alice_Reveal` also requires $$ x_{m}, t_{i, m}, t_{m, j} $$ to satisfy $$ t_{i, j} = H(t_{i, m}, t_{m, j}) $$ , where $$ H $$  refers to the hash function in the Merkle algorithm (i.e., SHA256).

- *Alice Reveal Tx* is a transaction signed by Alice. Its witness reveals the intermediate state and the roots of the left and right subtrees. Its output is a Taptree with two leaf scripts, and Bob can choose to spend either one. We take `Bob_Reveal_left` as an example, where *data* is $$ x_{i}, x_{j}, x'_{j}, t_{i, j}, t'_{i, j}, x_{m}, t_{i, m}, t_{m, j} $$ assembled through a Merkle tree. `Bob_Reveal_left` requires witness to provide elements in *data*, it also requires $$ x'_{m}, t'_{i, m}, t_{m, j} $$ , where $$ t'_{i, j} = H(t'_{i, m}, t_{m, j}) $$ .

- *Bob Reveal left Tx* is a transaction signed by Bob, its witness reveals the interval of the next challenge, and its output is a Taptree with $$ n $$ leaf scripts, one of which is to return to *Alice Reveal Tx* to continue exposing, and the other leaf scripts correspond to $$ f $$ the distribution function $$ f_0, ..., f_{n-1} $$ , and the *data* of each leaf script will be $$ x_m, x_{m+1}, x'_{m+1} $$ . A step-by-step script checks the witness to ensure that the data in the *data* is provided, and verifies $$ f_m(x_m) = x'_{m+1} $$  with the values in the *data*.

- *The Bob Reveal Right Tx* transaction is similar to the *Bob Reveal Left Tx* transaction process, which is omitted here.

- *The Leaf Tx* transaction is a transaction signed by Bob. As a transaction that terminates the loop, its output no longer requires specific constraints and can be defined by Bob.

## Constraint implementation

In the previous section, we introduced the binary challenge protocol and corresponding constraints. Similar to the implementation of Vault, in this section, we implement these constraints through some pseudocodes. For simplicity, the process of checking input and output data is only expressed relatively completely in the `Start_Challenge` script, while other scripts omit this process. In addition, we use a pseudo-opcode OP_MERKLE to represent the root of the Merkle Tree generated for a series of elements, which is easily achieved through OP_CAT and existing OP_SHA256. In addition, OP_5DUP is also easily implemented through other opcode combinations.

```javascript
// impl start_challenge：
OP_5DUP // copy all witness x, y, t_{0, n-1}, y', t'{0, n-1}
OP_TOALTSTACK * 5 // move to alt stack

OP_TOALTSTACK OP_TOALTSTACK // move y', t'{0, n-1} to alt stack
OP_3DUP // copy x, y, t_{0, n-1}
OP_MERKLE // construct data
<first input>
OP_CCV // check the binding data in the first intput

OP_FROMALTSTACK
OP_EUQALVERIFY // check t_{0, n-1} != t'{0, n-1}
OP_FROMALTSTACK
OP_EUQALVERIFY // check y != y'
OP_DROP // clear x 

OP_FROMALTSTACK * 5 // move all witness to top of stack
OP_MERKLE // construct next state
<first output>
OP_CCV // check the binding data in the first output
// ------------------------------------------------------------------
// impl Alice_Reveal：
// omitted input check
OP_MERKLE // H(t_{i, m}, t_{m, j})
3 OP_PICK // t_{i, j}
OP_EQUALVERIFY // check t_{i, j} = H(t_{i, m}, t_{m, j})
// omitted output check
// ------------------------------------------------------------------
// impl Bob_Reveal_left：
// omitted input check
OP_DUP // copy t_{m, j})
4 OP_PICK // copy t_{m, j} in the data
OP_EQUALVERIFY // check euqal
OP_MERKLE // calculate H(t'_{i, m}, t_{m, j})
5 OP_PICK // copy t'_{i, j}
OP_EQUALVERIFY // check euqal
// omitted output check
// ------------------------------------------------------------------
// impl Bob_Reveal_right：
// omitted
// ------------------------------------------------------------------
// impl m_th Leaf：
// omitted input check
2 OP_ROLL // put x_m to top of stack
<f_m> // f_m
OP_DUP OP_TOALTSTACK // save copy of f_m(x_m) to alt stack
OP_EQUALVERIFY // check f_m(x_m) = x'_{m+1}
OP_TOALTSTACK OP_NOTEQUALVERFIFY // check f_m(x_m) ！= x_{m+1}
// omitted output check
````

# Summary

As mentioned in the introduction of the previous chapter, [a general finite-state machine using only OP_CAT is still under development ](https://www.youtube.com/watch?v=U5qcL0hI30k&t=10784s&ab_channel=bitcoin%2B%2B), but this article roughly demonstrates that OP_CAT can implement a general finite-state machine by interpreting two proposals related to OP_CAT and corresponding prototypes ( [purrfect_vault ](https://github.com/taproot-wizards/purrfect_vault)and [pymatt ](https://github.com/Merkleize/pymatt)). For Rollup, there are already teams [implementing STARK verification based on OP_CAT ](https://github.com/Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark). For Web3 applications, there are already tools on the Bitcoin chain (such as [sCrypt ](https://scrypt.io/)) that have implemented some application frameworks based on OP_CAT and have some attractive application prototypes. Another Bitcoin application related to this article is [BitVM ](https://bitvm.github.io/). Although BitVM aims to achieve on-chain general computing without OP_CAT enabled, the correctness of transaction format and state transmission still relies on third-party multi-signature. Enabling OP_CAT can help BitVM provide more efficient optimization in terms of script scale, transaction format specification, etc. It can be said that the enable of OP_CAT will bring a big outbreak of applications on Bitcoin. Let's wait and see.

# Reference

- [OP_CAT Proposal for Restoration](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-October/022049.html)
- [Implementing purrfect_vault prototype with OP_CAT](https://github.com/taproot-wizards/purrfect_vault)
- [MATT: Implementing Universal Smart Contracts through Contracts in Bitcoin](https://merkle.fun)
- [Implementing MATT Prototypes with Python](https://github.com/Merkleize/pymatt)
- [Explanation of purrfect_vault and MATT in Bitcoin ++](https://www.youtube.com/watch?v=U5qcL0hI30k&t=10784s&ab_channel=bitcoin%2B%2B)
- [CatVM: Universal fine-state machine through OP_CAT](https://catvm.org/catvm.pdf)
- [Salvatore Ingala's slides on MATT](https://docs.google.com/presentation/d/1VCHJhXXzjn3qggQfNTZ3Ikiwi4QaXQYZzqkAea-QCBc/edit#slide=id.g2124bfa8973_0_15)
- [Implementing a validator for Circle STARK using Bitcoin scripts](https://github.com/Bitcoin-Wildlife-Sanctuary/bitcoin-circle-stark)
- [Simplicity: A blockchain programming language to replace Bitcoin scripting](https://github.com/BlockstreamResearch/simplicity)
- [sCrypt: a full stack Web3 development platform for Bitcoin compatible blockchain](https://scrypt.io)
- [BitVM Proposal: A Turing Computationally Complete Bitcoin Contract](https://bitvm.org)
- [Official implementation of BitVM2 running SNARK validator](https://github.com/BitVM/BitVM)

