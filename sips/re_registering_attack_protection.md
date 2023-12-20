| Author                      | Title                                     | Category         | Status              |
|-----------------------------|-------------------------------------------|------------------|---------------------|
| Lior Rutenberg (@lior-blox) | SSV Register Cluster copied shares attack | Core + Contracts | approved |

## Summary

A "copied shares" attack can occur when a malicious actor registers valid shares to a legitimate validator by either "frontrunning" a valid registration of a cluster transaction, or re-registering a previously removed validator.

In both scenarios, a validator would be operating on the SSV network under the control of a harmful user instead of the rightful validator owner.

#### Front-Running
In this scenario, a malicious actor could "front-run" an honest user trying to register a validator. The registration would appear as legitimate. To keep the attack ongoing, the attacker would cover the validator's fees.

To make any changes (such as removing or changing the cluster), the honest user would need to exit the validator.

#### Registration replay attack
In this case, an attacker could identify a validator that has been removed (from the SSV network) and quickly re-register it, thereby preventing the owner from transferring the validator elsewhere. Similar to the front-running scenario, the attacker would cover the validator's fees to keep the attack active.

In response, the honest user would need to exit the validator immediately.

## Rational & Design Goals
This SIP proposes a method to safeguard users by preventing unauthorized access to their validators by harmful entities. The proposal ensures that only the legitimate owner of a validator can enroll it in the SSV network. The process involves using the validator's private key to sign the address that is registering, along with a nonce.

The nonce is a necessary addition to prevent a unique edge case where, if a validator address that has been removed is compromised, a malicious party could potentially re-register it using the original address.

The BLS signature's authenticity will be confirmed by the SSV nodes, which will use the validator's public key and ensure that the nonce value is greater than the previous one. Instead of being on a validator level, the nonce will be on an account level to prevent scenarios where a validator is deregistered and re-registered by the same user.

> Note: While a validator's registration may be accepted at the contract level, it could be invalidated at the node level if the BLS signature verification fails - a process the contract cannot perform. Consequently, despite its accepted registration, the validator will not be able to execute its assigned duties due to this node-level validation failure.


## Specification
The specification is organized into three distinct sections:

1. Modifying the validator enrollment procedure to incorporate the validator's signature on the address.
2. Altering the validator map keys within the contract to include the address along with the public key.
3. Implementing signature verification within the SSV nodes during the validator registration process.


### SSV Contract

Changing the validator mapping to include the caller address in addition to the public key.

Including operators in the validator hashed value allows for the validation of a validator's affiliation with the cluster when it's being removed.
The last bit of the hashed value is used to indicate whether the validator is active or inactive.

From:
```solidity
    struct Validator {
        address owner;
        bool active;
    }
    mapping(bytes32 => Validator) public validatorPKs;
```
To: 
```solidity
    // Maps each validator's public key to its hashed representation of: operator Ids used by the validator and active / inactive flag (uses LSB)
    mapping(bytes32 => bytes32) validatorPKs;
    
    //example of setting the mapping
    function registerValidator(
        bytes calldata publicKey,
        uint64[] memory operatorIds,
        bytes calldata shares,
        bytes calldata signature, //signature on the address and nonce
        uint256 amount,
        Cluster memory cluster
    ) {
        ...
    bytes32 hashedPk = keccak256(abi.encodePacked(publicKey, msg.sender));
    validatorPKs[hashedPk] = bytes32(uint256(keccak256(abi.encodePacked(operatorIds))) | uint256(0x01)); // set LSB to 1
        ...
    }
```

### SSV Node

Every nonce associated with registration will be stored by the SSV node.

The nonce will be kept in a mapping that links addresses to their respective nonces.

The SSV node will confirm the signature of the address and nonce by using the validator's public key.

The nonce should be sequentially larger than the previously stored nonce.