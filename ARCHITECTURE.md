# Deterministic Event System — Architecture & Design

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│         DETERMINISTIC EVENT SYSTEM (DES)                    │
│                                                              │
│  "All state is a pure function of the immutable event log" │
└─────────────────────────────────────────────────────────────┘

                        THE CORE INVARIANT

                    S(t+1) = F(S(t), E(t))

                where:
                  S = immutable state
                  E = ordered event
                  F = pure reducer function
```

---

## Conceptual Layers

```
┌──────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                     │
│  (Your domain models: todos, bank accounts, etc.)       │
├──────────────────────────────────────────────────────────┤
│                   REDUCER DEFINITIONS                    │
│  (Pure functions: INCREMENT, ADD_ITEM, WITHDRAW)        │
├──────────────────────────────────────────────────────────┤
│               DETERMINISTIC EVENT SYSTEM                 │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │ EventLog      ReplayEngine    BranchingEngine   │    │
│  │ (history)     (reconstruct)   (counterfactual) │    │
│  │                                                │    │
│  │ StateSnapshot ReducerEngine   ProofEngine      │    │
│  │ (content-addressed) (dispatch)  (causality)    │    │
│  │                                                │    │
│  │ MetaGovernance      ValidationContract        │    │
│  │ (evolution)         (admission control)       │    │
│  └─────────────────────────────────────────────────┘    │
├──────────────────────────────────────────────────────────┤
│                    UTILITIES LAYER                       │
│  canonicalize() | hash() | verify() | merge()          │
├──────────────────────────────────────────────────────────┤
│                  STORAGE / PERSISTENCE                   │
│        (In-memory now, can extend to disk/DB)           │
└──────────────────────────────────────────────────────────┘
```

---

## Data Flow Diagram

### Adding an Event (Happy Path)

```
┌─────────────┐
│ New Event   │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────┐
│ 1. VALIDATE                         │
│   • Schema check                    │
│   • Causal integrity check          │
│   • Reducer exists?                 │
│   • Payload valid?                  │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│ 2. DRY-RUN SIMULATION               │
│   • Apply reducer to current state  │
│   • Does it throw? Reject if so     │
│   • Verify result is serializable   │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│ 3. APPEND TO LOG                    │
│   • Compute event hash              │
│   • Chain to previous event         │
│   • Index by hash                   │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│ 4. UPDATE STATE                     │
│   • Actually apply reducer          │
│   • Create new StateSnapshot        │
│   • Compute content hash            │
│   • Update currentState             │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│ RETURN { eventHash, stateHash }     │
└─────────────────────────────────────┘
```

### Querying History (Replay)

```
┌─────────────────────────────────────┐
│ Query: "What was state at event 5?" │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│ Get initial state                   │
│ Start with S₀                       │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│ Deterministically apply:            │
│ S₁ = F(S₀, E₀)                      │
│ S₂ = F(S₁, E₁)                      │
│ S₃ = F(S₂, E₂)                      │
│ ...                                 │
│ S₅ = F(S₄, E₄)                      │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│ Compute hash of S₅                  │
│ Verify matches expected (if given)  │
└──────┬──────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────┐
│ RETURN StateSnapshot(S₅, hash, 5)   │
└─────────────────────────────────────┘

** KEY PROPERTY: Same event log → same state ALWAYS **
```

---

## Component Architecture

### EventLog

```javascript
class EventLog {
  events: Event[]              // Append-only history
  hashIndex: Map<hash, Event>  // Fast lookup
  
  append(event)
    → Validate causal chain
    → Compute hash
    → Append to events
    → Index by hash
    
  getAt(index)
    → O(1) lookup by position
    
  getByHash(hash)
    → O(1) lookup by content hash
    
  verifyIntegrity()
    → Check: event[i].prevHash == event[i-1].hash
    → Check: event[i].hash == compute_hash(event[i])
}
```

**Invariants:**
- Events always append, never modify
- No gaps in sequence
- Chain unbroken: every event points to previous

### ReducerEngine

```javascript
class ReducerEngine {
  reducers: Map<type, Function>  // Type → pure function
  version: number                // Incremented on updates
  
  register(type, fn)
    → Validate fn is function
    → Store in map
    
  apply(state, event)
    → Look up reducer for event.type
    → Call reducer(state, event)
    → Catch and re-throw errors
    → Return new state
    
  updateReducers(newMap)
    → Replace entire reducer set
    → Increment version (for epochs)
}
```

**Invariants:**
- All reducers are pure functions
- No side effects allowed
- Errors don't mutate state
- System can evolve safely

### StateSnapshot

```javascript
class StateSnapshot {
  data: any              // Actual immutable state
  hash: string           // SHA256(canonical(data))
  eventCount: number     // How many events led here
  timestamp: number      // When snapshot created
  
  verify()
    → Recompute hash
    → Compare with stored hash
    → Return true if match
}
```

**Invariants:**
- Hash uniquely identifies state content
- Same state → same hash
- Hash verification detects tampering

### ReplayEngine

```javascript
class ReplayEngine {
  eventLog: EventLog
  reducerEngine: ReducerEngine
  
  replayFrom(initialState, upToIndex)
    state = initialState
    for i = 0 to upToIndex:
      state = reducerEngine.apply(state, eventLog.getAt(i))
    return state
    
  replayFromEvent(initialState, hash)
    → Find event by hash
    → Figure out its index
    → Call replayFrom(initialState, index)
}
```

**Invariants:**
- Same event log → same replay result
- No snapshots needed (but can optimize)
- Complete auditability

### BranchingEngine

```javascript
class BranchingEngine {
  branches: Map<name, Branch>
  
  Branch {
    state: StateSnapshot
    branchPoint: number      // Index where branch started
    events: Event[]          // Events applied to branch
  }
  
  createBranch(name, eventIndex)
    → Replay to eventIndex
    → Store as new branch
    → Branch doesn't affect main log
    
  applyToBranch(name, event)
    → Apply event to branch state only
    → Doesn't touch main log
    
  exploreFuture(events)
    → Simulate without branching
    → Return final state
    → Don't persist
}
```

**Invariants:**
- Branches are independent from main log
- Never affect main timeline
- Can explore infinite counterfactuals

### MetaGovernance

```javascript
class MetaGovernance {
  epochs: Epoch[]
  
  Epoch {
    number: number
    createdAt: timestamp
    reducerVersion: number
    previousVersion?: number
  }
  
  proposeReducerUpdate(newReducers)
    → Validate new reducers
    → Return proposal for review
    → (Not yet activated)
    
  acceptReducerUpdate(newReducers)
    → Update reducer engine
    → Create new epoch
    → Future events use new reducers
    
  getEpochHistory()
    → Return all epochs
    → Track when system evolved
}
```

**Invariants:**
- Old events always use old reducers
- Epoch defines which reducers were active
- System evolution is itself event-driven

### ProofEngine

```javascript
class ProofEngine {
  eventLog: EventLog
  reducerEngine: ReducerEngine
  
  proveStateOutcome(index, expectedState)
    → Replay to index
    → Compare with expected
    → Return proof with event chain
    
  explainTransition(fromIndex, toIndex)
    → Get events between indices
    → Compute state at each point
    → Show what changed and why
}
```

**Invariants:**
- Proof is deterministic
- Can be independently verified
- Proves "this state HAD to be this way"

### ValidationContract

```javascript
class ValidationContract {
  eventSchemas: Map<type, Schema>
  
  registerSchema(type, schema)
    → Store validation rules
    
  validate(event)
    → Check required fields
    → Check type-specific schema
    → Return { valid, errors[] }
}
```

**Invariants:**
- Validation is deterministic
- Rejects invalid events before append
- Guarantees only valid events enter log

---

## Design Decisions

### Why Append-Only?

```
✓ Prevents corruption
✓ Complete history preserved
✓ Enables perfect replay
✓ Simplifies integrity checks
✗ Storage grows unbounded (solve with snapshots)
```

### Why Pure Functions?

```
✓ Deterministic
✓ Testable
✓ Composable
✓ No hidden state
✓ Verifiable
✗ Can't do I/O (that's fine, separate concern)
```

### Why Content Addressing?

```
✓ Tamper detection
✓ Duplicate identification
✓ Cache optimization
✓ Order-independent hashing
```

### Why Snapshots Optional?

```
✓ System works without them
✓ Can optimize when needed
✓ No complexity by default
✓ Add incrementally
```

### Why Causal Chain?

```
✓ Detects tampering
✓ Proves ordering
✓ Enables distributed consensus
✓ Foundation for proofs
```

---

## Execution Model

### Event Processing Pipeline

```
Input Event
    │
    ▼
┌──────────────────────────────────────┐
│ GATE 1: VALIDATION                   │
│ • Schema valid?                      │
│ • Reducer exists?                    │
│ • Causal chain intact?               │
└──────┬───────────────────────────────┘
       │ ✓ Pass
       ▼
┌──────────────────────────────────────┐
│ GATE 2: DRY-RUN                      │
│ • Apply reducer to current state     │
│ • Does it throw?                     │
│ • Result serializable?               │
└──────┬───────────────────────────────┘
       │ ✓ Pass
       ▼
┌──────────────────────────────────────┐
│ GATE 3: APPEND                       │
│ • Add to event log                   │
│ • Update hash chain                  │
│ • Index for lookup                   │
└──────┬───────────────────────────────┘
       │ ✓ Success
       ▼
┌──────────────────────────────────────┐
│ GATE 4: STATE UPDATE                 │
│ • Apply reducer for real             │
│ • Create new StateSnapshot           │
│ • Update currentState reference      │
└──────┬───────────────────────────────┘
       │
       ▼
  Return { eventHash, stateHash }

At any GATE, if validation fails → Error, no mutation
```

---

## Correctness Properties

### Property 1: Determinism

```
Given event log E and initial state S₀:
  state₁ = apply(E, S₀)
  state₂ = apply(E, S₀)  (same computation, different time)
  
  Always: state₁ == state₂ (same hash)
```

**Enforced by:**
- Pure reducers
- Deterministic hashing
- Immutable state

### Property 2: Causality

```
event[i].prevHash always equals hash(event[i-1])

This forms a chain:
  E₀ ← E₁ ← E₂ ← ... ← Eₙ

Breaking the chain requires rewriting all subsequent events.
```

**Enforced by:**
- Hash chain validation on append
- verifyIntegrity() checks

### Property 3: Auditability

```
For any state S at time t:
  Can reproduce: S = fold(apply, S₀, E[0..t])
  Can verify: S.hash matches expected
  Can explain: Show events that led to S
```

**Enforced by:**
- ReplayEngine
- ProofEngine
- Immutable log

### Property 4: Immutability

```
Reducers never mutate input state.
New state is always created:

  newState = reducer(oldState, event)
  // oldState is unchanged
  // newState is new object
```

**Enforced by:**
- Implementation convention
- Testing

### Property 5: Content Addressing

```
hash({a: 1, b: 2}) == hash({b: 2, a: 1})

Because canonicalization sorts keys:
  canonical({a:1,b:2}) = '{"a":1,"b":2}'
  canonical({b:2,a:1}) = '{"a":1,"b":2}'
```

**Enforced by:**
- canonicalize() function
- Deterministic ordering

---

## Performance Considerations

### Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Append Event | O(1) | Hash + append to array |
| Get Current State | O(1) | In-memory reference |
| Replay to Point | O(n) | Must apply all reducers |
| Verify Integrity | O(n) | Check all hashes |
| Find Event by Hash | O(1) | Hash index |
| Branch Creation | O(n) | Must replay to branch point |

### Optimization Strategies

1. **Snapshots**: Save state at intervals, replay from nearest
2. **Indexing**: Secondary indices on event attributes
3. **Caching**: Memoize reducer results
4. **Lazy Replay**: Only materialize states on demand
5. **Incremental**: Cache deltas instead of full state

---

## Example: Counter

```javascript
System {
  initialState: { count: 0 }
  
  Reducers: {
    INCREMENT: (state, event) => 
      ({ count: state.count + event.payload.amount })
    
    DECREMENT: (state, event) => 
      ({ count: state.count - event.payload.amount })
  }
}

Timeline:
  
  t=0 State: { count: 0 }
  │
  ├─→ Event: { type: 'INCREMENT', payload: { amount: 5 } }
  │   Hash: 'abc123...'
  │
  t=1 State: { count: 5 }
  │
  ├─→ Event: { type: 'INCREMENT', payload: { amount: 3 } }
  │   Hash: 'def456...' (prevHash: 'abc123...')
  │
  t=2 State: { count: 8 }
  │
  └─→ Event: { type: 'DECREMENT', payload: { amount: 2 } }
      Hash: 'ghi789...' (prevHash: 'def456...')
  
  t=3 State: { count: 6 }

Proof at t=3:
  - Event chain: [INCREMENT(5), INCREMENT(3), DECREMENT(2)]
  - Deterministic replay: 0 → 5 → 8 → 6
  - State hash matches: ✓
  - Causality valid: ✓
  - Conclusion: State HAD to be 6
```

---

## Extensibility Points

### Where to Extend

1. **New Reducers**: Add domain logic
2. **Validation**: Register custom schemas
3. **Storage**: Implement persistence backend
4. **Networking**: Add event replication
5. **Compression**: Implement snapshot/delta encoding
6. **Monitoring**: Hook into append/replay
7. **Authorization**: Pre-validation checks
8. **Analytics**: Event stream analysis

### Extension Pattern

```javascript
// Example: Add persistence

class PersistentSystem extends DeterministicEventSystem {
  constructor(initialState, storageBackend) {
    super(initialState)
    this.storage = storageBackend
  }
  
  appendEvent(event) {
    const result = super.appendEvent(event)
    
    // Persist after successful append
    this.storage.save({
      event: event,
      stateHash: result.stateHash
    })
    
    return result
  }
}
```

---

## Security & Integrity

### Threat Model

| Threat | Mitigation |
|--------|-----------|
| Event tampering | Hash chain + verification |
| Out-of-order events | Causal chain check |
| Reducer injection | Type checking + registry |
| State mutation | Immutability enforcement |
| Fork attacks | Single log + causal ordering |

### Verification Checklist

- [ ] Event log integrity verified
- [ ] All hashes computed correctly
- [ ] Causal chain unbroken
- [ ] Reducers are pure
- [ ] No state mutations observed
- [ ] All events applied deterministically
- [ ] Replay produces identical state

---

## Conclusion

The system is elegant because it:

1. **Unifies concepts**: Events, state, proofs, causality
2. **Enforces correctness**: Immutability, purity, determinism
3. **Enables verification**: Replay, proof, integrity checks
4. **Scales elegantly**: Add snapshots, indexing as needed
5. **Remains simple**: Core is ~300 lines of code

The core insight:

> **State is not stored. State is computed. All computation is deterministic. All computation is verifiable.**

This is the foundation for:
- Fault tolerance
- Distributed consensus
- Temporal queries
- Causal reasoning
- Complete auditability
