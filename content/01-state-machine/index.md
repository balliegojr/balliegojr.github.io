+++
title = "Taking advantage of state machine concepts to organize code"
description =  "In this post we are going to explore the concept of a state machine to help organize our code. The goal is to explore different code designs, rather than build a full blown state machine."
date = 2023-05-05
draft = true
slug = "state-machine"
tags = ["state machine", "enums", "traits"]
+++

In this post we are going to explore the concept of a state machine to help organize our code. The goal is to explore different code designs, rather than build a full blown state machine.

<!-- more -->
___

# Code

The code seen in the snippets is available <a href="https://github.com/balliegojr/state-machine" target="_blank">here</a>, as well as a benchmark.

___


# What is a state machine?

[Wikipedia](https://en.wikipedia.org/wiki/Finite-state_machine) has a very nice article on the subject, but the definition is actually very short.
> It is an abstract machine that can be in exactly one of a finite number of states at any given time. 
The FSM can change from one state to another in response to some inputs; the change from one state to another is called a transition

Think about it as a sequence of individual steps, where only one step can be executed at a time. Very simple.

I think this abstraction is particularly useful for synchronization protocols. Imagine you need to synchronize the action of multiple nodes.
Whenever a node enters a state, it only cares about the data related to that particular state. By advancing all the nodes to the next state at the same time, 
the synchronization becomes simpler, it is easy to identify problems by just looking if a node is on a different state.

By using a state machine abstraction, your code can be simpler and easier to understand, which will lessen the possibility of bugs.

# Implementation requirements

The goal is to write code that is easy to read, write, and reuse.

We are going to implement a pseudo consensus process. The steps are:

1. Discover all nodes in the network
2. Connect to all nodes
3. Elect a leader
4. Start a sync process
    1. Followers will wait for events
    2. The Leader will only send events

We are going to try a few different implementations, first we will use enums and then traits, and of course, we are going to benchmark them.


# Enum implementation

Data carrying enums are perfect for implementing a state machine, because they impose the **one state at a time** rule by design

There are a few ways that we can implement this when using enums. One possible way is by using self consuming states.

We need a trait and an executor function, since we want it to be generic.
```rust
/// A trait that represents the state machine
/// This will be implemented for every state machine enum
pub trait StateMachine {
    fn execute(self) -> Result<Self, Box<dyn Error>>
    where
        Self: Sized;

    fn is_terminal_state(&self) -> bool;
}


/// An executor that drives that state machine
pub fn executor<T: StateMachine>(
    initial_state: T,
) -> Result<(), Box<dyn Error>> {
    let mut current_state = initial_state;

    while !current_state.is_terminal_state() {
        current_state = current_state.execute()?;
    }

    Ok(())
}
```

The trait has two functions, `execute` will be called to run that state to completion, `is_terminal_state` defines which states will end the state machine execution.  

```rust
/// These are all the possible states for this state machine
pub enum FullStateMachine {
    DiscoverNodes(DiscoverNodes),
    ConnectNodes(ConnectNodes),
    Consensus(Consensus),
    Leader(Leader),
    Follower(Follower),
    Terminate,
}


impl StateMachine for FullStateMachine {
    fn execute(self) -> Result<Self, Box<dyn Error>>
    where
        Self: Sized,
    {
        // Execution and transition are achieved in the same place
        match self {
            FullStateMachine::DiscoverNodes(discover_nodes) => {
                let nodes = discover_nodes.execute();
                Ok(FullStateMachine::ConnectNodes(ConnectNodes::new(nodes)))
            }
            FullStateMachine::ConnectNodes(connect_nodes) => {
                let connections = connect_nodes.execute();
                Ok(FullStateMachine::Consensus(Consensus::new(connections)))
            }
            FullStateMachine::Consensus(consensus) => {
                // Conditional state transition
                let (is_leader, connections) = consensus.execute();
                if is_leader {
                    Ok(FullStateMachine::Leader(Leader::new(connections)))
                } else {
                    Ok(FullStateMachine::Follower(Follower::new(connections)))
                }
            }
            FullStateMachine::Leader(leader) => {
                leader.execute();
                Ok(FullStateMachine::Terminate)
            }
            FullStateMachine::Follower(follower) => {
                follower.execute();
                Ok(FullStateMachine::Terminate)
            }
            FullStateMachine::Terminate => {
                // A proper error should be used here, to avoid panics.
                unreachable!()
            }
        }
    }

    fn is_terminal_state(&self) -> bool {
        matches!(self, Self::Terminate)
    }
}
```

What I like about this implementation:
- The executor is very straight forward, it starts with a initial state and then calls the `execute` function and replace the current state until it reaches a terminal state. 
- The `execute` function consumes the current state and returns a new one, this makes internal state management simple and decoupled.
- Conditional state transition is easy to achieve, event with async code

There is one detail about this particular implementation that I don't like, the execution and transition happen in the same function, this can be a problem, particularly if the state machine has a lot of states. 

Lets fix that


```rust
pub trait StateMachine {
    fn execute(&mut self) -> Result<(), Box<dyn Error>>;
    fn is_terminal_state(&self) -> bool;
    fn transition(self) -> Self;
}

pub fn executor<T: StateMachine>(
    initial_state: T,
) -> Result<(), Box<dyn Error>> {
    let mut current_state = initial_state;

    while !current_state.is_terminal_state() {
        current_state.execute()?;
        current_state = current_state.transition();
    }

    Ok(())
}

```

Now we have `execute` and `transition` as two separate functions, lets see how the implementation of the state machine looks like.

```rust
impl StateMachine for FullStateMachine {
     fn execute(&mut self) -> Result<(), Box<dyn Error>> {
        match self {
            FullStateMachine::DiscoverNodes(state) => state.execute(),
            FullStateMachine::ConnectNodes(state) => state.execute(),
            FullStateMachine::Consensus(state) => state.execute(),
            FullStateMachine::Leader(state) => state.execute(),
            FullStateMachine::Follower(state) => state.execute(),
            FullStateMachine::Terminate => unreachable!(),
        }
    }

    fn is_terminal_state(&self) -> bool {
        matches!(self, Self::Terminate)
    }

    fn transition(self) -> Self {
        match self {
            FullStateMachine::DiscoverNodes(state) => {
                FullStateMachine::ConnectNodes(ConnectNodes::new(state.nodes))
            }
            FullStateMachine::ConnectNodes(state) => {
                FullStateMachine::Consensus(Consensus::new(state.connections))
            }
            FullStateMachine::Consensus(state) => {
                if state.is_leader {
                    FullStateMachine::Leader(Leader::new(state.connections))
                } else {
                    FullStateMachine::Follower(Follower::new(state.connections))
                }
            }
            FullStateMachine::Leader(_) => FullStateMachine::Terminate,
            FullStateMachine::Follower(_) => FullStateMachine::Terminate,
            FullStateMachine::Terminate => unreachable!(),
        }
    }
}
```

With the `execute` and `transition` functions, the code is simpler, easier to understand.  

There is a downside however, since `execute` takes a reference to self, each state needs to hold any intermediate state until `transition` is called. If we look at the `DiscoverNodes` implementation for both versions, we can see the difference.  

```rust
// First implementation, with self consuming state
struct DiscoverNodes;

impl DiscoverNodes {
    pub fn execute(self) -> Result<(Vec<IpAddr>), Box<dyn Error>> {
        Ok(crate::get_service_nodes())
    }
}

// Second implementation
struct DiscoverNodes { 
    pub nodes: Vec<IpAddr>
}

impl DiscoverNodes {
    pub fn execute(&mut self) -> Result<(), Box<dyn Error>> {
        // We need to keep the state around until transition is called
        self.nodes = crate::get_service_nodes(); 
        Ok(())
    }
}
```

Both implementations we have seen so far expects that each `execute` call drives the state to completion, we can easily modify the code to react to external events. 
Lets see how to do it.  

```rust
pub trait ExternallyDrivenTransition {
    type EventType;

    fn execute(&mut self, input: Self::EventType) -> Result<(), Box<dyn Error>>;

    fn is_terminal_state(&self) -> bool;
    fn transition(self) -> Self;
}

pub fn externally_driven_executor<T: ExternallyDrivenTransition>(
    initial_state: T,
    events: Receiver<T::EventType>,
) -> Result<(), Box<dyn Error>> {
    let mut current_state = initial_state;

    while let Ok(input) = events.recv() {
        current_state.execute(input)?;

        current_state = current_state.transition();
        if current_state.is_terminal_state() {
            break;
        }
    }

    Ok(())
}

pub enum ExternalEvent {
...
}

impl ExternallyDrivenTransition for FullStateMachine {
    type EventType = ExternalEvent;

    fn execute(&mut self, input: Self::EventType) -> Result<(), Box<dyn Error>> {
        match self {
            FullStateMachine::DiscoverNodes(state) => state.execute(input),
            FullStateMachine::ConnectNodes(state) => state.execute(input),
            FullStateMachine::Consensus(state) => state.execute(input),
            FullStateMachine::Leader(state) => state.execute(input),
            FullStateMachine::Follower(state) => state.execute(input),
            FullStateMachine::Terminate => unreachable!(),
        }
    }
}
```


## Downsides of enums 
With the enum implementation, the transition control is tied to the enum, if you want to implement a similar state machine, you need to implement everything again. 
This may not be a problem if you only have one process, or process that do not overlap at all, otherwise there will be some boilerplate code to be written.

```rust
pub enum FullStateMachine {
    DiscoverNodes(DiscoverNodes),
    ConnectNodes(ConnectNodes),
    Consensus(Consensus),
    Leader(Leader),
    Follower(Follower),
    Terminate,
}

impl ExternallyDrivenTransition for FullStateMachine { 
... 
}

pub enum ConsensusLessStateMachine {
    DiscoverNodes(DiscoverNodes),
    ConnectNodes(ConnectNodes),
    Leader(Leader),
    Follower(Follower),
    Terminate,
}

impl ExternallyDrivenTransition for ConsensusLessStateMachine { 
... 
}
```

# Traits implementation

One way to address the downside from the enum implementation, is to implement a trait for the states and have an executor that works with `Box<dyn State>` states, the implementation will look like this. 

```rust
pub trait State {
    fn execute(self: Box<Self>) -> Result<Option<Box<dyn State>>, Box<dyn Error>>;
}

pub fn executor(initial_state: Box<dyn State>) -> Result<(), Box<dyn Error>> {
    let mut current_state = Some(initial_state);

    while let Some(state) = current_state {
        current_state = state.execute()?;
    }

    Ok(())
}
```
The implementation is very similar to the enum, except the termination control is done by return an `Option`.

--- 

This version has some clear downsides over the enum implementation.  

First, it introduces a performance penalty by boxing the state, this may or may not be important for your use case, but it is something to have in mind.

Second, it is even less re-usable than the enum version, because it delegates the flow to inside the state. Lets see how the `DiscoverNodes` would look like with this version

```rust
pub struct DiscoverNodes {}
impl State for DiscoverNodes {
    fn execute(self: Box<Self>) -> Result<Option<Box<dyn State>>, Box<dyn Error>> {
        let nodes = crate::get_service_nodes();

        // Here we define what is the next state
        Ok(Some(Box::new(ConnectNodes::new(nodes))))
    }
}
```

This design makes the code even harder to understand and to reuse. If you want to have a different state after the `DiscoverNodes`, you need to introduce an internal condition and way to drive that condition, and this becomes even more cumbersome as you go further in the state machine flow, making this a very bad choice for a state machine design, considering what our goals are.  

However, there is another way we can implement this, by using one of Rust nicest features, blanket implementations!

# Composable traits [Blanket implementations]

In Rust, it is possible to implement a trait for every type in your code
```rust
impl<T> MyTrait for T {}
```

We can use this feature to implement our state machine in a very composable way
```rust
first_state::new()
    .and_then(SecondState::new)
    .and_then(ThirdState::new)
    .execute();
```

Lets see how we can do that
```rust
pub trait State {
    type Output;

    fn execute(self) -> Result<Self::Output, Box<dyn Error>>;
}

pub trait StateComposer {}

// Implement StateComposer for every State in our crate
impl<T> StateComposer for T where T: State {}
```

Nice, we have our base structure in place, now we are going to add functions to be able to compose states together.

## AndThen

The `and_then` implementation will accept a closure that receives the output from the previous state and creates a new state. 

First we need a struct to hold the State transition
```rust
pub struct AndThen<T, U, F> {
    previous: T,
    map_fn: F,
    _marker: PhantomData<U>,
}
```

Without constraints, the struct itself is nothing special.
- `previous: T` holds "everything before the ." in our chain call.  
- `map_fn: F` holds our closure that constructs the next step.
- `_marker: PhantomData<U>` holds the return type of our state

Fun fact, without the `_marker` field, if you try to compile the code with the nightly async feature, it will crash the compiler.

___

Now we need to add a new function to the `StateComposer` trait
```rust
pub trait StateComposer {
    // The StateComposer just need to return our struct, nothing more
    fn and_then<U, F>(self, map_fn: F) -> AndThen<Self, U, F>
    where
        Self: State + Sized,
        U: State,
        F: FnOnce(Self::Output) -> U,
    {
        AndThen {
            previous: self,
            map_fn,
            _marker: Default::default(),
        }
    }
}
```

When we call this function, it will build a new State type, and for every new call, a new type will be built, very similar to what we have with Iterators, for example:
```rust
let state = One::new().and_then(|out: String| Two::new(out));
// state: AndThen<One, Two, fn(String) -> Two>

let state = state.and_then(|out: u64| Third::new(out));
// state: AndThen<AndThen<One, Two, fn(String) -> Two>, Third, fn(u64) -> Third>
```

Last but not least, we need to implement the `State` trait for our new `AndThen` struct

```rust
impl<T, U, F> State for AndThen<T, U, F>
where
    T: State,
    U: State,
    F: FnOnce(T::Output) -> U,
{
    type Output = U::Output;

    fn execute(self) -> Result<Self::Output, Box<dyn Error>>
    where
        Self: Sized,
    {
        // Execute the previous state, or "everything before the ."
        let previous_output = self.previous.execute()?;

        // call the map function to create the next task
        let next_task = (self.map_fn)(previous_output);

        // execute the next task
        next_task.execute()
    }
}
```

Neat! Now that we have everything in place, we can use our States in a very composable way.
```rust
DiscoverNodes
    .and_then(ConnectNodes::new)
    .and_then(Consensus::new)
    .and_then(LeaderOrFollower::new)
    .execute()
```

And if we want a different flow using the same states? Easy!!!
```rust
DiscoverNodes
    .and_then(ConnectNodes::new)
    .and_then(Leader::new)
    .execute()
```

The `StateComposer` trait can be extended with how many functions you want, some examples can be:
- `and` that ignores the output from the previous state
- `or` or `and_default` that will continue execution even if the previous steps have failed
- `loop` that will execute the same state multiple times

```rust
DiscoverNodes
    .and_then(ConnectNodes::new)
    .and_then(Consensus::new)
    .and_then(LeaderOrFollower::new)
    .and_then_default_to(|| Daemon::new().loop())
    .execute()
```

The major benefit of this approach, your workflow is validated at compile time and it is very explicit.  

However, this approach have one big downside compared to the previous ones, it doesn't allow conditional states, if you need conditional states, you need to implement an additional State.

```rust
pub struct LeaderOrFollower {
    is_leader: bool,
    connections: Vec<NodeConnection>,
}

impl State for LeaderOrFollower {
    type Output = ();

    fn execute(self) -> Result<Self::Output, Box<dyn Error>> {
        if self.is_leader {
            Leader::new(self.connections).execute()
        } else {
            Follower::new(self.connections).execute()
        }
    }
}
```

# Benchmarks

There is a benchmark in the github repo to compare the overhead of each of these implementations, it is no surprise that `Box<dyn State>` has the worst performance of all the implementations we have here.

{{ image(src="./violin.svg", style="background-color: white") }}

I believe the composing approach has better performance due to the lack of an executor loop function. But the overhead seems to be minimal.

I don't really trust "it ran on my machine" benchmarks, so I highly encourage you to download the code and try it yourself.


# Conclusion

Time for a very opinionated conclusion...   

My favorite approach is, of course, the composable trait implementation - I even had to write a blog post about it!!! - I think it makes for very clean and reusable code with minimal overhead. 

The enum implementation - with all the other variations you will find around - is simple and easy to put together, however it is not good if your intent is to reuse or make many similar processes with few different steps.

The dyn trait implementation was my first attempt to implement a state machine by using traits instead of enums. The only benefit it provides over the enum implementation is decoupling the States from each other, so you don't have a [potentially huge] match statement with all the states and transitions. But overall, I think I wouldn't use that implementation at all.

