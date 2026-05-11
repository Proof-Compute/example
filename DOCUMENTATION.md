# Deterministic Event System — Complete Documentation

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Architecture](#architecture)
3. [API Reference](#api-reference)
4. [Usage Examples](#usage-examples)
5. [Advanced Patterns](#advanced-patterns)
6. [Invariants & Guarantees](#invariants--guarantees)
7. [Limitations & Future Work](#limitations--future-work)

---

## Core Concepts

### The Central Invariant

```
S(t+1) = F(S(t), E(t))
```

**Where:**
- `S(t)` = immutable state at time t
- `E(t)` = ordered event at time t
- `F` = pure reducer function
- `S(t+1)` = new state derived from previous state + event

This means: **All state is purely derived from an immutable event log through deterministic reducers.**

### Event Sourcing

Instead of storing state directly, we store events:

```
Event Log: [Event₀, Event₁, Event₂, ..., Eventₙ]
State = fold(reducer, initialState, eventLog)
```

**Benefits:**
- Complete audit trail
- Perfect reproducibility
- Time-travel debugging
- Causality verification

### Pure Reducers

A reducer is a function with these properties:

```javascript
reducer(state, event) → newState
```

**Requirements:**
1. **Pure**: No side effects, no external I/O
2. **Deterministic**: Same input always produces same output
3. **Total**: Must handle all valid inputs
4. **Immutable**: Never mutates input state

### Content Addressing

Every state is identified by its content hash:

```
stateHash = SHA256(canonical(state))
```

Different order → same hash. Same content → same hash. Always.

This enables:
- Tampering detection
- Duplicate identification
- Cache-based optimization
- Integrity verification

### Causal Integrity

Events form a chain where each event references the previous:

```
Event[i].prevHash = Event[i-1].hash
Event[i].hash = SHA256(canonicalize(Event[i]))
```

This creates an immutable ledger where altering any event breaks the chain.

---

## Architecture

### System Layers

```
┌─────────────────────────────────────┐
│   DeterministicEventSystem (Main)   │  Coordinator
├─────────────────────────────────────┤
│  EventLog  │ ReducerEngine │ State  │  Core
├─────────────────────────────────────┤
│ Replay │ Branching │ Meta │ Proof   │  Advanced
├─────────────────────────────────────┤
│Canonicalize│Hash│Validate│Merge    │  Utilities
└─────────────────────────────────────┘
```

### Component Responsibilities

#### EventLog
- Maintains append-only history
- Enforces causal chain integrity
- Indexes events by hash
- Allows slice/query operations

```javascript
eventLog.append(event)           // Add event, verify chain
eventLog.getAt(index)            // Retrieve event by position
eventLog.getByHash(hash)         // Lookup by content hash
eventLog.verifyIntegrity()       // Check chain is unbroken
```

#### ReducerEngine
- Registry of transformation functions
- Applies reducers to state
- Validates reducer signatures
- Supports system evolution (meta-governance)

```javascript
reducerEngine.register(type, fn)  // Register reducer
reducerEngine.apply(state, event) // Apply to state
reducerEngine.has(type)           // Check if registered
reducerEngine.updateReducers()    // System evolution
```

#### StateSnapshot
- Immutable state representation
- Content hash for integrity
- Metadata (event count, timestamp)
- Verification capability

```javascript
snapshot = StateSnapshot.create(data, eventCount)
snapshot.verify()                 // Confirm hash matches data
snapshot.hash                     // Content-addressed hash
```

#### ReplayEngine
- Deterministic state reconstruction
- Replay from any point in history
- No snapshots required (but supported for optimization)

```javascript
state = replayEngine.replayFrom(initialState, upToIndex)
state = replayEngine.replayFromEvent(initialState, hash)
```

#### BranchingEngine
- Counterfactual exploration
- Fork state at specific points
- Explore hypothetical futures
- No impact on main timeline

```javascript
branchingEngine.createBranch(name, eventIndex)
branchingEngine.applyToBranch(name, event)
branchingEngine.exploreFuture(hypotheticalEvents)
```

#### MetaGovernance
- Controlled system evolution
- Reducer updates through events
- Epoch-based versioning
- Rollback capability

```javascript
proposal = metaGovernance.proposeReducerUpdate(newReducers)
metaGovernance.acceptReducerUpdate(newReducers)
metaGovernance.getEpochHistory()
```

#### ProofEngine
- Generate causality proofs
- Verify state outcomes
- Explain state transitions
- Trace causal chains

```javascript
proof = proofEngine.proveStateOutcome(index, expectedState)
explanation = proofEngine.explainStateTransition(from, to)
```

---

## API Reference

### DeterministicEventSystem

#### Constructor

```javascript
const system = new DeterministicEventSystem(initialState)
```

**Parameters:**
- `initialState` (object): Starting state. Defaults to `{}`

**Returns:** New system instance

#### registerReducer(eventType, reducerFn)

Register a handler for an event type.

```javascript
system.registerReducer('ADD_ITEM', (state, event) => ({
  ...state,
  items: [...state.items, event.payload.item]
}))
```

**Parameters:**
- `eventType` (string): Event type identifier
- `reducerFn` (function): Pure reducer function

**Throws:** Error if reducerFn is not a function

#### appendEvent(event)

Add event to log and derive new state.

```javascript
const { eventHash, stateHash } = system.appendEvent({
  type: 'ADD_ITEM',
  payload: { item: 'apple' },
  metadata: { source: 'user' }
})
```

**Parameters:**
- `event.type` (string, required): Event type
- `event.payload` (object, optional): Event data
- `event.metadata` (object, optional): Additional context
- `event.id` (string, optional): Unique ID, auto-generated if omitted
- `event.timestamp` (number, optional): Timestamp, auto-added if omitted

**Returns:** `{ eventHash, stateHash }`

**Throws:** 
- If no reducer registered for type
- If validation fails
- If reducer throws

#### getCurrentState()

Get the current state.

```javascript
const snapshot = system.getCurrentState()
// snapshot.data = actual state
// snapshot.hash = content hash
// snapshot.eventCount = total events applied
```

**Returns:** StateSnapshot object

#### getStateAt(eventIndex)

Reconstruct state at a specific event.

```javascript
const snapshot = system.getStateAt(5)  // State after event 5
```

**Parameters:**
- `eventIndex` (number): 0-based event index

**Returns:** StateSnapshot object

#### simulateFuture(hypotheticalEvents)

Explore what would happen with future events.

```javascript
const futureState = system.simulateFuture([
  { type: 'ADD_ITEM', payload: { item: 'banana' } },
  { type: 'REMOVE_ITEM', payload: { item: 'apple' } }
])
```

**Parameters:**
- `hypotheticalEvents` (array): Events to simulate

**Returns:** Resulting state (doesn't modify log)

#### proveOutcome(upToEventIndex, expectedState)

Prove that a state is correct given the event history.

```javascript
const proof = system.proveOutcome(10, { items: [...] })
// proof.proven: boolean
// proof.eventChain: causal path to state
```

**Parameters:**
- `upToEventIndex` (number): Event index to replay to
- `expectedState` (object): Expected resulting state

**Returns:** Proof object with proven, hashes, and event chain

#### explainTransition(fromIndex, toIndex)

Describe what events caused a state change.

```javascript
const explanation = system.explainTransition(5, 10)
```

**Parameters:**
- `fromIndex` (number): Starting event index
- `toIndex` (number): Ending event index

**Returns:** Explanation with transitioning events and hashes

#### verifyIntegrity()

Check that the entire event chain is unbroken.

```javascript
const result = system.verifyIntegrity()
// result.valid: boolean
// result.error: string (if invalid)
```

**Returns:** `{ valid, error? }`

#### getEventLog()

Get entire event history.

```javascript
const events = system.getEventLog()
// Array of all events
```

**Returns:** Array of event objects

#### getSystemStats()

Get overview of system state.

```javascript
const stats = system.getSystemStats()
// {
//   totalEvents: 42,
//   currentStateHash: "abc123...",
//   currentReducerVersion: 2,
//   currentEpoch: 1,
//   registeredReducers: ["EVENT_TYPE", ...]
// }
```

**Returns:** Stats object

---

## Usage Examples

### Example 1: Simple Counter

```javascript
const system = new DeterministicEventSystem({ count: 0 })

system.registerReducer('INCREMENT', (state, event) => ({
  count: state.count + event.payload.amount
}))

system.registerReducer('DECREMENT', (state, event) => ({
  count: state.count - event.payload.amount
}))

system.appendEvent({ type: 'INCREMENT', payload: { amount: 5 } })
system.appendEvent({ type: 'INCREMENT', payload: { amount: 3 } })
system.appendEvent({ type: 'DECREMENT', payload: { amount: 2 } })

console.log(system.getCurrentState().data)
// { count: 6 }
```

### Example 2: Todo List

```javascript
const system = new DeterministicEventSystem({ todos: [] })

system.registerReducer('ADD_TODO', (state, event) => ({
  todos: [...state.todos, {
    id: event.id,
    text: event.payload.text,
    done: false,
    createdAt: event.timestamp
  }]
}))

system.registerReducer('MARK_DONE', (state, event) => ({
  todos: state.todos.map(todo =>
    todo.id === event.payload.todoId
      ? { ...todo, done: true }
      : todo
  )
}))

system.appendEvent({ type: 'ADD_TODO', payload: { text: 'Buy milk' } })
system.appendEvent({ type: 'ADD_TODO', payload: { text: 'Walk dog' } })
system.appendEvent({ type: 'MARK_DONE', payload: { todoId: '...' } })
```

### Example 3: Bank Account

```javascript
const system = new DeterministicEventSystem({ 
  balance: 1000,
  transactions: []
})

system.registerReducer('DEPOSIT', (state, event) => ({
  balance: state.balance + event.payload.amount,
  transactions: [...state.transactions, {
    type: 'DEPOSIT',
    amount: event.payload.amount,
    timestamp: event.timestamp
  }]
}))

system.registerReducer('WITHDRAW', (state, event) => {
  if (state.balance < event.payload.amount) {
    throw new Error('Insufficient funds')
  }
  
  return {
    balance: state.balance - event.payload.amount,
    transactions: [...state.transactions, {
      type: 'WITHDRAW',
      amount: event.payload.amount,
      timestamp: event.timestamp
    }]
  }
})

// Try valid transaction
system.appendEvent({ type: 'DEPOSIT', payload: { amount: 500 } })

// Try invalid transaction (will throw)
try {
  system.appendEvent({ type: 'WITHDRAW', payload: { amount: 2000 } })
} catch (e) {
  console.log('Transaction rejected:', e.message)
}
```

### Example 4: Branching & Counterfactual Analysis

```javascript
const system = new DeterministicEventSystem({ funds: 1000 })

system.registerReducer('INVEST', (state, e) => ({
  funds: state.funds - e.payload.amount
}))

system.registerReducer('SAVE', (state, e) => ({
  funds: state.funds + e.payload.amount
}))

// Main path: invest
system.appendEvent({ type: 'INVEST', payload: { amount: 500 } })

// Create branch point
system.createBranch('alternative-path', 0)

// Explore: what if we had saved instead?
const alternative = system.simulateFuture([
  { type: 'SAVE', payload: { amount: 500 } }
])

console.log('Main path result:', system.getCurrentState().data)
// { funds: 500 }

console.log('Alternative path result:', alternative)
// { funds: 1500 }
```

### Example 5: Proving Causality

```javascript
// Set up system with some history
const system = new DeterministicEventSystem({ value: 100 })

system.registerReducer('MULTIPLY', (state, e) => ({
  value: state.value * e.payload.factor
}))

system.appendEvent({ type: 'MULTIPLY', payload: { factor: 2 } })
system.appendEvent({ type: 'MULTIPLY', payload: { factor: 3 } })

const expectedState = { value: 600 }

// Prove this state is correct
const proof = system.proveOutcome(1, expectedState)

console.log('Proven?', proof.proven)
// true

console.log('Event chain that led to this:')
proof.eventChain.forEach(e => {
  console.log(`  [${e.index}] ${e.type}`)
})
// [0] MULTIPLY
// [1] MULTIPLY

console.log('This proves the state HAD to be 600 given the event history')
```

---

## Advanced Patterns

### Pattern 1: Validation Before Append

```javascript
function validateAndAppend(system, event) {
  // Custom validation
  const errors = []
  
  if (!event.type) errors.push('Missing event type')
  if (!event.payload) errors.push('Missing payload')
  if (event.payload.amount < 0) errors.push('Amount must be positive')
  
  if (errors.length > 0) {
    throw new Error(`Validation failed: ${errors.join(', ')}`)
  }
  
  // If validation passes, append
  return system.appendEvent(event)
}
```

### Pattern 2: Reducer Composition

```javascript
function composeReducers(...reducers) {
  return (state, event) => {
    return reducers.reduce((s, reducer) => {
      try {
        return reducer(s, event)
      } catch (e) {
        return s  // If reducer throws, skip
      }
    }, state)
  }
}

system.registerReducer('COMBINED', composeReducers(
  (state, e) => ({ ...state, processed: true }),
  (state, e) => ({ ...state, count: state.count + 1 })
))
```

### Pattern 3: Snapshot for Performance

```javascript
function createSnapshot(system, afterEventIndex) {
  const state = system.getStateAt(afterEventIndex)
  
  // Save snapshot to storage
  saveToStorage({
    state: state.data,
    hash: state.hash,
    eventCount: state.eventCount
  })
  
  // Later, can bootstrap from this snapshot
  return state
}
```

### Pattern 4: Distributed System Merge

```javascript
const system1 = new DeterministicEventSystem({ items: [] })
const system2 = new DeterministicEventSystem({ items: [] })

// Both systems evolve independently
system1.appendEvent({ type: 'ADD', payload: { item: 'a' } })
system2.appendEvent({ type: 'ADD', payload: { item: 'b' } })

// Find common ancestor
const merge = DistributedMerge.mergeEventLogs(
  system1.getEventLog(),
  system2.getEventLog()
)

console.log('Divergence point:', merge.commonLength)
console.log('Conflicts:', merge.conflictingEvents)

// In practice: use causal ordering to resolve
```

### Pattern 5: Meta-Governance (System Evolution)

```javascript
const system = new DeterministicEventSystem({ count: 0 })

system.registerReducer('INCREMENT', (state, e) => ({
  count: state.count + 1
}))

// Evolve: add new reducer
system.acceptReducerUpdate({
  'DOUBLE': (state, e) => ({
    count: state.count * 2
  })
})

// Now DOUBLE is available
system.appendEvent({ type: 'DOUBLE' })
```

---

## Invariants & Guarantees

### Determinism Guarantee

```
∀ logs A, B:
A = B ⟹ apply(A) = apply(B)
```

Given identical event logs, identical state is guaranteed.

### Causal Integrity Guarantee

Every event in the log can be verified to have been added in sequence:

```
Event[i].prevHash = SHA256(Event[i-1])
```

Breaking this guarantee requires tampering with all subsequent events.

### Auditability Guarantee

Every state at every point in time is recoverable and verifiable:

```
state(t) = fold(reducer, initialState, eventLog[0..t])
```

### Immutability Guarantee

State is never mutated; new states are created:

```
newState = reducer(currentState, event)
// currentState is unchanged
```

### Content-Addressing Guarantee

Identical states always have identical hashes regardless of construction:

```
hash({ a: 1, b: 2 }) == hash({ b: 2, a: 1 })
```

---

## Limitations & Future Work

### Current Limitations

1. **Single-threaded**: Events apply sequentially. No concurrent event application.
2. **No networking**: This is a single-process implementation.
3. **Memory-only**: No persistence layer (but data is serializable).
4. **Synchronous only**: No async/await support in reducers.
5. **Total ordering required**: Events must have a global order.

### Future Enhancements

1. **Snapshots & Checkpoints**: Optimize replay for large histories
2. **Persistence**: Event log stored to disk/database
3. **Distributed Consensus**: Multi-node systems with causal ordering
4. **Async Reducers**: Support for I/O-based state transitions
5. **Time-travel UI**: Interactive timeline visualization
6. **Conflict Resolution**: Advanced CRDT semantics for divergence
7. **Compression**: Delta encoding for event logs
8. **Type Safety**: Full TypeScript implementation
9. **Schema Evolution**: Handle breaking changes to reducers
10. **Optimization Layers**: Memoization, incremental computation

---

## Implementation Notes

### Canonicalization

Objects are serialized deterministically:

```javascript
{ a: 1, b: 2 } → '{"a":1,"b":2}'
{ b: 2, a: 1 } → '{"a":1,"b":2}'
```

Keys are sorted alphabetically. No whitespace.

### Hashing

SHA256 over canonical form:

```
hash(obj) = SHA256(canonicalize(obj))
```

Same for both events and states. Provides collision-resistant integrity.

### Validation

Before appending, events are validated:

1. Schema check (required fields present)
2. Causality check (prevHash matches)
3. Reducer check (handler exists)
4. Dry-run (reducer doesn't throw)

### Error Handling

- Invalid events: Throw before appending (no log corruption)
- Reducer errors: Error message preserved, state unchanged
- Integrity violations: Detected on verify, never accepted

---

## Conclusion

This system unifies several concepts:

- **Event sourcing** (audit trail)
- **CQRS** (separate read/write)
- **Blockchain** (causal chain)
- **Determinism** (verifiable computation)
- **CRDTs** (distributed convergence)

Into a single, elegant model:

```
S(t+1) = F(S(t), E(t))
```

All state is derived. All state is verifiable. All transitions are provable.

For questions or contributions, see the Examples.js for working code.
