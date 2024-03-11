|     Author     |           Title            |  Category  |       Status        |    Date    |
| -------------- | -------------------------- | ---------- | ------------------- | ---------- |
| Matheus Franco | Network Topology           | Network    | open-for-discussion | 2024-03-06 |

## Summary

Modify the validator-topic assignment function using a greedy algorithm to minimize the number of non-committee messages an operator receives.

## Motivation

Currently, an operator needs to deal with the processing of several non-committee messages. The mean rate goes as high as $97$%. Even though non-committee messages require less cryptographic processing, in total, they may represent a scalability barrier.

## Rationale

The current topic assignment function is calculated using the validator public key modulo 128, which makes it similar to a *random* model.

```go
func ValidatorSubnet(validatorPKHex string) int {
	if len(validatorPKHex) < 10 {
		return -1
	}
	val := hexToUint64(validatorPKHex[:10])
	return int(val % subnetsCount)
}
```

The proposed model attempts to put validators with similar operator committees on the same topic.

## Viability

The model can be seen as a state machine that holds a validator-topic map and can be updated upon a validator addition or removal.

Any node in the network must agree on the current state so that the validator-topic map is consistent between nodes.

A state machine replication can be achieved if:
1. the replicas agree on the order of the events (validator addition and removal).
2. the transition function is deterministic.

Regarding condition (1), it's possible that a replica misses an event and produces an inconsistent state. To mitigate this issue, we can delay an event processing by some time interval to increase the probability of receiving all events before that. This is already included in the implementation which set the events to be processed only after 8 eth1 blocks.

Regarding condition (2), the operations in the transition function are well-defined and deterministic as we shall see next.

In our case, there are extra assumptions that must be met depending on the nature of the model. Namely, upon an event, we say that:
- a model is *unstable* if it may change the assignment of other validators added previously.
- a model is *stable* if it does not change the assignment of other validators added previously.

For a *stable* model, the only requirement is that the operators in the new validator's committee must have already processed the event by the time a duty arrives for such validator. But this condition is also necessary for the duty execution and is already established in the implementation.

For an *unstable* model, since nodes use the state to communicate on the correct topic, it's fundamental that the events are processed at the same time. Due to the possibility of unstable network conditions, we can't ensure that this condition holds by the time that the assignment of a changed validator is used. A mitigation technique is to set the state to be updated on pre-defined periods and execute only events older than a certain threshold. For example, it could be set that the state updates in the last slot of an epoch (to be used for the following epoch) for all events received up to the 22nd epoch's slot.

If one does not want to rely on these assumptions and mitigation techniques, it's also possible to add a consensus protocol for the entire network to agree on the replicated state machine.


## Algorithm

Below, it's shown the initialization algorithm in Go with complexity $\mathcal{O}(N + C \log C + t_{max} C)$ where $N$ is the number of validators, $C$ is the number of unique committees, and $t_{max}$ the number of topics.

```go
type TopicAssignment struct {
	topicMap         map[int]map[phase0.ValidatorIndex]struct{}
	operatorsInTopic map[int]map[types.OperatorID]struct{}
	maxTopics        int // Default: 128
}

func (t *TopicAssignment) Init(validatorCommittee map[phase0.ValidatorIndex][]types.OperatorID) {
	// Init topics
	topicIndex := 1
	for topicIndex <= t.maxTopics {
		t.topicMap[topicIndex] = make(map[phase0.ValidatorIndex]struct{})
		topicIndex+=1
	}

	// Init objects: existing committees and validators per committees
	var committees [][]types.OperatorID
	var committeeValidators map[int][]phase0.ValidatorIndex // Committee list index -> Validators

	// Sort committees by the number of validators in descending order
	committees, committeeValidators = t.SortCommitteesByValidators(validatorCommittee)

	// Insert the biggest committee in topic 1, the second biggest in topic 2, ..., until topic [maxTopics]
	committeeIndex := 0
	for committeeIndex < len(committees) {
		// Topic to be inserted
		insertionTopic := committeeIndex

		// Stop if reaches maxTopics
		if insertionTopic > t.maxTopics {
			break
		}

		// Insert validators in the topic
		for _, validatorIndex := range committeeValidators[committeeIndex] {
			t.topicMap[insertionTopic][validatorIndex] = struct{}{}
		}

		// insert committee operators
		for _, operatorID := range committees[committeeIndex] {
			t.operatorsInTopic[insertionTopic][operatorID] = struct{}{}
		}

		committeeIndex += 1
	}

	for committeeIndex < len(committees) {
		// Get topic with least impact
		topicWithLeastImpact := t.GetTopicWithLeastImpact(committees[committeeIndex], len(committeeValidators[committeeIndex]))

		// Insert validators in the topic
		for _, validatorIndex := range committeeValidators[committeeIndex] {
			t.topicMap[topicWithLeastImpact][validatorIndex] = struct{}{}
		}

		// insert committee operators
		for _, operatorID := range committees[committeeIndex] {
			t.operatorsInTopic[topicWithLeastImpact][operatorID] = struct{}{}
		}

		committeeIndex += 1
	}

}

func (t *TopicAssignment) GetTopicWithLeastImpact(committee []types.OperatorID, numberOfValidators int) int {
	// init best
	bestTopic := 1
	bestCost := math.MaxInt

	for topicIndex, topicValidators := range t.topicMap {
		// compute cost
		equalOperators := t.NumberOfIntersectingOperators(committee, t.operatorsInTopic[topicIndex]) // Compute intersection in O(n + m) time
		cost := (len(committee)-equalOperators)*len(topicValidators) + (len(t.operatorsInTopic[topicIndex])-equalOperators)*numberOfValidators

		// update
		if cost < bestCost {
			bestTopic = topicIndex
			bestCost = cost
		}
	}

	return bestTopic
}
```

### Addition Event

For the addition algorithm, there are two possible approaches.

#### Addition algorithm 1

If the new validator's committee already exists, the first algorithm adds the validator to its committee's topic, splits such topic and performs the *best aggregation*. The *best aggregation* operation compares the cost of doing an aggregation for each pair of committees ($\mathcal{O}(t_{max}^2)$) and performs the aggregation with minimal cost. The *split* operation creates two disjoint subsets of validators from a topic in a similar manner to the [algorithm](#algorithm) (for two meta-topics).

If the committee is new, it considers it as a new topic and performs the *best aggregation* operation.

Pseudo-code:
```R
1. procedure Add_Validator(Topics, Validator, committee)
2.      existsInTopic, topic <- Topics.Has(committee)
3.      if existsInTopic then
4.          Topics.AddValidator(topic, Validator)
5.          Topics.Split(topic)
6.          Topics.AggregationStep()
7.      else
8.          topic <- GenerateTopicID(committee)
9.          Topics.AddValidator(topic, Validator)
10.         Topics.AggregationStep()
```


#### Addition algorithm 2

If the new validator's committee already exists, the second algorithm adds the validator to its committee's topic.

If the committee is new, it finds the topic with minimal aggregation impact and inserts the validator in such topic ($\mathcal{O}(t_{max})$).

Pseudo-code:
```R
1. procedure Add_Validator(Topics, Validator, committee)
2.      existsInTopic, topic <- Topics.Has(committee)
3.      if existsInTopic then
4.          Topics.AddValidator(topic, Validator)
5.      else
6.          bestTopic <- Topics.GetTopicWithLeastImpact(committee, 1) # 1 validator
7.          Topics.AddValidator(bestTopic, Validator)
```

### Comparison

The first algorithm requires more processing time, produces a better solution in terms of non-committee messages, and produces an *unstable* model.

The second algorithm requires less processing time, produces a worse solution in terms of non-committee messages, and produces a *stable* model.

We propose going with the second solution since it produces a *stable* model.

### Removal Event

For the removal algorithm, there are also two possible approaches.

#### Removal algorithm 1

First, the algorithm removes the validator from the topic. Then, if the topic is not empty, the first algorithm splits the topic and performs the *best aggregation* operation ($\mathcal{O}(t_{max}^2)$). If the topic is empty, the first algorithm simply splits the topic.

Pseudo-code:
```R
1. procedure RemoveValidator(Validator, Topics)
2.      committee <- Committee(Validator)
3.      topic <- Topics.FindTopic(Validator)
4.      Topics.RemoveValidator(topic, Validator)
5.      if Topics.TopicLen(topic) > 0 then
6.          Topics.Split(topic)
7.          Topics.AggregationStep()
8.      else
9.          Topics.RemoveTopic(topic)
10.         Topics.SplitBestTopic()
```


#### Addition algorithm 2

The second algorithm just removes the validator from its assigned topic.

Pseudo-code:
```R
1. procedure Remove_Validator(V, Topics)
2.      committee <- Committee(V)
3.      topic <- Topics.FindTopic(V)
4.      Topics.RemoveValidator(topic, V)
```

### Comparison

The first algorithm requires more processing time, produces a better solution in terms of non-committee messages, and produces an *unstable* model.

The second algorithm requires less processing time, produces a worse solution in terms of non-committee messages, and produces a *stable* model.

We propose going with the second solution since it produces a *stable* model.


## Drawbacks

- A bigger number of nodes in a topic increases the reliability of a node receiving a published message by propagation in the network. For example, two geographically distant nodes may have connection problems and intermediary nodes may help propagate a message.
- Usually, more nodes in a network (or topic) represent a stronger defense against attacks. However, the model indirectly reduces the number of operators in some topics, which is understood to be a trade-off of this approach.



