# alan
State machine

## How ? State Machine, Pushdown Automata, Turing Machine
- State Machine
  - any time can access only current state and state machine (global) context
  - therefore limited memory
  - cannot affect execution of other states using state context
  - only next state can be affected using override context operation
- Pushdown Automata
  - introduces pop operation
  - push multiple symbols on tape
  - each symbol update only one state context and state machine context
- Turing
  - at any time access to all state context
  - change any and push any number
  - one state can affect later execution of other states

## Dev updates
### 0.1.0
- ~~Dispatcher~~
- ~~StateMachineDef~~
- ~~StateMachineExecutor~~
- ~~StateActionExecution~~
- ~~ExecutionStage~~
- ~~Exception Handling~~
- ~~Sync/Async StateTransition handling~~
- ~~Success and Failure stage handling~~
- ~~In memory serde persistance (primarily useful for testing)~~
- ~~Updating and/or resetting current state and state machine context~~
- ~~InMemory Lock~~
- ~~Markov~~
- ~~Test run (in memory)~~
- ~~Merge to develop~~
- ~~change the project name to alan~~
- Code refactoring (class designs)
  - ~~Interfaces~~
  - ~~Introduce Tape based persistance~~
  - ~~Fix Fluent State def~~
  - ~~kryo default serializer~~
  - ~~StateMachienDef arrange functions~~
  - ~~Contextless state~~
  - ~~Store event~~
  - ~~failure with error~~
  - ~~AlanException~~
  - ~~Timestamp~~
  - Custom thread factory
  - Logging
  - TapeLogFactory
- Persistance with Cassandra
- Lock using Apache Zookeeper
- Test run (db store)
- Revisit executor
  - Retry/Blocking queue
- Comments + License + README documentation
- Configurator and Factories support using typesafe-config
- Benchmark and Optimizations
- Test cases
- Merge to master

## Features
### 0.1.0
- State machine DSL
- Cassandra persistance layer
- Parallel Stateless Execution

### 0.1.1
- Fixes

### 0.2.0
- Alan 2.0
  - Pushdown automata
  - Turing machine
  - Reply to event from action
- Configurable Queues
- Windowed order gaurantees on machine
- Cached Persistance
- Zookeeper for other metadata storage (design stage)
- Markov cluster
  - Consistent Routing of events client
  - Routing event to machine already holding the lock
- Cache aware transfer of events
- MySQL persistance
- Connectors

### 0.3.0
- REST API (jersey)


## Design Notes

StateMachine
- Provide DSL to define state transitions

Dispatcher
  - register(executor)
    - add state machine as a subscriber for the related events
    - executor.begin() - start the executor
  - dispatch(event)
    - get state machines subscribed for the event
    - check if the state machine is receiving events or is in error state
      - enqueue the event on their respective executors
    - return with success
    - or Error containing information for each failed state machines

ExecutionStage
  

StateMachineExecutor (class)
  - An executor manages one or more instances (nI) of a state machine execution
    - note, this doesnt mean that there are multiple copy of state machine instance
    - just that, multiple events are executed at the same time
    - Use ParallelStateMachineExecutor(nI)
  - It maintains a LinkedBlockingQueue for events (1 producers -> nI consumers)
  - start()
    - create as many StateExectionAction
    - submit to the executor service
  - markAsIdle()
  - markAsScheduled()
  - markAsSuspended()
  - receive(event)
    - get stateMachineDef.getExecutionId(event)
    - enqueues the event+executionId in the blocking queue, and return immediately
      - if Idle
        - start()
  - StateExecutionAction extends RecursiveActions (no result bearing task)
    - dequeue an event from the queue
      - block until new events arrive or timeout
      - retry with exponential backoff
      - on finish set stateMachineExecutor status to Idle
    - if ! acquireLock(execution id), also retrieve the current execution stage step
      - enqueue(event) with retry++
      - new StateExecutionAction().fork()
    - Future<> = get execution stage from the store (if no step increment .. cached execution stage)
    - create a fork join task with the target action to run
    - run the action
    - create new execution stage
    - insert execution stage in the store
    - unlock the instance id
    - recursively call itself
    - on exception
      - if retries available
        - increment retry counter and enqueue the message again
  - Has internal statuses like Idle, Scheduled, Processing, Suspended, Terminated.
  - A ForkJoinTask that gets a State object and the context and runs it by calling the action

StateMachineExecutorFactory
  - create(stateMachineDef, config)

StateMachineExecutorFactoryProvider
  - getFactory(config)

Markov
  - Manages all the dependent components like store, dispatcher, etc
  - add(stateMachineDef, executorConfig)
    - Create Executors for the statemachine based on config
      - factoryProvider.getFactory(config).create(stateMachineDef, config)
    - dispatcher.register(executor)