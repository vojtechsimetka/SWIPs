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
In the current Swarm design, accounting of the data exchanged between peers and the payment for such data is coupled. To promote widespread adoption of Swarm it is best to abstract the actual payment mechanism and let the nodes participating in the network to decide what payment system better adapts to their needs.

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->


## Motivation
<!--The motivation is critical for SWIPs that want to change the Swarm protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the SWIP solves. SWIP submissions without sufficient motivation may be rejected outright.-->
Nodes consuming and providing services in the Swarm network may benefit of settling their debts using different payment systems they might already be participating in, like payment channel networks (e.g. Lumino, Raiden or Lightning network). Storage providers offering other paid services will find appealing not to be forced to support multiple payment systems, but being able to consolidate the payments they receive using a single technology. As a side effect, new nodes can bootstrap its participation in a payment channel network by providing storage services in Swarm with zero cost of entry, as described in Generalised Swap Swear and Swindle games, 2019, Tron and Fischer. 

Defining and implementing the required APIs to achieve this decoupling enables Swarm to interoperate accross Blockchains without being tied to a specific Blockchain or settlement technology. Additionally, it enable multicurrency support for users to pay for the storage they consume.

## Specification
<!--The technical specification should describe the syntax and semantics of any new feature. The specification should be detailed enough to allow competing, interoperable implementations for the current Swarm platform and future client implementations.-->

In Swarm there is an abstraction for the accounting, the ```Balance``` interface defined in protocols.Balance:

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

The current Swap implementation uses Ether to settle debts and requires interactions with the SWAP chequebook smart contract. The settlement process is tightly coupled with the Swarm node, making hard to support other currencies besides Ether or other settlement methods such as payment channels. Moreover, this coupling tights Swarm to Ethereum-like Blockchains. Several options were considered to decouple the payments technology to use from Swarm:

* Introduce ERC20 support directly into the SWAP chequebook smart contract: It seems feasible to follow this path, however for each new token to be supported a new chequebook needs to be deployed or multiple tokens support needs to be introduced to the chequebook. While this is possible, it might introduce unwanted complexity to the SWAP chequebook.
* Introduce support for payment channels directly into the SWAP chequebook smart contract: This idea requires an additional level of abstraction for the cheques and the chequebook. The SWAP smart contract should be modified to directly interact with different on-chain payment mechanisms. In the case of payment channel networks cheques should be generalized to allow modeling Balance Proof. The interaction between the chequebook and the payment channel network will occur during the on-chain settlement, when the SWAP smart contract should send the Balance Proof to the payement channel smart contract(s) being use. As with the previous approach, this requires several changes to the SWAP chequebook smart contract. For every payment system to be supported a different chequebook should be designed. Having a single chequebook to handle multiple payment systems will result in an smart contract too difficult to maintain and keep secure.
* Completely replace SWAP by a different payment mechanism: this option is the least flexible of all since it does not solve the problem at all, it only changes the coupling with a given technology (SWAP) for another. Additionally it could hurt the Swarm network, forcing the appearance of multiple subnetworks, each one handling its own payment mechanism, a situation that still could happen with the current Swarm design.

## Backwards Compatibility
<!--All SWIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The SWIP must explain how the author proposes to deal with these incompatibilities. SWIP submissions without a sufficient backwards compatibility treatise may be rejected outright.-->
All SWIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The SWIP must explain how the author proposes to deal with these incompatibilities. SWIP submissions without a sufficient backwards compatibility treatise may be rejected outright.

## Test Cases
<!--Test cases for an implementation are mandatory for SWIPs that are affecting changes to data and message formats. Other SWIPs can choose to include links to test cases if applicable.-->

No test cases for this SWIP are provided at this moment.

## Implementation
<!--The implementations must be completed before any SWIP is given status "Final", but it need not be completed before the SWIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

No implementation for this SWIP is provided at this moment.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
