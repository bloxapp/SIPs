| Author         | Title                  | Category | Status              |
| -------------- | ---------------------- | -------- | ------------------- |
| Gal Rogozinski | Verifiable Attestation | Core     | open-for-discussion |

## Summary

Add a `SignedBeaconBlockHeader` to attestation consensus data. Enable a consensus data value check that will ensure the attestation or sync committee  data will be accepted by the beacon chain.

## Rationale

Currently the consensus leader is trusted for creating valid, timely data for the latest state of the beacon chain. By making the data verifiable the trust assumption can be removed. 

This will open a pathway to future leaderless designs where the committee can pick the best value. 

## Design

The design is best achievable via a hard fork.
Introduce a new `VerifiableConsensusData` type.

```go
type VerifiableConsensusData struct {
    Duty    spec.Duty
    Version spec.DataVersion
    Proof   []phase0.SignedBeaconBlockHeader `ssz-max:"3"`
    DataSSZ []byte `ssz-max:"128"` // attestation data max size
}
```

All Attestation and SyncCommittee logic should be moved to work with the new type.

```go
func (cd VerifiableConsensusData) GetAttestationData (att phase0.AttestationData,
                                                     headProof phase0.SignedBeaconBlockHeader, 
                                                     targetProof phase0.SignedBeaconBlockHeader, 
                                                     sourceProof phase0.SignedBeaconBlockHeader, 
                                                     error) {
    if err := att.UnmarshalSSZ(cd.DataSSZ); err != nil {
        return nil, nil, nil, nil, errors.Wrap(err, "could not unmarshal attestation ssz")
    }
    // Head vote proof
    if err := headProof.UnmarshalSSZ(cd.Proof[0]); err != nil {
        return nil, nil, errors.Wrap(err, "could not unmarshal head vote proof ssz")
    }
    // Target proof
    if err := targetProof.UnmarshalSSZ(cd.Proof[1]); err != nil {
        return nil, nil, errors.Wrap(err, "could not unmarshal target proof ssz")
    }
    // Target proof
    if err := sourceProof.UnmarshalSSZ(cd.Proof[1]); err != nil {
        return nil, nil, errors.Wrap(err, "could not unmarshal source proof ssz")
    }
    return att, headProof, targetProof, sourceProof, nil
}

func (cd VerifiableConsensusData) GetSyncCommitteeBlockRoot (phase0.Root, phase0.SignedBeaconBlockHeader, error) {
    ret := SSZ32Bytes{}
    if err := ret.UnmarshalSSZ(cd.DataSSZ); err != nil {
        return nil, errors.Wrap(err, "could not unmarshal ssz")
    }
    proof := phase0.SignedBeaconBlockHeader
    if err := proof.UnmarshalSSZ(cd.Proof); err != nil {
        return nil, nil, errors.Wrap(err, "could not unmarshal proof ssz")
    }
    return phase0.Root(ret), proof, nil
}

func (cd VerifiableConsensusData) Validate error {
    switch cid.Duty.Type {
    case BNRoleAttester:
            if _, _, _, _, err := cid.GetAttestationData(); err != nil {
                return err
            }
            return nil
    case BNRoleSyncCommittee:
            if _, _, err := cid.GetSyncCommitteeBlockRoot(); err != nil {
                return err
            }
            return nil
        default:
        return errors.New("unknown duty role")
    }
}
```

In addition we do the following changes to value checks mechanisms:

- `verifyProof` - Verifies that the give blockheader's root is the same as attestation data. Asserts that the block header was signed correctly by the encoded proposer.
- `verifyAssignedBlockHeader` - Verifies that the encoded proposer in the block header was indeed assgined by the beacon chain for the slot given in the header.
- `beaconChecks` (attestations only) - Ensures that the beacon chain will accept the attestation. See the [CL spec](https://github.com/ethereum/consensus-specs/blob/dev/specs/deneb/beacon-chain.md#modified-process_attestation).


```go
// verifyProof verifies that the block header has the expected root and was signed by the validator encoded in the header
func verifyProof(expectedRoot phase0.Root, proof phase0.SignedBeaconBlockHeader) error {
    beaconRoot := proof.Message.HashRoot()
    if !bytes.Equal(expectedRoot, beaconRoot) {
        return errors.New("unexpected header root")
    }

    blockEpoch := getBeaconNode().getEpochBySlot(proof.Message.slot)
    domain := GetBeaconNode().DomainData(blockEpoch, spec.DomainProposer)
    root := spec.ComputeEthSigningRoot(beaconRoot, domain)

    //TODO change head to current slot?
    validator := GetBeaconNode.GetValidator("HEAD", proof.Message.ProposerIndex)
    return bls.Verify(sig, validator.PubKey, root)
}

// verifyBlockHeader verifies that the block header was created by the correctly assigned validator
// proposerDuties should contain all the duties from the last 2 epochs.
func verifyAssignedBlockHeader(blockHeader *phase0.BeaconBlockHeader, proposerDuties []ethApi.ProposerDuty) error {
    for _, proposerDuty := range proposerDuties {
        if proposerDuty.Slot == blockHeader.Slot {
            if proposerDuty.ValidatorIndex == blockHeader.ProposerIndex {
                return nil
            }
            else {
                return errors.New("blockheader proof has unexpected proposer index")
            }
        }
    }
    return errors.New("blockheader proof's slot is not in last 2 epochs")
}

//beaconChecks ensures that the beacon chain will accept the attestation
func beaconChecks(cd *VerifiableConsensusData) error {
        attestationData, headProof, targetProof, sourceProof, err := cd.GetAttestationData() // error checked in cd.validate()
        
        if cd.Duty.Slot != attestationData.Slot {
            return errors.New("attestation data slot != duty slot")
        }

        if cd.Duty.CommitteeIndex != attestationData.Index {
            return errors.New("attestation data CommitteeIndex != duty CommitteeIndex")
        }

        if attestationData.Target.Epoch < network.EstimatedCurrentEpoch()-1 {
            return errors.New("attestation data target epoch is into far past")
        }

        // Addition
        if attestationData.Target.Epoch != getBeaconNode().EpochFromSlot(headProof.Message.Slot) {
            return errors.New("Target epoch should be the same as the head vote epoch")
        }
        
        //Addition
        if attestationData.Target.Epoch != getBeaconNode().EpochFromSlot(targetProof.Message.Slot) {
            return errors.New("Target epoch doesn't match proof")
        }
        

        if attestationData.Source.Epoch >= attestationData.Target.Epoch {
            return errors.New("attestation data source > target")
        }
        
        // Addition
        if attestationData.Source.Epoch != getBeaconNode().EpochFromSlot(sourceProof.Message.Slot) {
            return errors.New("Source epoch doesn't match proof")
        }
        
        // Addition 
        return isSourceJustified(attestationData.Source))
}

// isSourceJustified checks against the beacon node if the source is justified. Part of beaconChecks
func isSourceJustified(attestationData)) error {
    currentEpoch :=  network.EstimatedCurrentEpoch() 
    // https://ethereum.github.io/beacon-APIs/#/Beacon/getStateFinalityCheckpoints
    checkpoints := getBeaconNode().getFinalityCheckpoints()
    if attestationData.Target.Epoch == currentEpoch {
        if attestationData.Source.Root != checkpoints.data.currentJustifiedRoot {
            return Errors.New("source is not expected currently justified root")
        }
    } else if attestationData.Source.Root!= checkpoints.data.previousJustifiedRoot {
        return Errors.New("source is not expected previously justified root")
    }

    return nil
}

func AttesterValueCheckF(
    signer types.BeaconSigner,
    network types.BeaconNetwork,
    validatorPK types.ValidatorPK,
    validatorIndex phase0.ValidatorIndex,
    sharePublicKey []byte,
    // obtained from the operator's beacon node, consists of duties for the last 2 epochs
    proposerDuties []ethApi.ProposerDuty 
    // obtained from the operator's beacon node
    attestationDataByts []byte,
) qbft.ProposedValueCheckF {
    return func(data []byte) error {
        // addition
        if bytes.Equal(attestationDataByts, data) {
            return nil
        }

        cd := types.VerifyableConsensusData{}
        if err := cd.Decode(data); err != nil {
            return errors.Wrap(err, "failed decoding consensus data")
        }
        if err := cd.Validate(); err != nil {
            return errors.Wrap(err, "invalid value")
        }

        if err := dutyValueCheck(&cd.Duty, network, types.BNRoleAttester, validatorPK, validatorIndex); err != nil {
            return errors.Wrap(err, "duty invalid")
        }

        if err := beaconChecks(&cd); err != nil {
            return errors.Wrap(err, "beacon checks failed")
        }

        if err := signer.IsAttestationSlashable(sharePublicKey, attestationData); err != nil {
            return err
        }
        
        // Addition
        // heavy checks should be in the end
        // can be optimized with batch verification
        if err := verifyProof(attestationData.BeaconBlockRoot, headProof); err != nil {
            return errors.Wrap("invalid head vote proof", err)
        }
        
        if err := verifyProof(attestationData.Target.Root, targetProof); err != nil {
            return error.Wrap("invalid target vote proof", err)
        } 
        
        if err := verifyProof(attestationData.source.Root, sourceProof); err != nil {
            return error.Wrap("invalid target vote proof", err)
        } 
        
        return nil
    }
    
    func SyncCommitteeValueCheckF(
    signer types.BeaconSigner,
    network types.BeaconNetwork,
    validatorPK types.ValidatorPK,
    validatorIndex phase0.ValidatorIndex,
    // obtained from the operator's beacon node, consists of duties for the last 2 epochs
    proposerDuties []ethApi.ProposerDuty 
    // obtained from the operator's beacon node
    syncCommitteeBlockRoot []byte,
) qbft.ProposedValueCheckF {
    return func(data []byte) error {
        // addition
        if bytes.Equal(syncCommitteeBlockRoot, data) {
            return nil
        }

        cd := types.VerifiableConsensusData{}
        if err := cd.Decode(data); err != nil {
            return errors.Wrap(err, "failed decoding consensus data")
        }
        if err := cd.Validate(); err != nil {
            return errors.Wrap(err, "invalid value")
        }

        if err := dutyValueCheck(&cd.Duty, network, types.BNRoleSyncCommittee, validatorPK, validatorIndex); err != nil {
            return errors.Wrap(err, "duty invalid")
        }
        
        root, proof, _, cd.GetSyncCommitteeBlockRoot \\ checked on validate

        verifyAssignedBlockHeader(headProof, proposerDuties)

        if err := verifyProof(root, headProof); err != nil {
            return errors.Wrap("invalid head vote proof", err)
        }	
    }
}
    
```

In addition currently value checks are being performed in 2 places:
1. Upon starting a new instance
2. Upon validating received`qbft.Proposal` messages.

For all beacon duties besides block proposal, it can be assumed the source of the proposed consensus data (beacon node) is trusted by the operator and thus the first check (upon starting a new instance).

## Drawbacks

Currently this adds extra BLS checks. It is manadatory to alleviate signature verification load before proceeding.