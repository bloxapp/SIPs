| Author      | Title                          | Category | Status |
|-------------|--------------------------------|----------|--------|
| Alon Muroch | Voluntary Exit | Core     | Draft  |


**Summary**
Describes how the ssv operators performs a voluntary exit for a validator.  
<em>Important to note that a user holding the validator’s private key can sign a voluntary exit as well.</em>


**Specification**

**Networking**
A dedicated subnet for voluntary exits will be added to the network for all exit related communication between operators.
All operators subscribe to the network.

**Messages**
New SSV MsgType is introduced

| Name       | Type    | Value | Description                          |
|------------|---------|-------|--------------------------------------|
| VoluntaryExitMsgType | MsgType | 4     | SSVMessage type for voluntary exit messages |

The SSVMessage Data field will encode the voluntary exit SignedMessage data structure. 

```go
type VoluntaryExitMessage struct {
    MsgType MsgType
    Identifier    RequestID
    Data    []byte
}

type SignedVoluntaryExitMessage struct {
    Message   *Message
    Signer    types.OperatorID
    Signature types.Signature
}
```

Voluntary exit Message types

| Name               | Type    | Value | Description                   |
|--------------------|---------|-------|-------------------------------|
| InitMsgType        | MsgType | 0     | Voluntary exit initial request                |
| PartialSigMsgType    | MsgType | 1     | Partial signature for the exit request        |

**RequestID**  
Is a unique identifier for the entire voluntary exit process which is a combination of the signing eth address of the message and an incremental index.

_**Requires Message.ID.GetETHAddress() == singing key**_

**Signing key**  
Messages are signed with the ECDSA key that registered the validator..

**Initializing a voluntary exit**  
```go
// VoluntaryExitInit is the first message sent to initiate a voluntary exit signature from operators
type VoluntaryExitInit struct {
  ValidatorPK types.ValidatorPK
  ExitMessage *spec.VoluntaryExit
}
```

**Partial voluntary exit signature**  
Operators assigned to the validatorPK in the init message will respond with a partial signature message over the exit message provided in the init message. 
```go
// VoluntaryExitPartialSig is sent by each operator in response to VoluntaryExitInit
type VoluntaryExitPartialSig struct {
    Signature types.Signature
}
```

**Signature Reconstruction**  
The user initiating the exit request will listen on the voluntary exit topic for partial signatures, verify them and reconstruct a valid exit message signature from them to be broadcasted to the ethereum network.
