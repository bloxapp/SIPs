| Author      | Title        | Category | Status |
|-------------|--------------|----------|--------|
| Alon Muroch & Matheus Franco | Fork Support | Core     | approved  |

**Summary**  
Describes how to support forks in the SSV network.
A fork is a non backwards compatible change to the protocol, i.e., change message structs/ types/ functionality.

Forks are a bit tricky in SSV as there are multiple QBFT instances running with no way of knowing when each ends.  
For a forks to work they need a clear trigger.

**Triggering**  
A fork will be triggered by a beacon epoch number as its deterministic and easy to compute independently. If epoch E is the trigger epoch then pre-fork are all messages refering epoch < E and post-fork are all messages referring epoch >= E.

**Transition**  
Once Epoch E starts, the runner and QBFT controller change domain type from T to T' (the new fork domain). 

Client implementation should calculate when a new duty starts to decide if it starts with T or T', transition during duty execution is impossible. 

**History Sync**  
Sycing messages from a peer stays unchanged as messages have the identifier in them with domain type 


## Previous State

Currently, there's **no fork change logic**. However, there's **network and fork identification** through the `DomainType`. For instance, we have
```go
type DomainType [4]byte
var (
	GenesisMainnet = DomainType{0x0, 0x0, 0x0, 0x0}
	PrimusTestnet  = DomainType{0x0, 0x0, 0x1, 0x0}
	ShifuTestnet   = DomainType{0x0, 0x0, 0x2, 0x0}
	ShifuV2Testnet = DomainType{0x0, 0x0, 0x2, 0x1}
	V3Testnet      = DomainType{0x0, 0x0, 0x3, 0x1}
)
```
The _DomainType_ major importance is the fact that **it's mixed in the signature** such that messages on different domains will have different signatures.

The node takes one of the above _DomainType_ values and **holds it until termination**.

In the _DomainType_, the third byte corresponds to the **network** (Mainnet, Primus, etc.), and the fourth byte corresponds to the **fork or protocol version** for this network.

In the actual spec, _DomainType_ appears in the following structures:
- `Share`
- `Config`
- `Controller`

Below, you can see how **DomainType** is linked with the main structures.

<p align="center">
<img src="./images/fork_support/domain_type_presence.png"  width="30%" height="10%">
</p>

### Use cases

- When an _Instance_ needs to verify a signature. For example
```go
signedMessage.Signature.VerifyByOperators(signedMessage, config.GetSignatureDomainType(), types.QBFTSignatureType, operators);
```


- Each _Runner_, by creating the message ID when it needs to create a message. For example
```go
msgToBroadcast := &types.SSVMessage{
		MsgType: types.SSVPartialSignatureMsgType,
		MsgID:   types.NewMsgID(r.GetShare().DomainType, r.GetShare().ValidatorPubKey, r.BaseRunner.BeaconRoleType),
		Data:    data,
	}
```

- When the _BaseRunner_ validates a partial signature. For instance
```go
func (b *BaseRunner) validatePartialSigMsgForSlot(...) {
	...
	signedMsg.GetSignature().VerifyByOperators(signedMsg, b.Share.DomainType, types.PartialSignatureType, b.Share.Committee)
	...
}
```




## Proposed Changes

Along with the `DomainType`, we include a `NetworkID` type and a new `ForkData` structure.


```go
// NetworkID are intended to separate different SSV networks. A network can have many forks in it.
type NetworkID [1]byte

var (
	MainnetNetworkID = NetworkID{0x0}
	PrimusNetworkID  = NetworkID{0x1}
	ShifuNetworkID   = NetworkID{0x2}
	JatoNetworkID    = NetworkID{0x3}
	JatoV2NetworkID  = NetworkID{0x4}
)

// DomainType is a unique identifier for signatures, 2 identical pieces of data signed with different domains will result in different sigs
type DomainType [4]byte

// DomainTypes represent specific forks for specific chains, messages are signed with the domain type making 2 messages from different domains incompatible
var (
	GenesisMainnet = DomainType{0x0, 0x0, byte(MainnetNetworkID), 0x0}
	PrimusTestnet  = DomainType{0x0, 0x0, byte(PrimusNetworkID), 0x0}
	ShifuTestnet   = DomainType{0x0, 0x0, byte(ShifuNetworkID), 0x0}
	ShifuV2Testnet = DomainType{0x0, 0x0, byte(ShifuNetworkID), 0x1}
	JatoTestnet    = DomainType{0x0, 0x0, byte(JatoNetworkID), 0x1}
	JatoV2Testnet  = DomainType{0x0, 0x0, byte(JatoV2NetworkID), 0x1}
)

// ForkData is a simple structure holding fork information for a specific chain (and its fork)
type ForkData struct {
	// Epoch in which the fork happened
	Epoch phase0.Epoch
	// Domain for the new fork
	Domain DomainType
}

func (domainType DomainType) GetNetworkID() NetworkID {
	return NetworkID{domainType[2]}
}

func (networkID NetworkID) GetForksData() []*ForkData {
	switch networkID {
	case MainnetNetworkID:
		return mainnetForks()
	case PrimusNetworkID:
		return []*ForkData{{Epoch: 0, Domain: PrimusTestnet}}
	case JatoNetworkID:
		return []*ForkData{{Epoch: 0, Domain: JatoTestnet}}
	case JatoV2NetworkID:
		return []*ForkData{{Epoch: 0, Domain: JatoV2Testnet}}
	default:
		return []*ForkData{}
	}
}

// mainnetForks returns all forks for the mainnet chain
func mainnetForks() []*ForkData {
	return []*ForkData{
		{
			Epoch:  0,
			Domain: GenesisMainnet,
		},
	}
}

func (networkID NetworkID) DefaultFork() *ForkData {
	return networkID.GetForksData()[0]
}

// GetCurrentFork returns the ForkData with highest Epoch smaller or equal to "epoch"
func (networkID NetworkID) ForkAtEpoch(epoch phase0.Epoch) (*ForkData, error) {
	// Get list of forks
	forks := networkID.GetForksData()

	// If empty, raise error
	if len(forks) == 0 {
		return nil, errors.New("Fork list by GetForksData is empty. Unknown Network")
	}

	var current_fork *ForkData
	for _, fork := range forks {
		if fork.Epoch <= epoch {
			current_fork = fork
		}
	}
	return current_fork, nil
}

func (f ForkData) GetRoot() ([]byte, error) {
	byts, err := json.Marshal(f)
	if err != nil {
		return nil, errors.Wrap(err, "could not marshal ForkData")
	}
	ret := sha256.Sum256(byts)
	return ret[:], nil
}
```

### Structures access to types

Regarding the indirect usage of Identifiers:
- `Instance`:
  - Must keep its identifier to send properly formed messages.
- `Controller`:
  - Should have its _Identifier_ updated when a new fork occurs. For that, it will have a setter function. These functions will be used by the `Runner` to update the controller.
  - The _Domain_ field will drop. Since the _config_ also has a _domain_, it's duplicated.

```go

type Controller struct {
	Identifier []byte
	Height     Height
	StoredInstances InstanceContainer
	Share           *types.Share
	config          IConfig
	// No more Domain
}

func (c *Controller) SetIdentifier(identifier []byte) {
	c.Identifier = identifier
}
```

The following function gets the appropriate domain type, given the BeaconNetwork and the NetworkID.
```go
// GetDomainTypeAtSlot returns the domain type for a given slot, given a BeaconNetwork and a NetworkID
func GetDomainTypeAtSlot(beaconNetwork types.BeaconNetwork, networkID types.NetworkID, slot phase0.Slot) (types.DomainType, error) {
	epoch := beaconNetwork.EstimatedEpochAtSlot(slot)
	fork, err := networkID.ForkAtEpoch(epoch)
	if err != nil {
		return networkID.DefaultFork().Domain, errors.Wrap(err, "Could not get fork for epoch.")
	}
	return fork.Domain, nil
}
```
