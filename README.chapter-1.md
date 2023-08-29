# rust-ractor-tutorial-1
Part of a tutorial that aims to elaborate Actor-based programming using 'ractor' ( Rust library ) 

This is a turotial on `ractor`, an Actor create that helps building Actor-based application using **#rustlang**. 

This is the first of the tuorial series. The second is [here](https://github.com/nsengupta/rust-ractor-tutorial-2).

All the code is in './src'.

Code structure:

![](./accompanying-contents/code-structre.png)


## Tutorial


### Prelude

The idea of using Actors as the underpinning of a non-trivial application interests me a lot. Some years back, along with learning [Scala](https://www.scala-lang.org/) programming language, I also began to dabble with [Akka](https://akka.io/), the Actor-based library from [Lightbend](https://www.lightbend.com/). Thereafter, I had the opportunity to build production-level, applications using Scala (and Java) and Akka, a couple of times. Happily, they are still in use. :smiley:

Since then, I have remained a very keen enthusiast to study and use the Actor-based approach while designing asynchronous yet performance applications, whenever possible. 

When I began to learn Rust - specifically its thread-model and its approach to asynchronous programming - I was curious to find out if Rustaceans had explored the world of Actors already and how. A search brought forth an endless list of crates - here is that famous compilation on [Reddit](https://www.reddit.com/r/rust/comments/n2cmvd/there_are_a_lot_of_actor_framework_projects_on/) - and I was at my wit's end! The sheer number was unsettling.

All of a sudden, my eyes fell on a post on `ractor`, which said to have been inspired by `erlang` and its `gen_server`. I didn't know  _erlang_ , the language but armed with the little knowledge collected from reading about its *processes* and *messages* - along with whatever I had learnt from Akka-days, I decided to find more.


When I searched for any conversation on `ractor` on Stackoverflow, it reported **zero** entries!!

I decided to go through their github repo and the companion website and try to understand the basic principles on which `ractor` was based. Whatever I understood, could be inputs to a tutorial, I thought. This blog series is a result of that attempt.

### Actors, a tl;dr

This tutorial is not meant to expound on `Actors`. Numerous resources exist, including research papers, books, articles, deep and wide discussions, example and prod-ready implementations, Wikipedia entries, QnA etc. I am going to skip all of that (leaving to you, where to start and how far to go) and simply state, that :

* An Actor is a typical Object (from the OO world) that doesn't expose public methods for the world to invoke. Instead, it advertises messages that it can *respond* to.
* These messages are asynchronous; also called fire-and-forget! The sender Actor is not sure if the message is received by the target Actor and when. Because there is no public method, there is no return value. The target Actor, may send a message to the sender Actor, if that is the protocol, and that is the only way to have a result in return.
* An Actor deals with messages, one at at time. No two messages are handled at the same time. Effectively, every Actor processes messages, in a single-threaded manner.

### Prepare to use `ractor`

My *Cargo.toml* lists the following dependencies:

```toml
ractor = "0.8.4"
async-trait = "0.1"
tokio = { version = "1", features = ["rt", "time", "sync", "macros", "rt-multi-thread", "signal"] }
```


### How to make an Actor

To begin with, we need an object of an user-defined type. This object, otherwise, would have one or more public(ly callable) methods. In Rust, we use a `struct` for this purpose.  

Because we want this object behave as an Actor, we *Actorify* it by implemnting an `ractor::Actor` trait for struct. For example:

```rust
use ractor::{Actor, ActorRef};
struct SomethingThatCanBehaveAsAnActor ;

impl Actor for SomethingThatCanBehaveAsAnActor {
    type Msg = Message;
    type State = u8;
    type Arguments = ();

    // .....
}
```

There are three *type aliase*s here!

* `Msg` denotes the type of message(s) that this Actor understands and can take action on. Messages of any other tyoe, if sent to this actor, is simply ignored.
* `State` denotes any kind of data that this actor wants to store between processing messages. 
* `Arguments` denotes values that should passed to the actor while it is being initialized.

In order to understand how exactly are these to be used, let's expand the sample Pingpong Actor that is shared at the `ractor` github's README.

### Conversations between two Actors

We intend to devise two Actors, who exchange messages between themselves. One Pings (sends a Ping message to the other) and then in response, another Pongs (sends a Pone message to the Ping-sender). The conversation is comprised of **only** these two messages. Neither of the actors understands any other message. 

### Overview of structure of an Actor

Any Object can decide to behave like an Actor. Thereafter, the object wears the capabilty of reacting to a message received.Why? Because, behaviorally, an Actor is equipped with the capabillity to respond to a message (one or more) that it is aware of.

Almost always, we refer to the object itself as an **Actor**, interachangebly. However, it is important to distinguish between a regular object - having its own data and processing logic - and its 'wore-on' Actor-like behavior.

Once it comes into being, such an *Actorified* object waits patiently till it receives such a message. If the message is unknown, the Actor is free not to take any action.

Importantly, the Actor handles these messages, **one at a time**!  Put it simply, even if a bunch of messages jostle its door, the Actor will process them **one at a time**. Obviously, while it is handling one message, other messages wait patiently.

So, there are three main pieces that go together to bring an Actor to life:
* A user-defined type (viz. `struct` ) that wants to behave like an Actor, in addition to its own behaviour, if any.
* A trait named `Actor` (of type `ractor::actor::Actor` ) which needs specific implementation for the user-defined type.
* A bunch of (1 or more) Messages that this Actor understands and can take action on.

### Build a Actor that can handle a _PING_ 

We want to build an Actor that can
1. understand a PING message, meaning PING is a part of its vocabulary. 
2. respond to the sender with a PONG message, meaning PONG is a part of its vocabulary too.

 We bring in, an user-defined entity named `CanRecognizePing` , a zero-element `struct`.

```rust
pub struct CanRecognizePing;
```

 In order to behave as an Actor, this needs to implement the _Actor_ trait: `ractor::actor`.
 
 ```rust
 impl Actor for CanRecognizePing {
 // ...
 }
 ```
It claims to understand a PING message. Well, the following is that `Ping` message :smiley: :

```rust
pub enum Message {
    Ping,
}
```
We have the Actor and a Message. How does the Actor deal with the message?

The `Actor` trait stiputales that `CanRecognizePing` must provide the logic to handle any message - *Ping* in this case - in the body of a method named `handle`:

```rust
async fn handle(
        &self,
        myself: ActorRef<Self::Msg>,
        message: Self::Msg,
        state: &mut Self::State,
    ) -> Result<(), ActorProcessingErr> {

        match message {
                Message::Ping => println!("Received a ping message"),     
                _ => println!("Unknown message {:?} received!", message),
        }

        Ok(())
    }
```
Let us understand what the method expects and does.

`&self` denotes the implementation itself - the CanRecognizePing. This is usual first parameter, in the signature of a *struct*'s method in Rust.

`myself` denotes the Actor. At this point, it is sufficient for us to take this as the *self* of the Actor itself. The *struct* has an *actor* avatar, as it were, and this refers to that inner avatar.

`message` denotes the type of messages that the Actor claims to understand. When it receives one, the Actor can handle that.

`state` denotes the Actor's internal data structure and contents. We will skip it for the time being.


Interestingly, the method can either return no value (absence of a value) or an `ActorProcessingError`. The significance of this will be clearer later.

Inside the method, we are checking if the message that has been received, is some message that the Actor claims to understand. The logic of `CanRecognizePing` states that it indeed understands a PING message. On receipt of it, the Actor simply prints an acknowledgement and returns. For any other message, it prints a complaint! That is all about our `CanRecognizePing`.

In order to see it in action, we have to ( a ) bring it into life and then ( b ) send a PING message to it. Let's do that.

We create an Actor, using a special method called `Actor::spawn()`.

```rust
let (ping_actor_ref, actor_handle) = Actor::spawn(None, CanRecognizePing, ())
        .await
        .expect("Failed to start CanRecognizePing");
```

The key parameter to the `spawn` funtion is `CanRecognizePing`. We are asking **ractor**'s runtime to create an Actor which is an incarnation of the `struct` named `CanRecognizePing`. The resultant pair, has two parts. The first is what we call a reference to the Actor of type `ActorRef`. For all interactions with the actor, this reference is to be used. We will ignore the second part, till later.

Then, we send a PING message to the actor we have

```rust
    ping_actor_ref.send_message(PingPongMessage::Ping).unwrap();
```

This is the complete `main` function:

```rust
#[tokio::main]
async fn main() {
    let (ping_actor_ref, actor_handle) = Actor::spawn(None, CanRecognizePing, ()).await.expect("Failed to start actor");

    ping_actor_ref.send_message(PingPongMessage::Ping).unwrap();

    actor_handle.await.expect("Actor failed to exit cleanly");
}
```

Ignore the last line, for the time being.

On being run, the program's output is:

```bash
CanRecognizePing has Received a ping message
^C
```

After printing that message, the program does nothing else. We stop it forcibly.

### Conversation between two actors

Sending a ping message, to an actor only once, like we have done in the `main`  function above, is of no use really. In order to receive as well as send messages, an Actor has to do more. In that case, two Actors can exchange messages and can have a meaningful conversation.

We have a `CanRecognizePing` which can handle a PING message. Let's create its counterpart: `CanRecognizePong` which can handle a PONG message.

Adding a PONG message to the vocabulary of the actors is easy.

```rust
#[derive(Debug, Clone)]
pub enum PingPongMessage {
    Ping,
    Pong,
}
```

A `CanRecognizePong` is exactly like a `CanRecognizePing`; the name differs, that is all.

```rust
pub struct CanRecognizePong;
```

The `handle` method is quite similar too.

```rust
 async fn handle(
        &self,
        myself: ActorRef<Self::Msg>,
        message: Self::Msg,
        state: &mut Self::State,
    ) -> Result<(), ActorProcessingErr> {

        match message { 
            PingPongMessage::Pong => println!("CanRecognizePong has received a pong message"),
            _ => println!("Unknown message {:?} received!", message),
        }
        Ok(())
    }
```

Let's create a `CanRecognizePong` and send a message to it. Thi is the modified `main` method:

```rust
#[tokio::main]
async fn main() {
    let (ping_actor_ref, actor_handle_ping) = Actor::spawn(None, CanRecognizePing, ()).await.expect("Failed to start actor");

    let (pong_actor_ref, actor_handle_pong) = Actor::spawn(None, CanRecognizePong, ()).await.expect("Failed to start actor");

    ping_actor_ref.send_message(PingPongMessage::Ping).unwrap();
    pong_actor_ref.send_message(PingPongMessage::Pong).unwrap();

    actor_handle_ping.await.expect("Actor failed to exit cleanly");
    actor_handle_pong.await.expect("Actor failed to exit cleanly");
}
```

Nothing unexpected appears in the output:

```bash
    CanRecognizePing has Received a ping message
    CanRecognizePong has received a pong message
^C
```

On receipt of a message, each of them responds as expected. Yet, they are not *conversing* between themselves. 

In order to converse between themselves, an Actor has to know who has sent in a message, so that the former can respond to the latter. One way to send a message to Actor, is by using its `ActorRef`. If a message carries the `ActorRef` of the sender, it is simple for the receiver to send a message in return. To facilitate this, we need to modify the messages:

```rust
pub enum PingPongMessage {
    Ping(ActorRef<PingPongMessage>),
    Pong(ActorRef<PingPongMessage>),
}
```

Then, we need to modify the corresponding _handler_ s, too:

```rust
// CanRecognizePing
async fn handle(
        &self,
        myself: ActorRef<Self::Msg>,
        message: Self::Msg,
        state: &mut Self::State,
    ) -> Result<(), ActorProcessingErr> {

        match message {         
          PingPongMessage::Ping(sender) => {
            println!("CanRecognizePing has Received a ping message");
            sender.send_message(PingPongMessage::Pong(myself)).unwrap();
          },
          _ => println!("Unknown message {:?} received!", message),
        }
        Ok(())
    }

// CanRecognizePong
 async fn handle(
        &self,
        myself: ActorRef<Self::Msg>,
        message: Self::Msg,
        state: &mut Self::State,
    ) -> Result<(), ActorProcessingErr> {

        match message {
            PingPongMessage::Pong(sender) => {
                println!("CanRecognizePong has received a pong message");
                sender.send_message(PingPongMessage::Ping(myself)).unwrap();
            },
            _ => println!("Unknown message {:?} received!", message),
        }
        Ok(())
    }
```

Finally, the `main` is modified to begin the conversation:

```rust
#[tokio::main]
async fn main() {
    let (ping_actor_ref, actor_handle_ping) = 
            Actor::spawn(
                    None, 
                    CanRecognizePing, 
                    ()
                ).await.expect("Failed to start actor");

    let (pong_actor_ref, actor_handle_pong) = 
            Actor::spawn(
                    None, 
                    CanRecognizePong, ()
                ).await.expect("Failed to start actor");

    ping_actor_ref.send_message(PingPongMessage::Ping(pong_actor_ref.clone())).unwrap();
    pong_actor_ref.send_message(PingPongMessage::Pong(ping_actor_ref.clone())).unwrap();
}
```

The output is an infinite series of messages:

```bash
CanRecognizePing has Received a ping message
CanRecognizePong has received a pong message
CanRecognizePong has received a pong message
CanRecognizePing has Received a ping message
CanRecognizePing has Received a ping message
CanRecognizePong has received a pong message
# .... continues like this
```

What causes this output? To explain that, let's take a closer look at the `handler`s.

The `CanRecognizePing` receives a Ping message. It uses the `ActorRef` of the sender ( the `CanRecognizePong` ) to send a Pong message to the sender. The `CanRecognizePong` receives a Pong message. It uses the `ActorRef` of the sender ( the `CanRecognizePing` ) to send a Ping message to the sender. The `CanRecognizePing` responds and then, the `CanRecognizePong` responds too. They continue doing this, *ad infinitum*.

Can we restrict them, to a maximum number of mutual responses? 

If we can keep a count of the receipts at each end, we can then decide when to stop responding. How do we go about it?

### State of an Actor

Let us start with the definition of `Actor` trait:

```rust
pub trait Actor: Sized + Sync + Send + 'static {
    type Msg: Message;       // <--- kind of message this Actor understands
    type State: State;       // <--- data that the actor holds inside during its life
    type Arguments: State;   // <--- data that the actor uses during its initialization
    //...
}
```    

While implementing the Actor for `CanRecognizePing`, we had ignored `State` and `Arguments`. We will make use of `State` this time.

Because we want to keep count of how many times a Ping message has been received so far, we need a counter. When the Actor comes into being, it is initialized to zero. Thereafter, every time the Ping message is handled, the counter is incremented and checked if it has become a certain predefined value. If it has, then the Actor should stop handling any further messages. Straightforward!

Let us say, that this counter is an `u16`. Then, `CanRecognizePing` implements an `Actor` this way:

```rust
#[async_trait::async_trait]
impl Actor for CanRecognizePing {
    // An actor has a message type
    type Msg = PingPongMessage; // As before
    type State = u16;           // <-- The 'type` of counter
    type Arguments = ();        // Ignore this, for the time being
    // ...
}
```

Now, the question is where do we initialize the counter?

Actors have a lifecycle: as they are created, used, stopped and destroyed, they go through stages of the lifecycle. We will explore this topic later; for the time being, important is a stage named `pre_start()` ([more here](https://docs.rs/ractor/latest/ractor/actor/trait.Actor.html#tymethod.pre_start)). This method is *guaranteed to be called* by `ractor`'s runtime, *just after* an Actor is *created* ( `spawned` to be precise ).

When we implement trait `Actor` for `CanRecognizePing`, we are allowed to provide necessary initialization steps, the key being a value for the `State` type:

```rust
#[async_trait::async_trait]
impl Actor for CanRecognizePing {
    type Msg = PingPongMessage;
    type State = u16;
    type Arguments = ();

    async fn pre_start(&self, myself: ActorRef<Self::Msg>, _: ()) -> Result<Self::State, ActorProcessingErr> {
        // We initialize the counter here, but there is no variable named 'count'.
        Ok(0u16)
    }
    // ...
}
```

Initialization is done. Now, the use:

The `CanRecognizePing` is going to use this `State`, when it handles the incoming Ping messages. `ractor`'s Actor runtime provides the value initialized in `pre_start()` function, as a parameter to `handle()` function:

```rust
async fn handle(
        &self,
        myself: ActorRef<Self::Msg>,
        message_known_to_me: Self::Msg,
        count: &mut Self::State,    // <-- count is the `State`, and is initialized to 0!
    ) -> Result<(), ActorProcessingErr> {
        // ...

    }
```

Alright. The `CanRecognizePing` possesses tool to keep track of *count* of messages received so far. For every message received, it increments this count. If the we want the Actor to stop processing any further messages beyond a limit - say, 10 - then the logic specifies that:

```rust
  match message {
         
          PingPongMessage::Ping(sender) => {
            println!("CanRecognizePing has Received a ping message");
            if *count == 10   { //
                println!("Max Ping messages already handled. No further!");
            }
            else {
                *count += 1;
                sender.send_message(PingPongMessage::Pong(myself)).unwrap();
            }
          },
          
          _ => println!("Unknown message {:?} received!", message),

        }
```

The output says that `CanRecognizePing` is indeed quite self-restricted now:

```rust
# ... similar lines printed earlier
CanRecognizePing has Received a ping message
CanRecognizePong has received a pong message
CanRecognizePing has Received a ping message
CanRecognizePong has received a pong message
CanRecognizePing has Received a ping message
Max Ping messages already handled. No further!
CanRecognizePing has Received a ping message
Max Ping messages already handled. No further!
^C
```

Once the `CanRecognizePing` stops further processing, it doesn't send Pong messages to `CanRecognizePong` any more. As a result, the `CanRecognizePong` becomes silent as well. So, the conversation stops. 

The point to note is the storage of `State`: the `count`. It is not returned from the `handle()` method. Instead, the Actor itself remembers the changes made to the count, its state. This modified value is put to use when the next message arrives. This arrangement has implications, which will be clearer as we move forward.

Let us look at the output above, again. The Actors stop conversing but they seem to live on, expecting another message to arrive,some time in the future; which of course, never comes! They have to be told to stop waiting forever. How?

### Starting and Stopping an Actor

We have briefly referred to an Actor's life-cycle, earlier in this article. An Actor is brought into being, is let to process messages and then is ended (destroyed) releasing all the comouting resources it has been holding so far. 

There exists only one way to create an Actor. We have seen this earlier of course:

```rust
let (ping_actor_ref, actor_handle_ping) = 
            Actor::spawn(   // <--- Creating a new Actor
                None, 
                CanRecognizePing, 
                ()
            )
            .await
            .expect("Failed to start actor");
```
 The first element of the tuple above, is an `ActorRef`. From this point onwards, It represents the 'Actor' object that has been created. All interactions with this object is done through this `ActorRef`.

 Once created, the Actor gets busy, waiting for a message to reach it, through its `handle()` function. On receipt of a message, it tries to identify it (through a `match` expression) and takes action accordingly. After that, it again begins its wait for the next message.  This eternal wait for the next message comes to an end, when it is terminated and then, desytroyed. That is the end of its life cycle. 
 
 How do we terminate the actor?

 By asking the Actor to **stop**! 

 In order to stop an Actor, we need to call its `<ActorRef>.stop()` method. Inside the `handle()` function, the Actor has access to its own `ActorRef`. It can use this `ActorRef`, to stop itself.

 ```rust
  // CanRecognizePing...
  match message {
          PingPongMessage::Ping(sender) => {
            println!("CanRecognizePing has Received a ping message");
            if *count == 10   { //
                println!("Max Ping messages already handled. No further!");
                // This actor decides to stop itself.
                myself.stop(Some(String::from("maximum number of ping messages received and processed")));
            }
            else {
                *count += 1;
                sender.send_message(PingPongMessage::Pong(myself)).unwrap();
                println!("Count {}, sending Pong in response once more", *count);
            }
          },
          // ..
  }
  ```

  Once it stops, the `CanRecognizePing` cases to exist. However, the `CanRecognizePong` is not aware of disappearance of its partner in conversation. Thus, it may still try to send another Ping message to the now-deceased `CanRecognizePing` and it will fail.

  ```rust
  // CanRecognizePong
   match message {
         
            PingPongMessage::Pong(sender) => {
                println!("CanRecognizePong has received a pong message");
                sender.send_message(PingPongMessage::Ping(myself)).unwrap(); // <-- This may fail
            },
            _ => println!("Unknown message {:?} received!", message),
        }
  ```

  The output proves that `CanRecognizePong` panics, at that `unwrap()`:

```bash
  # more messages before this
  Count 10, sending Pong in response once more
  CanRecognizePing has Received a ping message
  Max Ping messages already handled. No further!
  CanRecognizePong has received a pong message
  thread 'tokio-runtime-worker' panicked at 'called `Result::unwrap()` on an `Err` value: SendErr', src/participants/pong_capable_actor.rs:44:68
  note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

We can of course, relieve it from its trouble, by dealing with the outcome of `send_message()`:

```rust
    // CanRecognizePong
    match message {

            PingPongMessage::Pong(sender) => {
                println!("CanRecognizePong has received a pong message");
                if let Err(reason) = sender.send_message(PingPongMessage::Ping(myself.clone())) {
                    println!("Error in CanRecognizePong while sending Ping message {:?}", reason);
                    myself.stop(Some(String::from("No receiver. Stopping CanRecognizePong.")));
                }            
            },
            _ => println!("Unknown message {:?} received!", message),
     }
```

This time, the output is more conclusive:

```bash
# more outut before this ....
Count 10, sending Pong in response once more
CanRecognizePing has Received a ping message
Max Ping messages already handled. No further!
CanRecognizePong has received a pong message
Error in CanRecognizePong while sending Ping message SendErr
```

Both the Actors stop themselves, when conditions specific to them, are met. The program then terminates as expected.

The code is structured this way:

![](code-structre.png)

The final code is here.

-------------------------------------

The order in which the messages are printed on the console ( *stdout* ) will almost certainly differ from what I have shown here. Even multiple executions on my machine don't produce them in the same order, every time. This has to do with the **asynchronous nature of intracommunication of Actors**. This is a key concept, required to structure an application around Actors. We will explore it in future tutorials.

--------------------------------------

### Final Note

We have seen

* What are the main components of an Actor-based system, namely
  -   A User-defined compound type - a `struct` -  which wants to behave like an Actor
  -   A trait called `ractor::Actor` which infuses the `struct` above, with the characteristics an Actor
  -   One or more messages which form the vocabulary of the Actors created
* How to create Actors using `Ractor`'s facilities
* How to provide implementation of Actor-like behavior to the `struct`-s mentioned above
* How to let two Actors communicate with one another using the vocabulary stipulated
* How to bring the live Actors to an end, gracefully.

This is very simple set up, and this paves the way for the next tutorial ([here](https://github.com/nsengupta/rust-ractor-tutorial-2)).

------------------------------------------------------------------------------