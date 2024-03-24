| Author         | Title                 | Category | Status   |
| -------------- | --------------------- | -------- | -------- |
| Gal Rogozinski |  QBFT timeout | Core     | approved |


**Summary**  
Changes to the constant timeouts we defined in [SIP#6](https://github.com/bloxapp/SIPs/blob/main/sips/constant_qbft_timeout.md) that should improve performance. 

**Rational**  
Currently for all duties, we have a `quickTimeout` of 2 seconds before sending a `RoundChange` message and a `slowTimeout` of 2 minutes starting from the 8th round.The timer starts when the QBFT instance starts. This approach has 2 issues:

1. Duties that require a pre-consensus stage may start their instances at different times, causing some nodes to timeout sooner than others. This will result in a greater probability of a failed first round.
2. Different duties have different timing assumptions, which may cause some duties to execute earlier than others.



**Beacon Duty Timing Assumptions**  
The Beacon chain has some timing assumptions regarding duty execution for maximizing rewards.

| Duty           | Timing Assumption | Dependency     | Description                                                                                                         |
| -------------- | ----------------- | -------------- | ------------------------------------------------------------------------------------------------------------------- |
| Proposal       | < 4 Sec           | ---            | Attestations wait 4 seconds for a block                                                                             |
| Attester       | < 4 Sec           | Proposal       | Attestations wait 4 seconds for a block then need to be propogated within 4 seconds for aggregators to pick them up |
| Aggregator     | < 4 Sec           | Attester       | Aggregators start aggregating within 8 seconds from slot start and need to propogate within 4 seconds               |
| Sync Committee | < 4 sec           | Proposal       | Sync Committee wait 4 seconds for a block                                                                           |
| Contribution   | < 4 sec           | Sync Committee | Contributions wait 8 seconds from the slot start                                                                    |

        

**Specification**

From the slot start time after `baseDuration` delay wait 2 seconds before emitting a `RoundChange` message until the 6th round (inclusive). Afterwards, wait 2 minutes.

```go
var (
	// quickTimeoutThreshold is the round after which the timeout duration increases to 2 minutes
	quickTimeoutThreshold = Round(6)
	// quickTimeout is the timeout in seconds for the first 6 rounds
	quickTimeout int64 = 2 // 2 seconds
	// slowTimeout is the timeout in seconds for rounds after the first 6
	slowTimeout int64 = 120 // 2 minutes
)

// RoundTimeout returns the unix epoch time (seconds) in which we should send a RC message
// Called for all beacon duties other than proposals
func RoundTimeout(round Round, height Height, baseDuration int64, network types.BeaconNetwork) int64 {
	// Calculate additional timeout in seconds based on round
	var additionalTimeout int64
	additionalTimeout = int64(min(round, quickTimeoutThreshold)) * quickTimeout
	if round > quickTimeoutThreshold {
		slowPortion := int64(round-quickTimeoutThreshold) * slowTimeout
		additionalTimeout += slowPortion
	}

	// Combine base duration and additional timeout
	timeoutDuration := baseDuration + additionalTimeout

	// Get the unix epoch start time of the duty seconds
	dutyStartTime := network.EstimatedTimeAtSlot(phase0.Slot(height))

	// Calculate the time until the duty should start plus the timeout duration
	return dutyStartTime + timeoutDuration
}
```

*Proposals*

No changes from [SIP#6](https://github.com/bloxapp/SIPs/blob/main/sips/constant_qbft_timeout.md). Doesn't rely on the slot/duty start time.

*Attestations/Sync Committees*

Once the slot starts, wait 4 seconds for a block to be produced then start the timer according to the new rules.

```go
// AttestationOrSyncCommitteeTimeout returns the unix epoch time (seconds) in which we should send a RC message
func AttestationOrSyncCommitteeTimeout(round Round, height Height, network types.BeaconNetwork) int64 {
	return RoundTimeout(round, height, 4, network)
}
```

*Aggregator/Contribution*

Once the slot starts, wait 8 seconds for a block to be produced then start the timer according to the new rules

```go
// AggregationOrContributionTimeout returns the unix epoch time (seconds) in which we should send a RC message
func AggregationOrContributionTimeout(r Round, height Height, network types.BeaconNetwork) int64 {
	return RoundTimeout(r, height, 8, network)
}
```