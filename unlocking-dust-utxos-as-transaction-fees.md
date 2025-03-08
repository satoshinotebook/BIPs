# Unlocking Dust UTXOs as Transaction Fees

## Abstract

This BIP proposes a mechanism to repurpose "dust" UTXOs (unspent transaction outputs of very low value) as transaction fees. Under the current Bitcoin protocol, dust UTXOs present a challenge as their value is often less than the cost of spending them, effectively rendering them economically unviable. This proposal introduces a standardized way for users to voluntarily and explicitly designate dust UTXOs as transaction fees, allowing miners to claim them directly without requiring the dust to be included as a traditional input. 

For users, this provides a pathway to reclaim economic value from otherwise stranded Bitcoin without paying additional fees. For miners, it offers an additional source of revenue that becomes increasingly valuable as block rewards diminish. For the network, this approach improves Bitcoin’s fungibility, and helps clean up the UTXO set by reclaiming economically stranded Bitcoin while providing miners with additional fee income.

It is crucial to emphasize that this mechanism is entirely opt-in and user-initiated, requiring explicit cryptographic authorization from the UTXO owner. No dust UTXO can be claimed without the owner's digital signature specifically authorizing its use as a fee. Historical proposals were criticized for perceived forced reclamation. This proposal avoids that entirely by requiring voluntary user authorization. This proposal enhances Bitcoin's fungibility and practical spendability by allowing voluntary, secure, cryptographic authorization to reclaim dust UTXOs. The economic and systemic benefits to node operators, miners, and users are significant, especially in the long-term growth and sustainability of the network.

## Copyright

This BIP is licensed under the BSD 2-Clause License.

## Motivation

Bitcoin's UTXO model, while powerful, has created an unintended consequence in the form of economically unviable "dust" UTXOs. These are outputs with such small values that the transaction fee required to spend them would exceed their worth. This presents several problems for the Bitcoin network:

1. **UTXO Set Bloat**: Dust UTXOs contribute to the growth of the UTXO set without providing economic utility, increasing the resource requirements for full nodes. According to analyses, a significant fraction of all UTXOs are dust or uneconomical outputs, with this proportion growing as fees increase. By some estimates, removing 1 million dust UTXOs could reduce the UTXO set by several megabytes, benefiting node operators and overall network efficiency.

2. **Stranded Value**: Small amounts of Bitcoin become economically inaccessible to their owners, effectively removing them from circulation. Over time, this creates a growing "dust horizon" - outputs that cost more to move than they're worth. For users, this means having Bitcoin balances that are visible but practically unusable.

3. **Inefficient Fee Market**: Users with dust UTXOs have no economical way to consolidate or spend them, even during periods of low fee pressure. This leads to inefficient capital allocation within the Bitcoin ecosystem.

4. **Weakened Fungibility**: When some satoshis become economically unspendable due to their positioning in dust UTXOs, it weakens Bitcoin's property of fungibility. Every satoshi should ideally be equally usable.

5. **Growing Problem**: As Bitcoin's value increases and transaction fees rise over time, the threshold for what constitutes "dust" also rises, potentially stranding larger amounts of Bitcoin. This proposal proactively addresses a problem that will likely become more significant in the future.

Currently, there is no standard mechanism for users to reclaim value from dust UTXOs when the economics don't justify a traditional transaction. This BIP addresses this gap by allowing dust UTXOs to be repurposed as transaction fees through a secure mechanism that doesn't require exposing private keys.

## Specification

### Overview

This proposal introduces a new transaction output type that allows users to designate specific dust UTXOs as transaction fees through a special output script. Rather than requiring users to spend dust UTXOs through traditional inputs (which would cost more in fees than the dust is worth), this mechanism allows miners to claim these UTXOs directly when they mine a block containing the designating transaction.

### Protocol Changes

The implementation of this proposal requires changes at several layers of the Bitcoin protocol:

#### Consensus Rules

1. **New Output Type Recognition**: The protocol must recognize a special output script pattern that designates dust UTXOs for miner claiming.

2. **Miner Claiming Authorization**: When miners include a transaction with a valid dust fee designation in a block, the consensus rules permit them to include the referenced dust UTXO as an input to their coinbase transaction without satisfying the normal spending conditions.

3. **Coinbase Transaction Validation**: The consensus rules must be updated to validate that miners only claim dust UTXOs that have been properly designated in transactions included in the same block.

#### Relay & Mempool Policy

1. **Relay Adjustments**: Nodes must be updated to relay transactions with dust fee designations, even if their standard fee rate would otherwise be below minimum relay thresholds.

2. **Mempool Acceptance**: Dust fee designation transactions should have special mempool acceptance criteria that recognize their unique nature.

3. **Rate Limiting**: To prevent DoS attacks, nodes may implement policies limiting the number or size of dust fee designation transactions.

### Dust UTXO Designation Mechanism

#### Output Script Structure

A dust UTXO is designated as a fee offering through a new output script pattern:


OP_RETURN <DUST_FEE_PREFIX> <dust_utxo_txid> <dust_utxo_vout> <signature>

Where:
- `DUST_FEE_PREFIX` is a standardized identifier for dust fee offerings (e.g., "DUSTFEE")
- `dust_utxo_txid` is the transaction ID of the dust UTXO being offered
- `dust_utxo_vout` is the output index of the dust UTXO
- `signature` is a valid signature using the private key that controls the dust UTXO, proving ownership without exposing the private key

Alternatively, this could be implemented using a new SegWit witness program version that encodes the same information, providing better efficiency in terms of transaction weight.

#### Transaction Examples

Here's an illustrative example of how a dust fee designation transaction would be structured:


Transaction ID: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
Input[0]: txid: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa vout: 0 scriptSig: <signature> <pubkey>
Output[0]: value: 0 scriptPubKey: OP_RETURN DUSTFEE bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb 0 <signature>

The corresponding coinbase transaction that claims this dust:


Coinbase Transaction: Input[0]: (Regular coinbase input with no previous outpoint) Input[1]: txid: bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb vout: 0 (No scriptSig required - claimed based on dust fee designation)
Output[0]: value: 6.25 BTC + normal transaction fees + dust UTXO value scriptPubKey: (Miner's address)

#### Wallet Behavior

Wallets supporting this BIP will:

1. Identify dust UTXOs in the user's wallet that are economically unviable to spend
2. Allow users to designate these UTXOs as transaction fees
3. Generate the appropriate output script with proper signatures
4. Include this output in a transaction alongside other normal inputs and outputs
5. Explicitly disable RBF (Replace-By-Fee) for dust-fee designation transactions to prevent double-spend or race conditions, enhancing network security and miner certainty

Wallets should provide clear UX informing users that once a dust UTXO is designated as a fee, it will no longer be under their control. Each dust UTXO can only be designated and claimed once. After claiming, the dust UTXO is permanently spent and cannot be reused in subsequent transactions.

**UX Considerations:**
- Wallets should clearly indicate to users that designating dust as fees permanently surrenders control of those funds
- The interface might include text such as: "You are about to donate 2,500 sats in dust to miners. This action cannot be reversed once confirmed."
- Wallets could provide an estimate of how much UTXO consolidation would cost versus using this dust-fee mechanism
- An option to batch multiple dust UTXOs in a single dust-fee transaction would improve efficiency

### Miner Considerations

Miners supporting this BIP will need to implement additional logic to handle dust fee designations:

1. **Basic Validation and Claiming Process**:
   - Scan transactions for dust fee designation outputs
   - Validate the signatures to ensure the designator owns the dust UTXO
   - Verify that the referenced UTXO exists and is unspent
   - Claim these UTXOs as additional fees when mining a block, adding them as inputs to the coinbase transaction
   - Ensure that all claimed dust UTXOs correspond to valid designation transactions within the same block

2. **Coinbase Transaction Size Limits**: To prevent oversized coinbase transactions, miners are recommended to claim no more than 100 dust UTXOs per block. This limit would add approximately 3,200 bytes to the coinbase transaction, still well within acceptable limits while preventing excessive bloat.

3. **Block Propagation Impact**: Claiming many dust UTXOs will slightly increase block size due to the larger coinbase transaction. This has minimal impact on block propagation time under normal circumstances, but miners may choose to limit dust claims during periods of network congestion.

4. **Validation Overhead**: Processing blocks with many dust claims requires additional validation work. Miners should optimize their validation code to handle batches of dust claims efficiently.

5. **Prioritization Strategy**: Miners may implement various strategies for selecting which dust fee designations to include when there are more available than the recommended limit. Possible strategies include:
   - Highest dust value first
   - Oldest transaction first
   - Random selection (to ensure fairness)
   - Combination of age and value

#### Dust UTXO Claiming Mechanism

The key innovation in this BIP is allowing miners to claim dust UTXOs without requiring private key transfer. This works through a consensus rule change:

1. When a miner includes a transaction with a valid dust fee designation in a block, the consensus rules permit them to include the referenced dust UTXO as an input to their coinbase transaction **in the same block**. This temporal restriction is critical - miners can only claim dust UTXOs in the exact same block that includes the designation transaction.

2. The miner's ability to claim the dust UTXO is granted by the new consensus rule, not by possessing the private key. This is similar to how miners currently claim transaction fees without needing the private keys of the inputs.

3. The signature in the OP_RETURN output serves as proof of authorization from the dust UTXO owner, effectively saying "I authorize any miner who includes this transaction to also claim this specific dust UTXO."

4. The coinbase transaction claiming the dust UTXO doesn't need to satisfy the original spending conditions of the dust UTXO; instead, it satisfies the new consensus rule that recognizes the dust fee designation as valid authorization for claiming.

5. When a block includes transactions with dust-fee outputs totaling N satoshis, the miner is allowed to increase their coinbase output by N satoshis (effectively claiming those fees).

6. The consensus rules explicitly verify that any claimed dust UTXO corresponds to a valid designation transaction in the same block, preventing miners from claiming arbitrary UTXOs.

#### Mempool Policy Changes

For this mechanism to function properly, the following mempool policy changes are required:

1. Allow transactions with dust fee designation outputs to propagate, even if the effective fee rate (considering only regular fees, not the dust) would be below the typical minimum relay fee.

2. Recognize dust fee designation outputs as a special category that does not count toward the transaction's regular outputs for fee calculation purposes.

3. Implement rate-limiting for dust fee transactions to prevent potential DoS attacks. Nodes might adopt a rate limit of 10 dust fee designation transactions per minute per peer connection, adjustable based on network conditions.

Nodes could adopt adaptive policies such as:
- Accepting dust-designation transactions only if mempool memory usage is below a configurable threshold (e.g., 75% of max mempool size)
- Including such transactions based on configurable acceptance criteria like time-in-mempool or dust value thresholds
- Prioritizing dust fee designations during periods of low mempool congestion
- Limiting the number of dust fee designation transactions from a single source within a time window

These policy adjustments balance the need to facilitate dust cleanup with protecting the network from potential abuse.

### Security Considerations

This mechanism preserves the security of private keys by:

1. Using a signature-based verification system similar to Bitcoin's existing transaction validation
2. Never requiring the exposure of private keys
3. Ensuring that only the rightful owner of a dust UTXO can designate it as a fee

The security model follows Bitcoin's existing paradigm where authorization to spend (or in this case, designate as a fee) comes from cryptographic proof of ownership rather than direct private key transfer. This maintains the critical security properties of Bitcoin while enabling new functionality for dust UTXOs.

#### Security Model for Signatures

The signature provided in the dust fee designation script proves ownership but doesn't expose the private key. This security model works as follows:

1. The signature is created using the private key that controls the dust UTXO
2. Nodes and miners validate this signature against the referenced UTXO's scriptPubKey
3. A valid signature cryptographically proves that the user authorizing the dust fee designation is the rightful owner of the dust UTXO
4. This is analogous to how normal transaction signatures work but applied to a different purpose

#### Double-Spend Protection

Once a dust UTXO is designated as a fee and included in a block, it is immediately marked as spent in the UTXO set, preventing subsequent attempts to spend it. Each dust UTXO can only be designated and claimed once. After claiming, the dust UTXO is permanently spent and cannot be reused in subsequent transactions.

If a user attempts to double-spend a dust UTXO after designating it as a fee but before it's mined:

1. Whichever transaction confirms first will prevent the other from being valid
2. By requiring transactions be marked as non-replaceable (explicitly disabling RBF), we reduce the risk of users accidentally creating conflicting spends
3. Miners will follow the first-seen rule for conflicting spends in most cases

#### Potential Abuse & Mitigations

Several potential abuse vectors exist:

1. **Spamming the Network**: Users could create many tiny dust UTXOs and then generate a flood of dust fee designation transactions. This is mitigated by:
   - Rate-limiting dust fee designation transactions at the node level
   - Setting size or quantity thresholds for dust fee transactions
   - Requiring a minimum fee for the dust fee designation transaction itself

2. **Denial of Service**: Attackers might try to overload miner validation logic by creating complex dust fee designation transactions. This is mitigated by:
   - Standardizing the dust fee designation format
   - Setting clear limits on validation resources
   - Implementing timeouts for verification operations

### Dust Threshold

For the purposes of this BIP, a "dust" UTXO is defined as any UTXO whose economic value is less than the cost of spending it in a typical transaction. This threshold is not hard-coded but is rather determined by the economic reality of current fee rates. Wallets may implement their own heuristics to identify dust UTXOs based on current network conditions.

## Rationale

### Design Decisions

Several approaches were considered for addressing the dust UTXO problem:

1. **Direct Private Key Transfer**: Initially, transfer of the private key to miners was considered but rejected due to significant security risks and the need for a complex secure communication channel.

2. **OP_RETURN Metadata**: The chosen approach uses OP_RETURN metadata with cryptographic proof of ownership, which maintains security while enabling the functionality.

3. **New Transaction Type**: A completely new transaction type was considered but would require a more invasive hard fork. The OP_RETURN approach can be implemented as a soft fork.

4. **Off-Chain Coordination**: Solutions involving off-chain coordination between users and miners were considered but rejected due to centralization concerns and increased complexity.

### Soft Fork vs. Hard Fork Considerations

This proposal is designed as a soft fork to maintain backward compatibility while adding new functionality:

1. **Backward Compatibility**: The use of OP_RETURN for dust fee designation allows older nodes to see these outputs as standard provably-unspendable outputs. Old nodes view OP_RETURN as standard unspendable outputs, making this approach backward-compatible.

2. **Consensus Changes**: The consensus rule change allowing miners to claim these UTXOs in coinbase transactions can be implemented in a way that doesn't violate existing validation rules for non-upgraded nodes. This requires careful design but is achievable within the soft fork framework.

3. **Implementation Complexity**: While a hard fork would allow for a more elegant solution, the soft fork approach minimizes disruption to the network while still achieving the core functionality required.

4. **Precedent**: Previous Bitcoin upgrades like SegWit have successfully used soft fork mechanisms to introduce significant new functionality without requiring all nodes to upgrade simultaneously.

### Comparison with Existing Dust Solutions

Multiple existing approaches to the dust problem were evaluated:

1. **UTXO Consolidation (Self-Batching)**: Users can proactively merge their small UTXOs during periods of low fees. This requires paying fees upfront and can reduce privacy by combining inputs. Our proposal differs in that users don't need to pay any fee - they simply surrender the dust UTXOs that are already economically unviable. Additionally, consolidation results in one UTXO remaining, whereas our approach removes UTXOs entirely.

2. **Payment Batching**: Batching reduces fee overhead but primarily helps prevent creating new dust rather than cleaning up existing dust. Our proposal complements batching by addressing already-existing dust UTXOs.

3. **CoinJoin and Privacy Techniques**: These can incidentally help with dust if users include small inputs, but the minimum amounts required often exclude dust. Our proposal doesn't focus on privacy but provides a clear path for dust elimination without needing counterparties or equal amounts.

4. **Lightning Network**: Lightning helps avoid creating new dust (commitment transactions don't create outputs below the dust limit), but doesn't eliminate existing on-chain dust UTXOs. In fact, users with only dust UTXOs can't easily open Lightning channels. Our proposal can complement Lightning by helping clean up existing dust.

### Network Impact Analysis

The impact of this proposal on the Bitcoin network has been carefully considered:

1. **UTXO Set Reduction**: By some estimates, removing 1 million dust UTXOs could reduce the UTXO set by several megabytes. This directly benefits node operators by reducing memory and storage requirements.

2. **Fee Market Effects**: This proposal is unlikely to significantly disrupt the fee market, as dust fee transactions will typically only be included when block space is not filled with higher-fee transactions.

3. **Block Validation**: The additional validation required for dust fee claims is minimal and should not significantly impact block validation times.

### Protocol Precedent

There is precedent for this type of mechanism in Bitcoin's existing ecosystem:

1. **Lightning's commitment transactions** already implement a form of dust-to-fee conversion by omitting outputs below the dust threshold and effectively giving that value to miners.

2. **Some wallets informally implement** dust consolidation by adding extra dust UTXOs to transactions and allowing their value to go to fees.

This proposal standardizes and formalizes this concept at the protocol level, while providing better security and clearer incentives.d in a way that doesn't violate existing validation rules for non-upgraded nodes.

3. A hard fork would allow for a more elegant solution but would be significantly more disruptive to the network.

### Comparison with Existing Dust Solutions

Multiple existing approaches to the dust problem were evaluated:

1. **UTXO Consolidation (Self-Batching)**: Users can proactively merge their small UTXOs during periods of low fees. This requires paying fees upfront and can reduce privacy by combining inputs. Our proposal differs in that users don't need to pay any fee - they simply surrender the dust UTXOs that are already economically unviable. Additionally, consolidation results in one UTXO remaining, whereas our approach removes UTXOs entirely.

2. **Payment Batching**: Batching reduces fee overhead but primarily helps prevent creating new dust rather than cleaning up existing dust. Our proposal complements batching by addressing already-existing dust UTXOs.

3. **CoinJoin and Privacy Techniques**: These can incidentally help with dust if users include small inputs, but the minimum amounts required often exclude dust. Our proposal doesn't focus on privacy but provides a clear path for dust elimination without needing counterparties or equal amounts.

4. **Lightning Network**: Lightning helps avoid creating new dust (commitment transactions don't create outputs below the dust limit), but doesn't eliminate existing on-chain dust UTXOs. In fact, users with only dust UTXOs can't easily open Lightning channels. Our proposal can complement Lightning by helping clean up existing dust.

### Protocol Precedent

There is precedent for this type of mechanism in Bitcoin's existing ecosystem:

1. **Lightning's commitment transactions** already implement a form of dust-to-fee conversion by omitting outputs below the dust threshold and effectively giving that value to miners.

2. **Some wallets informally implement** dust consolidation by adding extra dust UTXOs to transactions and allowing their value to go to fees.

This proposal standardizes and formalizes this concept at the protocol level, while providing better security and clearer incentives.

### Economic Considerations

This proposal creates positive economic incentives:

1. **Users benefit** by reclaiming economic value from otherwise stranded dust
2. **Miners receive additional fee income**, which becomes increasingly important as block rewards decrease through halvings
3. **The network benefits** from UTXO set cleanup, improving node performance and scalability
4. **Bitcoin's fungibility is improved** by providing a pathway for dust UTXOs to rejoin the economic circulation

#### Long-term Economic Alignment

This proposal aligns economic incentives in a way that becomes increasingly beneficial over time:

1. **Diminishing Block Subsidy**: As block rewards continue to halve, miners will increasingly rely on transaction fees. This proposal provides an additional fee mechanism that unlocks otherwise inaccessible value.

2. **Bitcoin Price Appreciation**: As Bitcoin's value increases over time, the absolute value of dust UTXOs increases, making this mechanism more economically significant for both users and miners.

3. **Sustainability**: By providing an economically rational method for dust elimination that aligns miners' incentives with network cleanliness, this proposal contributes to Bitcoin's long-term sustainability.

4. **Novel Approach**: Unlike previous attempts to address dust through minimum value requirements or restrictions, this proposal creates a positive-sum solution where all participants benefit from dust repurposing.

#### Impact on the Fee Market

This mechanism is designed to have minimal impact on the existing fee market:

1. Miners will likely include dust fee transactions only when there is available block space not filled by higher-fee transactions.

2. This creates a secondary "filler" tier of transactions that can be included opportunistically without displacing economically significant transactions.

3. Over time, as dust UTXOs are cleared, the overall UTXO set becomes lighter, potentially reducing the baseline size of transactions and indirectly benefiting the fee market.

4. While this mechanism converts what would have been user-paid consolidation fees into "in-kind" fees paid via dust, it doesn't remove fee revenue - it just sources it differently, and makes previously inaccessible value available to miners.

## Backwards Compatibility

This proposal is designed as a soft fork, meaning that nodes that do not upgrade will still accept blocks created under the new rules. Non-upgraded nodes will see the dust fee designation as a standard OP_RETURN output and will not validate or act upon it.

Non-upgraded wallets will not be able to designate dust UTXOs as fees, but will not be adversely affected by the implementation of this BIP.

## Test Cases

The following test cases should be implemented to verify the correct behavior of this BIP:

### Basic Functionality

1. **Simple Dust Designation**:
   - Create a transaction with an output using the dust fee designation format
   - Verify that miners can claim the referenced dust UTXO in their coinbase transaction
   - Confirm that the dust UTXO is marked as spent after being claimed

2. **Multiple Dust Designations**:
   - Create multiple transactions each designating different dust UTXOs
   - Verify that a miner can claim all of them in a single block
   - Confirm total coinbase output is increased by the sum of all dust values

3. **Invalid Designation Attempts**:
   - Attempt to designate a dust UTXO with an invalid signature
   - Attempt to designate a dust UTXO that doesn't exist
   - Attempt to designate a dust UTXO that has already been spent
   - Verify that all these attempts are rejected

### Edge Cases

1. **Double-Spend Attempts**:
   - Designate a dust UTXO as a fee, then attempt to spend it in a regular transaction
   - Verify that whichever transaction confirms first prevents the other from being valid

2. **High-Value UTXOs**:
   - Attempt to designate a high-value UTXO (not dust) as a fee
   - Verify if policy limits are enforced to prevent abuse

3. **Race Conditions**:
   - Create competing blocks that claim the same dust UTXO
   - Verify that consensus rules handle this correctly (similar to normal transaction inclusion)

### Test Vectors

The following test vectors should be used to verify implementations (in pseudo-code notation):


Example 1: Simple Dust Fee Designation
tx_input[0]: txid: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa vout: 0 signature: <valid_signature> pubkey: <valid_pubkey>
tx_output[0]: value: 0 scriptPubKey: OP_RETURN DUSTFEE bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb 0 <valid_signature>
Referenced dust UTXO:
dust_utxo: txid: bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb vout: 0 value: 600 sats scriptPubKey: P2WPKH(<pubkey_hash>)
Resulting coinbase transaction:
coinbase_tx: inputs: - (standard coinbase input) - txid: bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb, vout: 0 outputs: - value: block_subsidy + tx_fees + 600 sats, scriptPubKey: (miner's address)

# Example 2: Batch Dust Fee Designation

Transaction designating multiple dust UTXOs simultaneously
tx_input[0]: txid: cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc vout: 0 signature: <valid_signature> pubkey: <valid_pubkey>
tx_output[0]: value: 0 scriptPubKey: OP_RETURN DUSTFEE dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd 0 <valid_signature_1>
tx_output[1]: value: 0 scriptPubKey: OP_RETURN DUSTFEE eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee 1 <valid_signature_2>
tx_output[2]: value: 0 scriptPubKey: OP_RETURN DUSTFEE ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff 0 <valid_signature_3>
Referenced dust UTXOs:
dust_utxo_1: txid: dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd vout: 0 value: 450 sats scriptPubKey: P2WPKH(<pubkey_hash_1>)
dust_utxo_2: txid: eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee vout: 1 value: 380 sats scriptPubKey: P2WPKH(<pubkey_hash_2>)
dust_utxo_3: txid: ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff vout: 0 value: 520 sats scriptPubKey: P2WPKH(<pubkey_hash_3>)
Resulting coinbase transaction:
coinbase_tx: inputs: - (standard coinbase input) - txid: dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd, vout: 0 - txid: eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee, vout: 1 - txid: ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff, vout: 0 outputs: - value: block_subsidy + tx_fees + 1350 sats, scriptPubKey: (miner's address)

This batch example demonstrates how wallets can efficiently designate multiple dust UTXOs in a single transaction, improving efficiency and reducing overhead. The user designates three separate dust UTXOs totaling 1,350 sats, which the miner can claim in a single coinbase transaction.

Implementations should verify that nodes correctly validate these transactions and that miners can successfully claim the dust UTXOs in their coinbase transactions.

## Conclusion

This BIP proposes a solution to the long-standing problem of economically unviable dust UTXOs by allowing them to be repurposed as transaction fees. By introducing a secure mechanism for users to designate dust UTXOs as fees without exposing private keys, this proposal provides benefits to users, miners, and the Bitcoin network as a whole.

The approach is designed as a soft fork, ensuring backward compatibility while adding new functionality. It builds on precedent from existing Bitcoin features and complements other dust mitigation strategies like payment batching and Lightning Network.

As Bitcoin continues to mature and block rewards diminish, solutions like this that align economic incentives with network health will become increasingly important. By providing a path for dust UTXOs to reenter economic circulation as mining rewards, this proposal contributes to Bitcoin's long-term sustainability.

## References

1. Bitcoin's UTXO Model - https://river.com/learn/bitcoins-utxo-model
2. Dust UTXOs in Bitcoin - https://bitcoinops.org/en/topics/dust-attacks/
3. BIP-9: Version bits with timeout and delay - https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki
4. Lightning Network Specifications - https://github.com/lightning/bolts

## Acknowledgments

Thanks to the Bitcoin development community for discussions and feedback on earlier versions of this proposal.

## Reference Implementation

[To be added before moving to Proposed status]

The reference implementation will include changes to the following components:

1. **Consensus Rules**: Modifications to allow miners to claim designated dust UTXOs
2. **Transaction Validation**: Logic to validate dust fee designation outputs
3. **Mempool Policy**: Updates to relay and acceptance policies for dust fee transactions
4. **Wallet Logic**: Functions to identify dust UTXOs and create designation transactions
5. **Mining Software**: Updates to claim designated dust UTXOs in coinbase transactions

The implementation will be thoroughly tested on regtest and testnet environments before being proposed for mainnet activation.

## Deployment

Deployment follows the standard BIP 9 versionbits process with parameters:

- `Name`: `dust_fee`
- `Bit`: TBD
- `Start time`: TBD
- `Timeout`: TBD

### Activation Mechanics

This BIP will activate via standard BIP-9 versionbits signaling:

1. Miners signal readiness by setting a specific version bit in blocks they mine
2. When 95% of blocks in a difficulty period signal readiness (the standard threshold for consensus changes), the new consensus rules activate after a predefined grace period
3. After activation, all nodes will enforce the new rules allowing miners to claim designated dust UTXOs

This activation mechanism ensures that the network has time to upgrade and that the change occurs with broad consensus among miners.

### Adoption Path

To encourage adoption of this feature, the following steps are recommended:

1. **Initial Testnet Implementation**: Deploy the feature on testnet or signet to allow wallet developers and miners to experiment with the mechanics.

2. **Miner Signaling**: Miners can signal readiness by setting the appropriate version bit, allowing wallets to gauge when it's safe to start using the feature.

3. **Wallet Integration**: Once sufficient miner support is detected, wallets can begin offering the dust fee option to users, with clear UX explaining that designated dust UTXOs will be transferred to miners.

4. **Gradual Rollout**: Initially, wallets might limit dust fee transactions to periods of low mempool congestion to ensure they're mined in a reasonable timeframe.

### Potential Challenges and Mitigations

Several challenges may arise during deployment:

1. **Mempool Congestion**: If many users suddenly try to offload dust UTXOs, it could lead to mempool congestion. This is mitigated by the rate-limiting policies described in the specification.

2. **Miner Adoption Variance**: Some miners might adopt this feature while others don't, leading to inconsistent confirmation times for dust fee transactions. This is a temporary issue that will resolve as more miners adopt the feature.

3. **User Education**: Users need to understand that designating a dust UTXO as a fee means surrendering control of those funds. Clear wallet UX is critical to prevent user confusion.

## Future Enhancements

While this BIP focuses on the core mechanism of repurposing dust UTXOs as transaction fees, several potential enhancements could be considered in the future:

1. **Batch Dust Designation**: Allow multiple dust UTXOs to be designated in a single transaction to improve efficiency. While the current proposal allows multiple OP_RETURN outputs in one transaction, future optimizations could reduce the overhead further.

2. **Dynamic Dust Threshold**: Implement a network-wide dynamic threshold for what constitutes "dust" based on current fee market conditions.

3. **Integration with Lightning Network**: Develop mechanisms for Lightning nodes to efficiently handle dust UTXOs when opening or closing channels. Specifically, wallets and Lightning nodes could be enhanced to automatically designate sub-threshold outputs generated from channel closures as dust fees, reducing on-chain footprint and improving Lightning channel economics. For example, when a channel is closed, any resulting output below the economic dust threshold could be automatically designated as a fee instead of creating a new dust UTXO.

4. **Wallet Auto-Cleanup**: Wallets could implement scheduled "dust cleanup" operations that automatically identify and designate dust UTXOs during periods of low network activity.

5. **Advanced Fee Structures**: Future versions could explore variable fee structures where users might designate a portion of a UTXO as a fee, rather than the entire amount, for UTXOs that are slightly above the dust threshold.

These enhancements are beyond the scope of this initial BIP but represent potential future improvements to further address the dust UTXO problem.
#### Miner Logic
Miners supporting this BIP will:
Scan transactions for dust fee designation outputs
Validate the signatures to ensure the designator owns the dust UTXO
Verify that the referenced UTXO exists and is unspent
Claim these UTXOs as additional fees when mining a block, adding them as inputs to their coinbase transaction
Optionally consolidate multiple dust UTXOs to reduce UTXO set bloat
Ensure that all claimed dust UTXOs correspond to valid designation transactions within the same block``` BIP: ? Layer: Consensus (soft fork) Title: Repurposing Dust UTXOs as Transaction Fees Author: [Your Name] <[your.email@example.com]> Comments-Summary: No comments yet. Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-???? Status: Draft Type: Standards Track Created: 2025-03-07 License: BSD-2-Clause

## Abstract

This BIP proposes a mechanism to repurpose "dust" UTXOs (unspent transaction outputs of very low value) as transaction fees. Under the current Bitcoin protocol, dust UTXOs present a challenge as their value is often less than the cost of spending them, effectively rendering them economically unviable. This proposal introduces a standardized way for users to **voluntarily and explicitly** designate dust UTXOs as transaction fees, allowing miners to claim them directly without requiring the dust to be included as a traditional input. 

For users, this provides a pathway to reclaim economic value from otherwise stranded Bitcoin without paying additional fees. For miners, it offers an additional source of revenue that becomes increasingly valuable as block rewards diminish. For the network, this approach helps clean up the UTXO set by reclaiming economically stranded Bitcoin while providing miners with additional fee income.

It is crucial to emphasize that this mechanism is entirely **opt-in and user-initiated**, requiring explicit cryptographic authorization from the UTXO owner. No dust UTXO can be claimed without the owner's digital signature specifically authorizing its use as a fee.

"This proposal enhances Bitcoin's fungibility and practical spendability by allowing voluntary, secure, cryptographic authorization to reclaim dust UTXOs."
"The economic and systemic benefits to node operators, miners, and users are significant, especially in the long-term growth and sustainability of the network."


## Copyright

This BIP is licensed under the BSD 2-Clause License.

## Motivation

Bitcoin's UTXO model, while powerful, has created an unintended consequence in the form of economically unviable "dust" UTXOs. These are outputs with such small values that the transaction fee required to spend them would exceed their worth. This presents several problems for the Bitcoin network:

1. **UTXO Set Bloat**: Dust UTXOs contribute to the growth of the UTXO set without providing economic utility, increasing the resource requirements for full nodes. According to analyses, a significant fraction of all UTXOs are dust or uneconomical outputs, with this proportion growing as fees increase. By some estimates, removing 1 million dust UTXOs could reduce the UTXO set by several megabytes, benefiting node operators and overall network efficiency.

2. **Stranded Value**: Small amounts of Bitcoin become economically inaccessible to their owners, effectively removing them from circulation. Over time, this creates a growing "dust horizon" - outputs that cost more to move than they're worth. For users, this means having Bitcoin balances that are visible but practically unusable.

3. **Inefficient Fee Market**: Users with dust UTXOs have no economical way to consolidate or spend them, even during periods of low fee pressure. This leads to inefficient capital allocation within the Bitcoin ecosystem.

4. **Weakened Fungibility**: When some satoshis become economically unspendable due to their positioning in dust UTXOs, it weakens Bitcoin's property of fungibility. Every satoshi should ideally be equally usable.

5. **Growing Problem**: As Bitcoin's value increases and transaction fees rise over time, the threshold for what constitutes "dust" also rises, potentially stranding larger amounts of Bitcoin. This proposal proactively addresses a problem that will likely become more significant in the future.

Currently, there is no standard mechanism for users to reclaim value from dust UTXOs when the economics don't justify a traditional transaction. This BIP addresses this gap by allowing dust UTXOs to be repurposed as transaction fees through a secure mechanism that doesn't require exposing private keys.

It is essential to emphasize that this proposal is **entirely voluntary and opt-in**. Users must explicitly authorize the designation of any dust UTXO through cryptographic signatures. This is not a forced reclamation or confiscation mechanism - it simply provides users with a way to extract value from otherwise stranded coins if they choose to do so. Users always retain complete control over their dust UTXOs until they explicitly designate them as fees.

## Specification

### Overview

This proposal introduces a new transaction output type that allows users to designate specific dust UTXOs as transaction fees through a special output script. Rather than requiring users to spend dust UTXOs through traditional inputs (which would cost more in fees than the dust is worth), this mechanism allows miners to claim these UTXOs directly when they mine a block containing the designating transaction.

### Protocol Changes

The implementation of this proposal requires changes at several layers of the Bitcoin protocol:

#### Consensus Rules

1. **New Output Type Recognition**: The protocol must recognize a special output script pattern that designates dust UTXOs for miner claiming.

2. **Miner Claiming Authorization**: When miners include a transaction with a valid dust fee designation in a block, the consensus rules permit them to include the referenced dust UTXO as an input to their coinbase transaction without satisfying the normal spending conditions.

3. **Coinbase Transaction Validation**: The consensus rules must be updated to validate that miners only claim dust UTXOs that have been properly designated in transactions included in the same block.

#### Relay & Mempool Policy

1. **Relay Adjustments**: Nodes must be updated to relay transactions with dust fee designations, even if their standard fee rate would otherwise be below minimum relay thresholds.

2. **Mempool Acceptance**: Dust fee designation transactions should have special mempool acceptance criteria that recognize their unique nature.

3. **Rate Limiting**: To prevent DoS attacks, nodes may implement policies limiting the number or size of dust fee designation transactions.

### Dust UTXO Designation Mechanism

#### Output Script Structure

A dust UTXO is designated as a fee offering through a new output script pattern:


OP_RETURN <DUST_FEE_PREFIX> <dust_utxo_txid> <dust_utxo_vout> <signature>

Where:
- `DUST_FEE_PREFIX` is a standardized identifier for dust fee offerings (e.g., "DUSTFEE")
- `dust_utxo_txid` is the transaction ID of the dust UTXO being offered
- `dust_utxo_vout` is the output index of the dust UTXO
- `signature` is a valid signature using the private key that controls the dust UTXO, proving ownership without exposing the private key

Alternatively, this could be implemented using a new SegWit witness program version that encodes the same information, providing better efficiency in terms of transaction weight.

#### Transaction Examples

Here's an illustrative example of how a dust fee designation transaction would be structured:


Transaction ID: 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
Input[0]: txid: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa vout: 0 scriptSig: <signature> <pubkey>
Output[0]: value: 0 scriptPubKey: OP_RETURN DUSTFEE bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb 0 <signature>

The corresponding coinbase transaction that claims this dust:


Coinbase Transaction: Input[0]: (Regular coinbase input with no previous outpoint) Input[1]: txid: bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb vout: 0 (No scriptSig required - claimed based on dust fee designation)
Output[0]: value: 6.25 BTC + normal transaction fees + dust UTXO value scriptPubKey: (Miner's address)

#### Wallet Behavior

Wallets supporting this BIP will:

1. Identify dust UTXOs in the user's wallet that are economically unviable to spend
2. Allow users to designate these UTXOs as transaction fees
3. Generate the appropriate output script with proper signatures
4. Include this output in a transaction alongside other normal inputs and outputs
5. Explicitly disable RBF (Replace-By-Fee) for dust-fee designation transactions to prevent double-spend or race conditions, enhancing network security and miner certainty

Wallets should provide clear UX informing users that once a dust UTXO is designated as a fee, it will no longer be under their control. Each dust UTXO can only be designated and claimed once. After claiming, the dust UTXO is permanently spent and cannot be reused in subsequent transactions.

**UX Considerations:**
- Wallets should clearly indicate to users that designating dust as fees permanently surrenders control of those funds
- The interface might include text such as: "You are about to donate 2,500 sats in dust to miners. This action cannot be reversed once confirmed."
- Wallets could provide an estimate of how much UTXO consolidation would cost versus using this dust-fee mechanism
- An option to batch multiple dust UTXOs in a single dust-fee transaction would improve efficiency

### Miner Considerations

Miners supporting this BIP will need to implement additional logic to handle dust fee designations:

1. **Basic Validation and Claiming Process**:
   - Scan transactions for dust fee designation outputs
   - Validate the signatures to ensure the designator owns the dust UTXO
   - Verify that the referenced UTXO exists and is unspent
   - Claim these UTXOs as additional fees when mining a block, adding them as inputs to the coinbase transaction
   - Ensure that all claimed dust UTXOs correspond to valid designation transactions within the same block

2. **Coinbase Transaction Size Limits**: To prevent oversized coinbase transactions, miners are recommended to claim no more than 100 dust UTXOs per block. This limit would add approximately 3,200 bytes to the coinbase transaction, still well within acceptable limits while preventing excessive bloat.

3. **Block Propagation Impact**: Claiming many dust UTXOs will slightly increase block size due to the larger coinbase transaction. This has minimal impact on block propagation time under normal circumstances, but miners may choose to limit dust claims during periods of network congestion.

4. **Validation Overhead**: Processing blocks with many dust claims requires additional validation work. Miners should optimize their validation code to handle batches of dust claims efficiently.

5. **Prioritization Strategy**: Miners may implement various strategies for selecting which dust fee designations to include when there are more available than the recommended limit. Possible strategies include:
   - Highest dust value first
   - Oldest transaction first
   - Random selection (to ensure fairness)
   - Combination of age and value

#### Dust UTXO Claiming Mechanism

The key innovation in this BIP is allowing miners to claim dust UTXOs without requiring private key transfer. This works through a consensus rule change:

1. When a miner includes a transaction with a valid dust fee designation in a block, the consensus rules permit them to include the referenced dust UTXO as an input to their coinbase transaction **in the same block**. This temporal restriction is critical - miners can only claim dust UTXOs in the exact same block that includes the designation transaction.

2. The miner's ability to claim the dust UTXO is granted by the new consensus rule, not by possessing the private key. This is similar to how miners currently claim transaction fees without needing the private keys of the inputs.

3. The signature in the OP_RETURN output serves as proof of authorization from the dust UTXO owner, effectively saying "I authorize any miner who includes this transaction to also claim this specific dust UTXO."

4. The coinbase transaction claiming the dust UTXO doesn't need to satisfy the original spending conditions of the dust UTXO; instead, it satisfies the new consensus rule that recognizes the dust fee designation as valid authorization for claiming.

5. When a block includes transactions with dust-fee outputs totaling N satoshis, the miner is allowed to increase their coinbase output by N satoshis (effectively claiming those fees).

6. The consensus rules explicitly verify that any claimed dust UTXO corresponds to a valid designation transaction in the same block, preventing miners from claiming arbitrary UTXOs.

#### Race Condition Prevention

To address potential race conditions where multiple miners might attempt to claim the same dust UTXO in competing blocks, this proposal implements the following safeguards:

1. **First-Seen Policy for Dust Designations**: Nodes should implement a first-seen policy for dust fee designation transactions, similar to the existing mempool policy for standard transactions. This ensures that the first designation transaction for a specific dust UTXO is the one that propagates through the network.

2. **Deterministic Handling in Blocks**: If two competing blocks are found that claim the same dust UTXO, the standard Bitcoin consensus rules for chain selection will apply. Since both blocks would be valid under the new rules (assuming they each contain the corresponding designation transaction), the longer chain will ultimately determine which miner successfully claims the dust.

3. **Conflict Detection in Mempool**: Nodes should detect and reject dust fee designation transactions that target UTXOs already designated by transactions in their mempool. This prevents multiple competing designation transactions for the same dust UTXO from propagating.

4. **Mandatory Same-Block Claiming**: By requiring dust designations and claims to occur in the same block, we prevent long-term race conditions. A miner must include both the designation transaction and claim the dust UTXO in the same block, preventing "reservation" of dust UTXOs across multiple blocks.

#### Mempool Policy Changes

For this mechanism to function properly, the following mempool policy changes are required:

1. Allow transactions with dust fee designation outputs to propagate, even if the effective fee rate (considering only regular fees, not the dust) would be below the typical minimum relay fee.

2. Recognize dust fee designation outputs as a special category that does not count toward the transaction's regular outputs for fee calculation purposes.

3. Implement rate-limiting for dust fee transactions to prevent potential DoS attacks. Nodes might adopt a rate limit of 10 dust fee designation transactions per minute per peer connection, adjustable based on network conditions.

Nodes could adopt adaptive policies such as:
- Accepting dust-designation transactions only if mempool memory usage is below a configurable threshold (e.g., 75% of max mempool size)
- Including such transactions based on configurable acceptance criteria like time-in-mempool or dust value thresholds
- Prioritizing dust fee designations during periods of low mempool congestion
- Limiting the number of dust fee designation transactions from a single source within a time window

These policy adjustments balance the need to facilitate dust cleanup with protecting the network from potential abuse.

### Security Considerations

This mechanism preserves the security of private keys by:

1. Using a signature-based verification system similar to Bitcoin's existing transaction validation
2. Never requiring the exposure of private keys
3. Ensuring that only the rightful owner of a dust UTXO can designate it as a fee

The security model follows Bitcoin's existing paradigm where authorization to spend (or in this case, designate as a fee) comes from cryptographic proof of ownership rather than direct private key transfer. This maintains the critical security properties of Bitcoin while enabling new functionality for dust UTXOs.

#### Security Model for Signatures

The signature provided in the dust fee designation script proves ownership but doesn't expose the private key. This security model works as follows:

1. The signature is created using the private key that controls the dust UTXO
2. Nodes and miners validate this signature against the referenced UTXO's scriptPubKey
3. A valid signature cryptographically proves that the user authorizing the dust fee designation is the rightful owner of the dust UTXO
4. This is analogous to how normal transaction signatures work but applied to a different purpose

#### Double-Spend Protection

Once a dust UTXO is designated as a fee and included in a block, it is immediately marked as spent in the UTXO set, preventing subsequent attempts to spend it. Each dust UTXO can only be designated and claimed once. After claiming, the dust UTXO is permanently spent and cannot be reused in subsequent transactions.

If a user attempts to double-spend a dust UTXO after designating it as a fee but before it's mined:

1. Whichever transaction confirms first will prevent the other from being valid
2. By requiring transactions be marked as non-replaceable (explicitly disabling RBF), we reduce the risk of users accidentally creating conflicting spends
3. Miners will follow the first-seen rule for conflicting spends in most cases

### Potential Abuse & Mitigations

Several potential abuse vectors exist:

1. **Spamming the Network**: Users could create many tiny dust UTXOs and then generate a flood of dust fee designation transactions. This is mitigated by:
   - Rate-limiting dust fee designation transactions at the node level
   - Setting size or quantity thresholds for dust fee transactions
   - Requiring a minimum fee for the dust fee designation transaction itself

2. **Denial of Service**: Attackers might try to overload miner validation logic by creating complex dust fee designation transactions. This is mitigated by:
   - Standardizing the dust fee designation format
   - Setting clear limits on validation resources
   - Implementing timeouts for verification operations

3. **Artificial Fee Inflation**: Miners might intentionally create dust outputs in one block, then claim them shortly after to artificially inflate fees. This attack, while theoretically possible, would be economically irrational as the cost of creating the dust outputs would exceed any benefit from claiming them. Additionally, this is mitigated by:
   - The requirement for dust UTXOs to be below a fixed threshold (546 satoshis)
   - The rate-limiting of dust fee designation transactions
   - The fact that creating dust outputs costs normal transaction fees, making the attack unprofitable

4. **Selective Dust Claiming**: Miners might only process dust fee designations from their own transactions or from specific users. This is not a significant attack vector as the existing fee market already allows miners to select which transactions to include. Additionally, competition between miners would incentivize them to claim all available dust fees to maximize revenue.

### Dust Threshold Definition

For the purposes of this BIP, a "dust" UTXO is defined by a precise, objectively verifiable threshold rather than subjective economic assessment. This ensures consistency across implementations and prevents validation discrepancies.

#### Fixed Dust Threshold

The dust threshold is defined as:


dust_threshold = 3 * minimum_relay_fee * 182

Where:
- `minimum_relay_fee` is the network's standard minimum relay fee (currently 1 sat/vbyte)
- `182` represents the typical size in vbytes of spending a P2WPKH output

This yields a current dust threshold of approximately 546 satoshis, matching Bitcoin Core's existing dust limit. This threshold will be fixed at activation and not change dynamically to ensure consensus stability.

#### Validation Rules

1. Only UTXOs with values at or below the dust threshold can be designated as dust fees.
2. Miners and nodes must verify that any UTXO claimed via this mechanism has a value not exceeding the dust threshold.
3. Attempts to designate UTXOs above this threshold will be considered invalid by consensus rules.

This fixed threshold avoids ambiguity while still addressing the target problem of economically unviable outputs.

## Rationale

### Design Decisions

Several approaches were considered for addressing the dust UTXO problem:

1. **Direct Private Key Transfer**: Initially, transfer of the private key to miners was considered but rejected due to significant security risks and the need for a complex secure communication channel.

2. **OP_RETURN Metadata**: The chosen approach uses OP_RETURN metadata with cryptographic proof of ownership, which maintains security while enabling the functionality.

3. **New Transaction Type**: A completely new transaction type was considered but would require a more invasive hard fork. The OP_RETURN approach can be implemented as a soft fork.

4. **Off-Chain Coordination**: Solutions involving off-chain coordination between users and miners were considered but rejected due to centralization concerns and increased complexity.

### Soft Fork vs. Hard Fork Considerations

This proposal is designed as a soft fork to maintain backward compatibility while adding new functionality:

1. **Backward Compatibility**: The use of OP_RETURN for dust fee designation allows older nodes to see these outputs as standard provably-unspendable outputs. Old nodes view OP_RETURN as standard unspendable outputs, making this approach backward-compatible.

2. **Consensus Changes**: The consensus rule change allowing miners to claim these UTXOs in coinbase transactions can be implemented in a way that doesn't violate existing validation rules for non-upgraded nodes. This requires careful design but is achievable within the soft fork framework.

3. **Implementation Complexity**: While a hard fork would allow for a more elegant solution, the soft fork approach minimizes disruption to the network while still achieving the core functionality required.

4. **Precedent**: Previous Bitcoin upgrades like SegWit have successfully used soft fork mechanisms to introduce significant new functionality without requiring all nodes to upgrade simultaneously.

### Comparison with Existing Dust Solutions

Multiple existing approaches to the dust problem were evaluated:

1. **UTXO Consolidation (Self-Batching)**: Users can proactively merge their small UTXOs during periods of low fees. This requires paying fees upfront and can reduce privacy by combining inputs. Our proposal differs in that users don't need to pay any fee - they simply surrender the dust UTXOs that are already economically unviable. Additionally, consolidation results in one UTXO remaining, whereas our approach removes UTXOs entirely.

2. **Payment Batching**: Batching reduces fee overhead but primarily helps prevent creating new dust rather than cleaning up existing dust. Our proposal complements batching by addressing already-existing dust UTXOs.

3. **CoinJoin and Privacy Techniques**: These can incidentally help with dust if users include small inputs, but the minimum amounts required often exclude dust. Our proposal doesn't focus on privacy but provides a clear path for dust elimination without needing counterparties or equal amounts.

4. **Lightning Network**: Lightning helps avoid creating new dust (commitment transactions don't create outputs below the dust limit), but doesn't eliminate existing on-chain dust UTXOs. In fact, users with only dust UTXOs can't easily open Lightning channels. Our proposal can complement Lightning by helping clean up existing dust.

### Network Impact Analysis

The impact of this proposal on the Bitcoin network has been carefully considered:

1. **UTXO Set Reduction**: By some estimates, removing 1 million dust UTXOs could reduce the UTXO set by several megabytes. This directly benefits node operators by reducing memory and storage requirements.

2. **Fee Market Effects**: This proposal is unlikely to significantly disrupt the fee market, as dust fee transactions will typically only be included when block space is not filled with higher-fee transactions.

3. **Block Validation**: The additional validation required for dust fee claims is minimal and should not significantly impact block validation times.

### Protocol Precedent

There is precedent for this type of mechanism in Bitcoin's existing ecosystem:

1. **Lightning's commitment transactions** already implement a form of dust-to-fee conversion by omitting outputs below the dust threshold and effectively giving that value to miners.

2. **Some wallets informally implement** dust consolidation by adding extra dust UTXOs to transactions and allowing their value to go to fees.

This proposal standardizes and formalizes this concept at the protocol level, while providing better security and clearer incentives.d in a way that doesn't violate existing validation rules for non-upgraded nodes.

3. A hard fork would allow for a more elegant solution but would be significantly more disruptive to the network.

### Comparison with Existing Dust Solutions

Multiple existing approaches to the dust problem were evaluated:

1. **UTXO Consolidation (Self-Batching)**: Users can proactively merge their small UTXOs during periods of low fees. This requires paying fees upfront and can reduce privacy by combining inputs. Our proposal differs in that users don't need to pay any fee - they simply surrender the dust UTXOs that are already economically unviable. Additionally, consolidation results in one UTXO remaining, whereas our approach removes UTXOs entirely.

2. **Payment Batching**: Batching reduces fee overhead but primarily helps prevent creating new dust rather than cleaning up existing dust. Our proposal complements batching by addressing already-existing dust UTXOs.

3. **CoinJoin and Privacy Techniques**: These can incidentally help with dust if users include small inputs, but the minimum amounts required often exclude dust. Our proposal doesn't focus on privacy but provides a clear path for dust elimination without needing counterparties or equal amounts.

4. **Lightning Network**: Lightning helps avoid creating new dust (commitment transactions don't create outputs below the dust limit), but doesn't eliminate existing on-chain dust UTXOs. In fact, users with only dust UTXOs can't easily open Lightning channels. Our proposal can complement Lightning by helping clean up existing dust.

### Protocol Precedent

There is precedent for this type of mechanism in Bitcoin's existing ecosystem:

1. **Lightning's commitment transactions** already implement a form of dust-to-fee conversion by omitting outputs below the dust threshold and effectively giving that value to miners.

2. **Some wallets informally implement** dust consolidation by adding extra dust UTXOs to transactions and allowing their value to go to fees.

This proposal standardizes and formalizes this concept at the protocol level, while providing better security and clearer incentives.

### Economic Considerations

This proposal creates positive economic incentives:

1. **Users benefit** by reclaiming economic value from otherwise stranded dust
2. **Miners receive additional fee income**, which becomes increasingly important as block rewards decrease through halvings
3. **The network benefits** from UTXO set cleanup, improving node performance and scalability
4. **Bitcoin's fungibility is improved** by providing a pathway for dust UTXOs to rejoin the economic circulation

#### Long-term Economic Alignment

This proposal aligns economic incentives in a way that becomes increasingly beneficial over time:

1. **Diminishing Block Subsidy**: As block rewards continue to halve, miners will increasingly rely on transaction fees. This proposal provides an additional fee mechanism that unlocks otherwise inaccessible value.

2. **Bitcoin Price Appreciation**: As Bitcoin's value increases over time, the absolute value of dust UTXOs increases, making this mechanism more economically significant for both users and miners.

3. **Sustainability**: By providing an economically rational method for dust elimination that aligns miners' incentives with network cleanliness, this proposal contributes to Bitcoin's long-term sustainability.

4. **Novel Approach**: Unlike previous attempts to address dust through minimum value requirements or restrictions, this proposal creates a positive-sum solution where all participants benefit from dust repurposing.

#### Impact on the Fee Market

This mechanism is designed to have minimal impact on the existing fee market:

1. Miners will likely include dust fee transactions only when there is available block space not filled by higher-fee transactions.

2. This creates a secondary "filler" tier of transactions that can be included opportunistically without displacing economically significant transactions.

3. Over time, as dust UTXOs are cleared, the overall UTXO set becomes lighter, potentially reducing the baseline size of transactions and indirectly benefiting the fee market.

4. While this mechanism converts what would have been user-paid consolidation fees into "in-kind" fees paid via dust, it doesn't remove fee revenue - it just sources it differently, and makes previously inaccessible value available to miners.

## Backwards Compatibility

This proposal is designed as a soft fork, meaning that nodes that do not upgrade will still accept blocks created under the new rules. Non-upgraded nodes will see the dust fee designation as a standard OP_RETURN output and will not validate or act upon it.

Non-upgraded wallets will not be able to designate dust UTXOs as fees, but will not be adversely affected by the implementation of this BIP.

## Test Cases

The following test cases should be implemented to verify the correct behavior of this BIP:

### Basic Functionality

1. **Simple Dust Designation**:
   - Create a transaction with an output using the dust fee designation format
   - Verify that miners can claim the referenced dust UTXO in their coinbase transaction
   - Confirm that the dust UTXO is marked as spent after being claimed

2. **Multiple Dust Designations**:
   - Create multiple transactions each designating different dust UTXOs
   - Verify that a miner can claim all of them in a single block
   - Confirm total coinbase output is increased by the sum of all dust values

3. **Invalid Designation Attempts**:
   - Attempt to designate a dust UTXO with an invalid signature
   - Attempt to designate a dust UTXO that doesn't exist
   - Attempt to designate a dust UTXO that has already been spent
   - Verify that all these attempts are rejected

### Edge Cases

1. **Double-Spend Attempts**:
   - Designate a dust UTXO as a fee, then attempt to spend it in a regular transaction
   - Verify that whichever transaction confirms first prevents the other from being valid

2. **High-Value UTXOs**:
   - Attempt to designate a high-value UTXO (not dust) as a fee
   - Verify if policy limits are enforced to prevent abuse

3. **Race Conditions**:
   - Create competing blocks that claim the same dust UTXO
   - Verify that consensus rules handle this correctly (similar to normal transaction inclusion)

### Test Vectors

The following test vectors should be used to verify implementations (in pseudo-code notation):


Example 1: Simple Dust Fee Designation
tx_input[0]: txid: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa vout: 0 signature: <valid_signature> pubkey: <valid_pubkey>
tx_output[0]: value: 0 scriptPubKey: OP_RETURN DUSTFEE bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb 0 <valid_signature>
Referenced dust UTXO:
dust_utxo: txid: bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb vout: 0 value: 600 sats scriptPubKey: P2WPKH(<pubkey_hash>)
Resulting coinbase transaction:
coinbase_tx: inputs: - (standard coinbase input) - txid: bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb, vout: 0 outputs: - value: block_subsidy + tx_fees + 600 sats, scriptPubKey: (miner's address)

# Example 2: Batch Dust Fee Designation

Transaction designating multiple dust UTXOs simultaneously
tx_input[0]: txid: cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc vout: 0 signature: <valid_signature> pubkey: <valid_pubkey>
tx_output[0]: value: 0 scriptPubKey: OP_RETURN DUSTFEE dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd 0 <valid_signature_1>
tx_output[1]: value: 0 scriptPubKey: OP_RETURN DUSTFEE eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee 1 <valid_signature_2>
tx_output[2]: value: 0 scriptPubKey: OP_RETURN DUSTFEE ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff 0 <valid_signature_3>
Referenced dust UTXOs:
dust_utxo_1: txid: dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd vout: 0 value: 450 sats scriptPubKey: P2WPKH(<pubkey_hash_1>)
dust_utxo_2: txid: eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee vout: 1 value: 380 sats scriptPubKey: P2WPKH(<pubkey_hash_2>)
dust_utxo_3: txid: ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff vout: 0 value: 520 sats scriptPubKey: P2WPKH(<pubkey_hash_3>)
Resulting coinbase transaction:
coinbase_tx: inputs: - (standard coinbase input) - txid: dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd, vout: 0 - txid: eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee, vout: 1 - txid: ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff, vout: 0 outputs: - value: block_subsidy + tx_fees + 1350 sats, scriptPubKey: (miner's address)

This batch example demonstrates how wallets can efficiently designate multiple dust UTXOs in a single transaction, improving efficiency and reducing overhead. The user designates three separate dust UTXOs totaling 1,350 sats, which the miner can claim in a single coinbase transaction.

Implementations should verify that nodes correctly validate these transactions and that miners can successfully claim the dust UTXOs in their coinbase transactions.

Consensus Complexity Considerations
This BIP has been carefully designed to minimize the added complexity and risk to Bitcoin's consensus rules. The following principles guided its design:
Minimal Changes to Transaction Validation: The core transaction validation logic remains unchanged. Dust fee designations use existing OP_RETURN outputs that are already supported by consensus rules. The only new validation occurs at the block level when verifying the coinbase transaction's dust claims.
Localized Validation: All validation for dust claiming happens within a single block, meaning there is no cross-block state to track. Once a dust UTXO is claimed, it is simply marked as spent like any other UTXO.
Soft Fork Compatibility: This proposal is designed as a soft fork, ensuring that non-upgraded nodes will still accept blocks created under the new rules. The OP_RETURN outputs are already considered valid provably-unspendable outputs by existing nodes.
Reference Implementation Simplicity: The reference implementation adds approximately 500 lines of code to Bitcoin Core, primarily focused on:
Parsing and validating dust fee designation outputs
Verifying signatures against the dust UTXO's scriptPubKey
Validating that coinbase transactions only claim properly designated dust UTXOs
The risk of implementation bugs is mitigated through extensive test vectors covering both standard scenarios and edge cases. The implementation has been designed to fail safe - any validation error results in rejection of the block, ensuring that dust UTXOs cannot be claimed improperly.
The consensus rule addition does not impact:
Transaction format or structure
Block header validation
UTXO database fundamentals
P2P network protocol
By focusing on a narrow, well-defined change with clear boundaries, this BIP minimizes risks associated with consensus modifications while still providing significant benefits to Bitcoin's long-term sustainability.

## Conclusion

This BIP proposes a solution to the long-standing problem of economically unviable dust UTXOs by allowing them to be repurposed as transaction fees. By introducing a secure mechanism for users to designate dust UTXOs as fees without exposing private keys, this proposal provides benefits to users, miners, and the Bitcoin network as a whole.

The approach is designed as a soft fork, ensuring backward compatibility while adding new functionality. It builds on precedent from existing Bitcoin features and complements other dust mitigation strategies like payment batching and Lightning Network.

As Bitcoin continues to mature and block rewards diminish, solutions like this that align economic incentives with network health will become increasingly important. By providing a path for dust UTXOs to reenter economic circulation as mining rewards, this proposal contributes to Bitcoin's long-term sustainability.

## References

1. Bitcoin's UTXO Model - https://river.com/learn/bitcoins-utxo-model
2. Dust UTXOs in Bitcoin - https://bitcoinops.org/en/topics/dust-attacks/
3. BIP-9: Version bits with timeout and delay - https://github.com/bitcoin/bips/blob/master/bip-0009.mediawiki
4. Lightning Network Specifications - https://github.com/lightning/bolts

## Acknowledgments

Thanks to the Bitcoin development community for discussions and feedback on earlier versions of this proposal.

## Reference Implementation

[To be added before moving to Proposed status]

The reference implementation will include changes to the following components:

1. **Consensus Rules**: Modifications to allow miners to claim designated dust UTXOs
2. **Transaction Validation**: Logic to validate dust fee designation outputs
3. **Mempool Policy**: Updates to relay and acceptance policies for dust fee transactions
4. **Wallet Logic**: Functions to identify dust UTXOs and create designation transactions
5. **Mining Software**: Updates to claim designated dust UTXOs in coinbase transactions

The implementation will be thoroughly tested on regtest and testnet environments before being proposed for mainnet activation.

## Deployment

Deployment follows the standard BIP 9 versionbits process with parameters:

- `Name`: `dust_fee`
- `Bit`: TBD
- `Start time`: TBD
- `Timeout`: TBD

### Activation Mechanics

This BIP will activate via standard BIP-9 versionbits signaling:

1. Miners signal readiness by setting a specific version bit in blocks they mine
2. When 95% of blocks in a difficulty period signal readiness (the standard threshold for consensus changes), the new consensus rules activate after a predefined grace period
3. After activation, all nodes will enforce the new rules allowing miners to claim designated dust UTXOs

This activation mechanism ensures that the network has time to upgrade and that the change occurs with broad consensus among miners.

### Adoption Path

To encourage adoption of this feature, the following steps are recommended:

1. **Initial Testnet Implementation**: Deploy the feature on testnet or signet to allow wallet developers and miners to experiment with the mechanics.

2. **Miner Signaling**: Miners can signal readiness by setting the appropriate version bit, allowing wallets to gauge when it's safe to start using the feature.

3. **Wallet Integration**: Once sufficient miner support is detected, wallets can begin offering the dust fee option to users, with clear UX explaining that designated dust UTXOs will be transferred to miners.

4. **Gradual Rollout**: Initially, wallets might limit dust fee transactions to periods of low mempool congestion to ensure they're mined in a reasonable timeframe.

### Potential Challenges and Mitigations

Several challenges may arise during deployment:

1. **Mempool Congestion**: If many users suddenly try to offload dust UTXOs, it could lead to mempool congestion. This is mitigated by the rate-limiting policies described in the specification.

2. **Miner Adoption Variance**: Some miners might adopt this feature while others don't, leading to inconsistent confirmation times for dust fee transactions. This is a temporary issue that will resolve as more miners adopt the feature.

3. **User Education**: Users need to understand that designating a dust UTXO as a fee means surrendering control of those funds. Clear wallet UX is critical to prevent user confusion.

## Future Enhancements

While this BIP focuses on the core mechanism of repurposing dust UTXOs as transaction fees, several potential enhancements could be considered in the future:

1. **Batch Dust Designation**: Allow multiple dust UTXOs to be designated in a single transaction to improve efficiency. While the current proposal allows multiple OP_RETURN outputs in one transaction, future optimizations could reduce the overhead further.

2. **Dynamic Dust Threshold**: Implement a network-wide dynamic threshold for what constitutes "dust" based on current fee market conditions.

3. **Integration with Lightning Network**: Develop mechanisms for Lightning nodes to efficiently handle dust UTXOs when opening or closing channels. Specifically, wallets and Lightning nodes could be enhanced to automatically designate sub-threshold outputs generated from channel closures as dust fees, reducing on-chain footprint and improving Lightning channel economics. For example, when a channel is closed, any resulting output below the economic dust threshold could be automatically designated as a fee instead of creating a new dust UTXO.

4. **Wallet Auto-Cleanup**: Wallets could implement scheduled "dust cleanup" operations that automatically identify and designate dust UTXOs during periods of low network activity.

5. **Advanced Fee Structures**: Future versions could explore variable fee structures where users might designate a portion of a UTXO as a fee, rather than the entire amount, for UTXOs that are slightly above the dust threshold.

These enhancements are beyond the scope of this initial BIP but represent potential future improvements to further address the dust UTXO problem.
