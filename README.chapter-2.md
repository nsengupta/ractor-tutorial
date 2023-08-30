# rust-ractor-tutorial-2

Part 2 of a tutorial that aims to elaborate Actor-based programming using
'ractor' ( Rust library )

All the code is in './ractor-tutorial-chapter-2/src'. In order to run the
code, move to the aforementioned directory ( `ractor-tutorial-chapter-2` )
and do a `cargo run`.

Code structure:

![Alt text](./project-structure-chapter-2.png "Tutorial-2 structure at a
glance")

# Tutorial

### Prelude

In the previous   tutorial, we had gone through the basics of Actors in `ractor`. We saw the primary constituents of *Actorified* application objects and implementation of a conversation between two Actors. This conversation is built around a vocabulary, a collection of messages which both the Actors understand and can take action on. Additionally, each Actor can keep its own state: one or more pieces of data that held inside the Actor and is available for modification when the next message is processed.

### Inherently non-blocking communication

One of the key aspects of communication between Actors is that it is non-blocking by design. A sender  Actor sends a message and has to remain satisfied with the possibility that:

* It may or may not have been received by the intended Actor.

* It may or may not be processed by the intended Actor.

In the absence of any guarantee that a message has indeed been seen and acted upon by the intended actor,  the protocol of communication between Actors has to be designed with appropriate structures. 

### Vocabulary

If ActorA sends a message M1 to ActorB, it will not know if ActorB has processed it till it hears from ActorB, most possibly through message M2. Thus, messages M1 &  M2  form the vocabulary of conversation between ActorA and ActorB. Both of them have to understand both the messages. Obviously, the number of such messages can be arbitrarily large and that can become unwieldy later. A manageable vocabulary should be comprised of messages which are necessary and sufficient.

### Protocol

Vocabulary is not enough for actors to have a meaningful conversation. A protocol has to accompany it too. Referring to the Actors ( A, B ) and Messages ( M1, M2 ) above, it is important to ascertain, who starts the conversation and who replies. If M2 is passed across only after M1 is received by an Actor, then that is the  basis of the protocol: if ActorA has sent a M1 to ActorB, then ActorA can expect an M2 from ActorB. 

From the previous tutiorial in the series, we know that `handle()` method takes action on a message that arrives.  Drawing upon the structure of `handle()` method, the code for Actors A and B and messages M1 and M2 will look something like:

```rust
#[derive(Debug, Clone)]
pub enum PrivateVocabulary {
    M1,
    M2,
}

// ....

// For Actor A
async fn handle(
       // ..
    ) -> Result<(), ActorProcessingErr> {
        match message {
           PrivateVocabulary::M2 => // ... It understands M2
           _  => // ....
        } 
        Ok(())
    }

// For Actor B
async fn handle(
       // ..
    ) -> Result<(), ActorProcessingErr> {
        match message {
           PrivateVocabulary::M1 => // ... It understands M1
           _  => // ....
        } 
        Ok(())
    }
```

It is clear that when preparing the message-handling logic of an Actor ( the `handle()` method to be precise ), the messages that the Actor is supposed to understand and possibly take action on, are the primary considerations. Only when someone sends such a message, the Actor stirs into action!

### Who can send messages to an Actor

Any part of the application, where an `ActorRef` is available, can send a message to the Actor it *refers* to. To put it more clearly, the following are sources of messages that an Actor can receive:

- From other Actors (assuming that these 'other' Actors has the `ActorRef`  of the recipient Actor)

- From other methods of the application - including `main()` 

- From within itself (to itself,  so as to say)

- From the runtime of `ractor` ( messages that scheduled for a point of time in future)

### Order of arrival of messages

If ActorB and ActorC send messages to ActorA, then order in which these messages reach ActorA can be random, but messages from any particular Actor will reach ActorA exactly in the order in which they are sent. To elaborate, say:

ActorB sends messages m1B, m2B, m3B to ActorA, in that order

ActorC sends messages m1C, m2C, m3C to ActorA, in that order

Then, ActorA may receive messages in a mix of combined order, like:

m1B; m1C, m2C, m2B, m3B, m3C     OR

m1B, m2B, m1C, m2C, m3C, m3B     OR

m1B, m2B, m3B, m1C, m2C, m3C     OR

any such mix of m[123]{B|C}  but

messages from ActorB will always be in the order of m1B,m2B,m3B. Message m3B will **never** reach ActorA, before m2B which in turn, will never reach ActorA before m1B. Ditto for messages from ActorC!

This may seem obviousl but this guarantee has surprising and beneficial outcomes, as we will see.

### Messages are passed non-blockingly

When an Actor sends a message to another actor, the semantics of sending is equivalent to non-blocking function calls. The act of send returns immediately (if the target Actor is non-existent, then an error is generated but that is not important here). and the caller doesn't know if and when, the target Actor will process the message. This looseness of guarantee calls for a very different approach towards weaving the logic of an application: usual ways to design the application's flow by calling  method (or functions) and using their return values.

This carries siginificant ramifications. For example:

Say, ActorA asks ActorB and ActorC for a particular value. It will send a message to the latter two, non-blockingly. That means, once the send action succeeds (and *returns*) ActorA now has to find some means of knowing:

* if both of them have received/understood/ignored/rejected/processed  the messages

* when both of them have responded to the message (with the value asked for)

The problem is the first is impossible to verify till the second happens; but, then the second may happen, may happen but much later than expected or may never happen! 

In this section of the tutorial, we will try and model such situations.

### Actors participating in the tutorial

We bring in two actors we had built in the previous tutorial:

- A Pong-aware Actor: it understands Pong messages and responds with a Ping message

- A Ping-aware Actor: it understands Ping messages and responds with a Pong message

They interact between themselves by exchanging Ping and Pong messages a _fixed_ number of times and then stop themselves (and then destroyed).

We introduce a 3rd Actor. Its job is to inquire the abovementioned two Actors - as if to say, 'are you fine and dandy?' - and the two respond with 'I am fine, thank you' along with a running serial number. This number helps to keep track of how many times, have these two been inquired. In the accompanying code, this 3rd "Inquirer" Actor is referred to as `CanInquire` :

```rust
pub struct CanInquire;

// the implementation of our actor's "logic". It is built to handle messages that deal with
// inquiries only (refer to pingpong_vocabulary)
#[async_trait::async_trait]
impl Actor for CanInquire {
    type Msg = PingPongMessage;
    type State = Option<PingPongCountInquiry>;
    // No startup arguments for actor initialization, at the moment
    type Arguments = ();
```

To accommodate this additional interaction, the vocabulary has to change:

```rust
pub enum PingPongMessage {
    Ping(ActorRef<PingPongMessage>),
    Pong(ActorRef<PingPongMessage>),
    Inquire(ActorRef<PingPongMessage>),
    StartInquiry((u16,ActorRef<PingPongMessage>,ActorRef<PingPongMessage>)),
    InquiryReport(u16,ActorRef<PingPongMessage>)
}
```

The three new messages added to the vocabulary are:

- `Inquire(ActorRef<PingPongMessage>)` This message is sent by the Inquirer to "{ Ping | Pong } Aware" Actors. It carries the reference to the sender Actor (the _Inquirer_ ) itself. The target Ping|Pong Actors can use this for responding.

- `InquiryReport(u16,ActorRef<PingPongMessage>)` This message is sent in response to the message above by the Ping|Pong Actors, to the Inquirer. It carries a number that represents a running serial number of the responses and a reference to the sender.

- `StartInquiry((u16,ActorRef,ActorRef))`: we will take a look at it later in this tutorial.

The following sequence diagram makes it clearer:

![Alt text](./ractor-tutorial-chapter-2/accompanying-content/ractor-tutorial-interaction-Inquirer-Ping-Pong-1.png)

It is important to remember that the interaction is completely non-blocking. There is **no guarantee** that PongAware will receive the message **later than** when PingAware will receive the message. Ditto for Inquirer receiving the responses: the order is completely random.

The diagram above doesn't capture the whole set of interactions between the Actors. The Ping | Pong Actors also exchange Ping <--> Pong messages between themselves (and the Inquirer knows nothing about that).

![Alt text](./ractor-tutorial-chapter-2/accompanying-content/ractor-tutorial-Inquiry-PingPong-conversation-2.png)

Again, the order of arrival is unknown, in all cases.

Now, the Ping|Pong Actors have to be equipped with the logic to deal with either of the messages: ( a ) Inquiry and ( b ) Ping / Pong. 

For PongAware Actor:

```rust
async fn handle(
        &self,
        myself: ActorRef<Self::Msg>,
        message: Self::Msg,
        counts: &mut Self::State,
    ) -> Result<(), ActorProcessingErr> {

        match message {
            PingPongMessage::Inquire(sender) => {
               // ...
            }, 
            PingPongMessage::Pong(sender) => { // <--- Pong only
               // ...
            }, 
            _ => println!("{} received Unknown message {:?}!", myself.get_name().unwrap(),message),
        }
```

For PingAware Actor:

```rust
async fn handle(
        &self,
        myself: ActorRef<Self::Msg>,
        message: Self::Msg,
        counts: &mut Self::State,
    ) -> Result<(), ActorProcessingErr> {

        match message {
            PingPongMessage::Inquire(sender) => {
                // ...
            },
            PingPongMessage::Ping(sender) => { // <-- Ping only
                // ...
            },
            _ => println!("{} received Unknown message {:?}!", myself.get_name().unwrap(),message),

        }
```

The ways `PingPongMessage::Ping(sender)` and `PingPongMessage::Pong(sender)` are handled, ale almost exactly the same as what we had seen in the previous ([ here](./ractor-tutorial-chapter-1/README.chapter-1.md))) tutorial. In short, each of the Actors keeps a count of how many times the message has been received. If that number is less than a pre-defined limit, then it responds no more and stops itself; otherwise, it responds with a corresponding message (Ping | Pong, as applicable).

### Handling inquiries: how the Inquirer works

- It waits for a `PingPongMessage::StartInquiry(((u16,ActorRef<PingPongMessage>,ActorRef<PingPongMessage>))`. The message is interpreted as:
  
  - u16 indicates the max  number of inquiries to be made
  
  - first ActorRef is of PingAware Actor; it will be inquired
  
  - sscond ActorRef is of PongAware Actor; it will be inquired as well

- At this point, the Inquirer does two things:
  
  - It sets up its own data structure, used for tracking the inquiries. This data structure is initialized as its own `State` and is used to track the number of inquiries which is limited by the maximum set in the parameter passed.
  
  - It sends the first inquiries to the Ping | Pong Actors. 

```rust
async fn handle(
        &self,
        myself: ActorRef<Self::Msg>,
        message: Self::Msg,
        state: &mut Self::State,
    ) -> Result<(), ActorProcessingErr> {

        match message {
            PingPongMessage::StartInquiry(inquiry_instructions) => {    
                if let None = state {
                    *state = Some(PingPongCountInquiry::new ( 
                        inquiry_instructions.max_inquiries_to_be_made, 
                        inquiry_instructions.ping_actor_to_inquire, 
                        inquiry_instructions.pong_actor_to_inquire 
                    ));

                };

                // This actor understands Ping
                inquiry_instructions.ping_actor_to_inquire
                    .send_message(
                        PingPongMessage::Inquire (myself.clone() )
                    )
                    .unwrap();

                // This actor understands Pong
                inquiry_instructions.ping_actor_to_inquire
                    .send_message(
                        PingPongMessage::Inquire (myself.clone() ))
                    .unwrap();
            },
```

`StartInquiry` is a `PingPongMessage`:

```rust
#[derive(Debug, Clone)]
pub struct InquiryInstructions {
    pub max_inquiries_to_be_made: u16,
    pub ping_actor_to_inquire: ActorRef<PingPongMessage>,
    pub pong_actor_to_inquire: ActorRef<PingPongMessage>
}

// These constitute the vocabulary of interactions, using `PingPongMessage`
#[derive(Debug, Clone)]
pub enum PingPongMessage {
    Ping(ActorRef<PingPongMessage>),
    Pong(ActorRef<PingPongMessage>),
    Inquire(ActorRef<PingPongMessage>),
    StartInquiry(InquiryInstructions),
    InquiryReport(u16,ActorRef<PingPongMessage>)
}
```

Using the contents of `StartInquiry`, the Inquirer Actor initializes its own state:

```rust
if let None = state {
               *state = Some(PingPongCountInquiry::new ( 
                        inquiry_instructions.max_inquiries_to_be_made, 
                        inquiry_instructions.ping_actor_to_inquire, 
                        inquiry_instructions.pong_actor_to_inquire 
               ));

       };
```

The `state` is an Option: this is so, because the when the Inquirer Actor is initialized ( by `pre_start()` lifecycle method), the state is bereft of any known value. The first time it gets that is when `StartInquiry` is received.

```rust
impl Actor for CanInquire {
    type Msg = PingPongMessage;
    type State = Option<PingPongCountInquiry>;
    // No startup arguments for actor initialization, at the moment
    type Arguments = ();

    async fn pre_start(&self, myself: ActorRef<Self::Msg>, _: ()) -> Result<Self::State, ActorProcessingErr> {
        Ok(None)  // <-- Initialized with Some(..) later
    }
    // ....
```

Upon receiving the `Inquire(actor_ref)`, the Ping|Pong Actors are supposed to respnd with a `InquiryReport(u16,ActorRef<PingPongMessage>)`. That `u16` carries a serial number, generated by the Ping | Pong Actors. The Inquirer Actor checks the maximum number of inquires have been made to the Ping | Pong Actors. If so, then the Inquirer Actor stops itself.

```rust
PingPongMessage::InquiryReport(serial_received, from_actor) => {
     if let Some(state) = state {
          let sender_name = from_actor.get_name().unwrap();
          let last_report = state.track_reports_of_inquiry(sender_name.as_str());

          if state.are_we_done_inquiring() {
               myself.stop(None) // <-- stopping itself
          }
          else {
             from_actor
               .send_message(
                    PingPongMessage::Inquire (myself.clone())
               )
              .unwrap();
              state.track_reports_of_inquiry(sender_name.as_str()); 
          }
      }        
},
```

Let us explore the `state` of Inquirer Actor:

```rust
impl Actor for CanInquire {
    type Msg = PingPongMessage;
    type State = Option<PingPongCountInquiry>;
    // No startup arguments for actor initialization, at the moment
    type Arguments = ();

    async fn pre_start(&self, myself: ActorRef<Self::Msg>, _: ()) -> Result<Self::State, ActorProcessingErr> {
        Ok(None)
    }
```

The `state` is an `Option<PingPingCountInquiry>`. 

```rust
#[derive(Debug)]
pub struct InquiryReportTracker (ActorRef<PingPongMessage>, u16);

#[derive(Debug)]
pub struct PingPongCountInquiry {
    max_inquiry_reports_expected: u16,
    actor_trackers: HashMap<String, InquiryReportTracker>,
}
```

It prepares a HashMap of Actors and the count of inquiries made so far. The Inquirer Actor uses this  to check if maximum number of inquiries have been made to Ping | Pong Actors.

```rust
pub fn track_reports_of_inquiry(&mut self, actor_name: &str) -> u16 {
  let may_be_an_actor = self.actor_trackers.get_mut(actor_name).unwrap();
  may_be_an_actor.1 += 1u16;
  may_be_an_actor.1
}

pub fn are_we_done_inquiring(&self) -> bool {
    let mut combined_inquiry_completion = true;
    for (k,v) in self.actor_trackers.iter() {
    // even if inquiry reports are yet to be received 
    // from 1 of the actors, we have to wait
        if v.1 < self.max_inquiry_reports_expected {
            // Admittedly, not very idiomatic, but gets the job done! 
            combined_inquiry_completion = 
                     combined_inquiry_completion || false;
            }
    }
   return combined_inquiry_completion;
}
```

### Putting things together

Referring to the sequence diagrams above, we see what the arrangement is. Ping and Pong Actors can keep exchaning messages between themselves, as long as the limiting condition is not reached. Meanwhile, the Inquirer Actor keeps on inquiring with these two, intermittently. They submit reports of inquiry, dutifully, while continuing with their own conversation. These two separate conversations continue till their terminating conditions have reached and the stop themselves.

The relevant portion (below) in `main()` tells the story:

```rust
// ....
ping_actor_ref.send_message(
                PingPongMessage::Ping(pong_actor_ref.clone())
).unwrap();

pong_actor_ref.send_message(
                PingPongMessage::Pong(ping_actor_ref.clone())
).unwrap();

inquirer_actor_ref.send_message(
        PingPongMessage::StartInquiry(
            InquiryInstructions {
                max_inquiries_to_be_made: 5u16,
                ping_actor_to_inquire: ping_actor_ref.clone(),
                pong_actor_to_inquire: pong_actor_ref.clone()
            }
 ))
 .unwrap();


// ....
```

The Actors, after being created, are set on, by means of messages. Ping Actor receives a PING message and gets into action. It sends a PONG message to Pong Actor. Similarly, Pong Actor stirs into action when it receives the first PONG message. Inquirer Actor is told to `StartInquiry` and it begins to inquire, as described earlier in this article. In other words, `main()` starts the ball rolling by sending appropriate messages to the Actors. Thereafter, the Actors converse amongst themselves.

### Output

If the program is run, the output will look something like this:

```bash
PongAware-Actor <- Inquirer-Actor, inquiry 
Inquirer-Actor <-- PongAware-Actor, report serial[0], of total 1 so far!
PingAware-Actor <- Inquirer-Actor, inquiry 
PongAware-Actor <- Inquirer-Actor, inquiry 
Inquirer-Actor <-- PingAware-Actor, report serial[0], of total 1 so far!
Inquirer-Actor <-- PongAware-Actor, report serial[1], of total 2 so far!
PingAware-Actor <- Inquirer-Actor, inquiry 
PongAware-Actor <- Inquirer-Actor, inquiry 
Inquirer-Actor <-- PingAware-Actor, report serial[1], of total 2 so far!
Inquirer-Actor <-- PongAware-Actor, report serial[2], of total 3 so far!
PingAware-Actor <- Inquirer-Actor, inquiry 
Inquirer-Actor <-- PingAware-Actor, report serial[2], of total 3 so far!
# .... (more lines here)
```

However, the output is very likely to differ between runs. In some case, the program may seem to hang, even. Simply put, the program will not seem to run to completion, uniformly, every time! 

Why?

### Asynchronousness

Because the order in which messages from other Actors, reach a particular target Actor, is unknown, exactly when the targer Actor processes a particular message is unpredictable. Additionally, the action that the target Actor takes on receipt of a message, may affect the way it processes the other messages. 

Such non-determinism in message processing is responsible for the variations in the output. But Actors are meant to process messages asynchronously and therefore, the non-determinism in processing is unavoidable. How shall make the application produce predictable output? 

That is the topic of the [next](./ractor-tutorial-chapter-3) tutorial.

--------------------------------------------
