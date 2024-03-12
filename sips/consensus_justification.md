|     Author     |          Title          | Category |       Status        |    Date    |
| -------------- | ----------------------- | -------- | ------------------- | ---------- |
| Matheus Franco | Consensus Justification | Core     | open-for-discussion | 2024-03-08 |

## Summary

Drop decided messages and include a consensus justification nested in post-consensus messages.

## Motivation

Decided messages represent $16$% of the exchanged messages in the network and also significantly increase the thresholds of allowed messages in message validation. Furthermore, it's not a necessary part of the protocol and the speed-up that it provides can be provided by a simpler justification in the post-consensus message.

## Rationale

As it's done with the pre-consensus justification, that is included in the *Proposal* message, we propose including a consensus justification (an aggregated commit message) in the post-consensus message instead of sending decided messages.

First of all, the decided messages are useful to allow nodes to start the post-consensus phase if they are "behind" in the protocol execution due to not receiving messages. However, a consensus justification inside a post-consensus message allows a node to accomplish the same result while progressing even further due to the post-consensus signature. Moreover, this proposal aligns the design decisions of the (pre-consensus, consensus) and (consensus, post-consensus) inter-phases.

## Improvements

This proposal reduces the number of messages exchanged in the network by $16$%.

## Spec change

### Message structure

**Adding a justification in post-consensus messages**

Within a `SignedPartialSignatureMessage.PartialSignatureMessages`, all `PartialSignatureMessage` objects refer to the same `QBFT Instance`. Thus, we can simply add the justification to the `SignedPartialSignatureMessage` as follows:

```go
type SignedPartialSignatureMessage struct {
	Message   PartialSignatureMessages
	Signature Signature `ssz-size:"96"`
	Signer    OperatorID
    Justification *SignedMessage // Aggregated commit message (or []*SignedMessage if we ever use another asymmetric scheme that doesn't allow aggregation)
}
```

It could even be added to the `PartialSignatureMessages` types. The difference is that this would *hold* the `SignedPartialSignatureMessage.Signature` also to the consensus justification.

### New post-consensus logic

To implement the justification logic, once a post-consensus message is received, if the consensus has not been decided yet, we ask the consensus module to process a justification in a similar way that a decided message is processed (except y the feature of holding a state and broadcasting new decided messages). If the processing is successful, we update the runner state and continue with the current post-consensus logic.

```go
func (b *BaseRunner) basePostConsensusMsgProcessing(runner Runner, signedMsg *types.SignedPartialSignatureMessage) (bool, [][32]byte, error) {

	if !b.hasRunningDuty() {
		// do nothing
	}

	// If has not decided yet, process the justification
	if b.State.DecidedValue == nil && b.State != nil && b.State.RunningInstance != nil {
		// Process
		decidedMsg, err := b.QBFTController.ProcessConsensusJustification(signedMsg.Justification)
		if err != nil {
			// return error
		}

		// Decode
		decidedValue = &types.ConsensusData{}
		// ...

		// Set state
		b.highestDecidedSlot = decidedValue.Duty.Slot
		b.State.DecidedValue = decidedValue
	}
	// Follow-up with the current post-consensus processing
}
```

The justification may be large and there's no need for it in pre-consensus messages. Thus, we also recommend adding a check in the pre-consensus processing to assert that there's no consensus justification. This is useful as a part of a protection against buffer DoS attacks.

```go
func (b *BaseRunner) basePreConsensusMsgProcessing(runner Runner, signedMsg *types.SignedPartialSignatureMessage) (bool, [][32]byte, error) {
	if signedMsg.Justification != nil {
		// Return error
	}
}
```


## Drawbacks

- The post-consensus messages would be bigger due to the aggregated commit message.
- Exporters use decided messages to compute statistics of the network. Therefore, the proposal implies not only a protocol change but also changes in the exporter implementation.