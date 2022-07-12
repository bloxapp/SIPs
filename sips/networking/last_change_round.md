| Author     | Title                                             | Category | Status |
|------------|---------------------------------------------------|----------|--------|
| Niv Muroch | Handle non committee change round for all signers | Core     | Draft  |

**Summary**  
Allow nodes to save non committee (validators that are not belong to the node) change round messages in order to serve
other nodes when requesting last change round messages.

Rational & Design Goals  
When using subnets instead of topic per validator there is a need for supporting get last change round from any peer in
the subnet. in order to supports that, each node (peer) needs to handle incoming change round messages and save it per
signer and only the highest once. the reason for saving messages per signer is to keep the same behavior as it was with
validator topic.

In order to achieve change round quorum we need 3 different signers. other option is f+1 where we need 2 signers.
save change round per signer instead of the last one solve the that needs.

----------

**Specification**

**Networking**

*Saving*
Each operator peer listen for the non committee change round messages within it's subscribed subnets.
once receive non committee change round message, the following process run -  

- When local not exist. New message saves instantly.
- New change round message should be with higher or equal height than the local message.
  If higher, save instantly. if not, check if round is higher. if so, update.
- need to maintenance the last change round message for each of the signers in the specific validator committee.

```go
    res, err := c.changeRoundStorage.GetLastChangeRoundMsg(c.Identifier, msg.GetSigners()...)
	if err != nil {
		return errors.Wrap(err, "failed to get last change round msg")
	}

	logger := c.logger.With(zap.Any("signers", msg.GetSigners()))

	if len(res) == 0 {
		// no last changeRound msg exist, save the first one
		c.logger.Debug("no last change round exist. saving first one", zap.Int64("NewHeight", int64(msg.Message.Height)), zap.Int64("NewRound", int64(msg.Message.Round)))
		return c.changeRoundStorage.SaveLastChangeRoundMsg(msg)
	}
	lastMsg := res[0]
	logger = logger.With(
		zap.Int64("lastHeight", int64(lastMsg.Message.Height)),
		zap.Int64("NewHeight", int64(msg.Message.Height)),
		zap.Int64("lastRound", int64(lastMsg.Message.Round)),
		zap.Int64("NewRound", int64(msg.Message.Round)))

	if msg.Message.Height < lastMsg.Message.Height {
		// height is lower than the last known
		logger.Debug("new changeRoundMsg height is lower than last changeRoundMsg")
		return nil
	} else if msg.Message.Height == lastMsg.Message.Height {
		if msg.Message.Round <= lastMsg.Message.Round {
			// round is not higher than last known
			logger.Debug("new changeRoundMsg round is lower than last changeRoundMsg")
			return nil
		}
	}

	// new msg is higher than last one, save.
	logger.Debug("last change round updated")
	return c.changeRoundStorage.SaveLastChangeRoundMsg(msg)
```

*Syncing*
Last change round sync place on top of Libp2p stream channel between two peers, the request part and the handler.  
the request peer picks X random peers from the validator subnet `ssv.v1.1.[validator subnet]` and ask them for their last change round.

**Storage**
store interface
```go
// ChangeRoundStore manages change round data
type ChangeRoundStore interface {
	// GetLastChangeRoundMsg returns the latest broadcasted msg from the instance
	GetLastChangeRoundMsg(identifier message.Identifier, signers ...message.OperatorID) ([]*message.SignedMessage, error)
	// SaveLastChangeRoundMsg returns the latest broadcasted msg from the instance
	SaveLastChangeRoundMsg(msg *message.SignedMessage) error
	// CleanLastChangeRound cleans last change round message of some validator, should be called upon controller init
	CleanLastChangeRound(identifier message.Identifier)
}
```

Implementation
*key*
`lastChangeRoundKey` + `identifier` + `signer`

*value*
```go
type SignedMessage struct {
Message   *Message
Signer    types.OperatorID
Signature types.Signature
}
```
