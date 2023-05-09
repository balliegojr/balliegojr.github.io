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

# Note

<a href="https://github.com/balliegojr/state-machine" target="_blank">
Shutup and show me the code
</a>

The code examples in the post use async/await functions. The code in the github repository uses only sync functions, for benchmarking purposes.

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

```rust
#![feature(async_fn_in_trait)] 
// This implementation uses a nightly feature, but it is interchangeable with async_trait crate

/// A trait that represents the state machine
pub trait InternallyDrivenTransition {
    async fn execute(self) -> Result<Self, Box<dyn Error>>
    where
        Self: Sized;

    fn is_terminal_state(&self) -> bool;
}


/// An executor that drives that state machine
pub async fn internally_driven_executor<T: InternallyDrivenTransition>(
    initial_state: T,
) -> Result<(), Box<dyn Error>> {
    let mut current_state = initial_state;

    while !current_state.is_terminal_state() {
        current_state = current_state.execute().await?;
    }

    Ok(())
}

/// These are all the possible states for this state machine
pub enum FullStateMachine {
    DiscoverNodes(DiscoverNodes),
    ConnectNodes(ConnectNodes),
    Consensus(Consensus),
    Leader(Leader),
    Follower(Follower),
    Terminate,
}


impl InternallyDrivenTransition for FullStateMachine {
    async fn execute(self) -> Result<Self, Box<dyn Error>>
    where
        Self: Sized,
    {
        // Execution and transition are achieved in the same place
        match self {
            FullStateMachine::DiscoverNodes(discover_nodes) => {
                let nodes = discover_nodes.execute().await;
                Ok(FullStateMachine::ConnectNodes(ConnectNodes::new(nodes)))
            }
            FullStateMachine::ConnectNodes(connect_nodes) => {
                let connections = connect_nodes.execute().await;
                Ok(FullStateMachine::Consensus(Consensus::new(connections)))
            }
            FullStateMachine::Consensus(consensus) => {
                // Conditional state transition
                let (is_leader, connections) = consensus.execute().await;
                if is_leader {
                    Ok(FullStateMachine::Leader(Leader::new(connections)))
                } else {
                    Ok(FullStateMachine::Follower(Follower::new(connections)))
                }
            }
            FullStateMachine::Leader(leader) => {
                leader.execute().await;
                Ok(FullStateMachine::Terminate)
            }
            FullStateMachine::Follower(follower) => {
                follower.execute().await;
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
- The executor is very straight forward, it starts with a initial state, it calls the `execute` function and replace the current state until it reaches a terminal state. 
- The `execute` function consumes the current state and returns a new one, this makes internal state management simple and decoupled.
- Conditional state transition is easy to achieve, event with async code

There is one detail about this particular implementation, the execution and transition happen in the same function, this can be a problem, particularly if the state machine has a lot of states. 

Here we have a variation with different execute and transition functions, this one also works with external input for the transition.

```rust
pub trait ExternallyDrivenTransition {
    type EventType;

    async fn execute(&mut self, input: Self::EventType) -> Result<(), Box<dyn Error>>;
    fn is_terminal_state(&self) -> bool;
    fn transition(self) -> Self;
}

pub async fn externally_driven_executor<T: ExternallyDrivenTransition>(
    initial_state: T,
    events: Receiver<T::EventType>,
) -> Result<(), Box<dyn Error>> {
    let mut current_state = initial_state;

    while let Ok(input) = events.recv() {
        current_state.execute(input).await?;

        current_state = current_state.transition();
        if current_state.is_terminal_state() {
            break;
        }
    }

    Ok(())
}

pub enum ExternalEvent {}

impl ExternallyDrivenTransition for FullStateMachine {
    type EventType = ExternalEvent;

    async fn execute(&mut self, input: Self::EventType) -> Result<(), Box<dyn Error>> {
        match self {
            FullStateMachine::DiscoverNodes(state) => state.execute(input).await,
            FullStateMachine::ConnectNodes(state) => state.execute(input).await,
            FullStateMachine::Consensus(state) => state.execute(input).await,
            FullStateMachine::Leader(state) => state.execute(input).await,
            FullStateMachine::Follower(state) => state.execute(input).await,
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
Now we have smaller `execute` and `transition` functions, and both do only one thing. However, with this implementation, the `execute` function receives a `&mut self`, this means we may need to keep more state around, and maybe clone some state. Let's compare the `DiscoverNodes` for both implementations


```rust
// First implementation, with self consuming state
impl DiscoverNodes {
    pub async fn execute(self) -> Result<(Vec<IpAddr>), Box<dyn Error>> {
        Ok(crate::get_service_nodes().await)
    }
}

// Second implementation
impl DiscoverNodes {
    pub async fn execute(&mut self, _input: ExternalEvent) -> Result<(), Box<dyn Error>> {
        // We need to keep the state around until transition is called
        self.nodes = crate::get_service_nodes().await; 
        Ok(())
    }
}
```

## Downsides
With the enum implementation, the transition control is tied to the enum, if you want to implement a similar state machine, you need to implement everything again.

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

One way to address the downside from the enum implementation, is to implement a trait for the states and have an executor that works with `Box<dyn State>` states, the implementation will look like this

```
// We need to use async_trait here, because we receive a Box<Self> parameter
#[async_trait]
pub trait State {
    async fn execute(self: Box<Self>) -> Result<Option<Box<dyn State>>, Box<dyn Error>>;
}

pub async fn executor(initial_state: Box<dyn State>) -> Result<(), Box<dyn Error>> {
    let mut current_state = Some(initial_state);

    while let Some(state) = current_state {
        current_state = state.execute().await?;
    }

    Ok(())
}
```

We can already see some downsides for this implementation.

First, it introduces a performance penalty because every state is boxed, this may or may not be important for your use case, but it is something to have in mind.

Second, this implementation is even less re-usable than the last one, because it delegates the flow to inside the state, we can take a look in the `DiscoverNodes` implementation

```
pub struct DiscoverNodes {}
#[async_trait]
impl State for DiscoverNodes {
    async fn execute(self: Box<Self>) -> Result<Option<Box<dyn State>>, Box<dyn Error>> {
        let nodes = crate::get_service_nodes().await;
        Ok(Some(Box::new(ConnectNodes::new(nodes))))
    }
}
```

This design makes the code even harder to understand and to reuse, making it a very bad choice, considering what our goals are.  

However, there is another way we can implement this, by using one of Rust nicest features, blanket implementations!

# Composable traits [Blanket implementations]

In Rust, it is possible to implement a trait for every type in your code
```
impl<T> MyTrait for T {}
```

We can use this feature to implement our state machine in a very composable way
```
first_state.and_then(NextState::new).execute().await;
```

Lets see how we can do that
```
pub trait State {
    type Output;

    async fn execute(self) -> Result<Self::Output, Box<dyn Error>>;
}

pub trait StateComposer {}

// Implement StateComposer for every State in our crate
impl<T> StateComposer for T where T: State {}
```

Nice, we have our base structure in place, now we are going to add functions to be able to compose states together.

## AndThen

The `and_then` implementation will accept a closure that receives the output from the previous state and creates a new state

```
// We need a structure to hold our closure
pub struct AndThen<T, U, F> {
    previous: T,
    map_fn: F,
    _marker: PhantomData<U>,
}

pub trait StateComposer {
    // The StateComposer just need to return our struct, nothing more
    fn and_then<T, F>(self, map_fn: F) -> AndThen<Self, T, F>
    where
        Self: State + Sized,
        T: State,
        F: FnOnce(Self::Output) -> T,
    {
        AndThen {
            previous: self,
            map_fn,
            _marker: Default::default(),
        }
    }
}

// Last, we need to implement State for our new struct
// This will allow us to chain call and_then for all the states
impl<T, U, F> State for AndThen<T, U, F>
where
    T: State,
    U: State,
    F: FnOnce(T::Output) -> U,
{
    type Output = U::Output;

    async fn execute(self) -> Result<Self::Output, Box<dyn Error>>
    where
        Self: Sized,
    {
        let previous_output = self.previous.execute().await?;
        let next_task = (self.map_fn)(previous_output);
        next_task.execute().await
    }
}
```

With this in place, we can now use our States in a very composable way
```
DiscoverNodes
    .and_then(ConnectNodes::new)
    .and_then(Consensus::new)
    .and_then(LeaderOrFollower::new)
    .execute()
    .await
```

And if we want a different flow using the same states? Easy!!!
```
DiscoverNodes
    .and_then(ConnectNodes::new)
    .and_then(Leader::new)
    .execute()
    .await
```

Other composable functions can be added to your composer, some examples are
- `and` that ignores the output from the previous state
- `or` or `and_default` that will continue execution even if the previous steps failed
- `loop` that will execute the same state multiple times

The major benefit of this approach, your workflow is validated at compile time and it is very explicit. However, this approach have one big downside compared to the previous ones, it doesn't allow conditional states, if you need conditional states, there are two possible solutions.

One is to change the `execute` function to receive `self: Box<self>`, another way is to have one step that is responsible for chosing the next state, like so

```
pub struct LeaderOrFollower {
    is_leader: bool,
    connections: Vec<NodeConnection>,
}

impl State for LeaderOrFollower {
    type Output = ();

    async fn execute(self) -> Result<Self::Output, Box<dyn Error>> {
        if self.is_leader {
            Leader::new(self.connections).execute().await
        } else {
            Follower::new(self.connections).execute().await
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

Time for a very opinionated conclusion....   

My favorite approach is, of course, the composable trait implementation - I even had to write a blog post about it!!! - I think it makes for very clean and reusable code with minimal overhead. 

The enum implementation - with all the other variations you will find around - is simple and easy to put together, however it is not good if your intent is to reuse or make many similar processes with few different steps.

The dyn trait implementation was my first attempt to implement a state machine by using traits instead of enums. The only benefit it provides over the enum implementation is decoupling the States from each other, so you don't have a [potentially huge] match statement with all the states and transitions. But overall, I think I wouldn't use that implementation at all.

