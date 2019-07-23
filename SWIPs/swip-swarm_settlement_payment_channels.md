---
SWIP: <to be assigned>
title: Swarm settlement using layer 2 payment networks
author: Diego Masini (@diegomasini)
discussions-to: <URL>
status: Draft
type: Standards Track
category: Core
created: 2019-07-22
---

<!--You can leave these HTML comments in your merged SWIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new SWIPs. Note that a SWIP number will be assigned by an editor. When opening a pull request to submit your SWIP, please use an abbreviated title in the filename, `SWIP-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the SWIP.-->
Decouple the accounting of data done by SWAP from the actual payment, allowing payments to be performed by using either SWAP cheques and the SWAP smart contract or by using payment channel networks.

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->


## Motivation
<!--The motivation is critical for SWIPs that want to change the Swarm protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the SWIP solves. SWIP submissions without sufficient motivation may be rejected outright.-->
Nodes consuming and providing services in the Swarm network may benefit of settling their debts by using payment networks they might already be participating in. Allowing payments for storage to be done through payment channels may be attractive for new storage providers, this way storage providers can bootstrap its participation in the payment network by providing storage services in Swarm with zero cost of entry. 

Defining and implementing the required APIs to achieve this decoupling enables Swarm to interoperate accross Blockchains without being tied to a specific Blockchain or settlement technology.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for the current Swarm platform and future client implementations.-->

In Swarm there is an abstraction for the accounting, the Balance interface defined in protocols.Balance:

```golang
// Balance is the actual accounting instance
// Balance defines the operations needed for accounting
// Implementations internally maintain the balance for every peer
type Balance interface {
	// Adds amount to the local balance with remote node `peer`;
	// positive amount = credit local node
	// negative amount = debit local node
	Add(amount int64, peer *Peer) error
}
```

Swap (defined in swap/protocol.go) is an implementation of this interface and its Spec defines the ```EmitChequeMsg``` and the ```ChequeRequestMsg``` messages:

```golang
var Spec = &protocols.Spec{
	Name:       "swap",
	Version:    1,
	MaxMsgSize: 10 * 1024 * 1024,
	Messages: []interface{}{
		ChequeRequestMsg{},
		EmitChequeMsg{},
		ErrorMsg{},
		ConfirmMsg{},
	},
}
```

Which are handled by the devp2p Peer defined for the Swap protocol in swap/peer.go in by the ```handleEmitChequeMsg``` and ```handleChequeRequestMsg``` functions respectively:

```golang
func (sp *Peer) handleChequeRequestMsg(ctx context.Context, msg interface{}) (err error)
```

```golang
func (sp *Peer) handleEmitChequeMsg(ctx context.Context, msg interface{}) error 
```

This two functions are tightly coupled with Swap, which makes difficult to support different settlement strategies. 

The ```handleMsg``` function defined in swap/peer.go should delegate the processing of ```EmitChequeMsg``` and ```ChequeRequestMsg``` messages to a service implementing a payments API to decouple Swap from the payments processing. The existing code for ```handleChequeRequestMsg``` and ```handleEmitChequeMsg``` will become part of the SWAP implementation of this new API.

Upon connection and during the handshake, peers should indicate the supported payment methods, being the SWAP implementation of the payments API the default and fallback payment method to use.

Furthermore, nodes could support a combination of payment methods, expressing the order of preference during the handshake. If no indication of supported payment methods is sent, or if there is no match between the payment methods supported by the two nodes then all settlements will be done by using SWAP cheques and the SWAP smart contract.

## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

The current Swap implementation uses Ether to settle debts and requires interaction with the SWAP chequebook smart contract contract. The settlement process is tightly coupled with the Swarm node, making hard to support other settlement methods such as payment channels. Moreover, this coupling tights Swarm to Ethereum-like Blockchains. Several options were considered to decouple the Swarm from the payments technology:

The options considered to implement this feature were:
* Pay with ERC20 tokens: this requires to provide support to handle ERC20 tokens in the SWAP chequebook smart contract. It seems feasible to follow this path but it introduces unwanted complexity to the SWAP chequebook.
* Pay using a payment channel: this requires an additional level of abstraction for the cheques and the chequebook. The idea is for the SWAP smart contract to be able to directly interact with the layer 2 payment network smart contracts. In this case a SWARM node should be able to generate Offchain Balance Proof to use them as cheques, which in turn could be converted to Onchain Balance proof and be submitted to the Lumino / Raiden TokenNetwork smart contract. The SWAP smart contract would be responsible for calling one of the settle methods of the TokenNetwork smart contract.
* Replace SWAP by a Payment Network: this option is the least flexible of all and it could hurt the SWARM network, forcing the appearance of multiple subnetworks, each one handling its own payment mechanism.

Pay with Ether: this is the current implementation of the SWAP chequebook.


## Backwards Compatibility
<!--All SWIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The SWIP must explain how the author proposes to deal with these incompatibilities. SWIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
All SWIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The SWIP must explain how the author proposes to deal with these incompatibilities. SWIP submissions without a sufficient backwards compatibility treatise may be rejected outright.

## Test Cases
<!--Test cases for an implementation are mandatory for SWIPs that are affecting changes to data and message formats. Other SWIPs can choose to include links to test cases if applicable.-->
Test cases for an implementation are mandatory for SWIPs that are affecting changes to data and message formats. Other SWIPs can choose to include links to test cases if applicable.

## Implementation
<!--The implementations must be completed before any SWIP is given status "Final", but it need not be completed before the SWIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->
The implementations must be completed before any SWIP is given status "Final", but it need not be completed before the SWIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
