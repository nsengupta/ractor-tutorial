# rust-ractor-tutorial-3

Part 3 of a tutorial that aims to elaborate Actor-based programming using
'ractor' ( Rust library )

All the code is in './ractor-tutorial-chapter-3/src'. In order to run the
code, move to the aforementioned directory ( `ractor-tutorial-chapter-3` )
and do a `cargo run`.

Code structure:

![Alt text](./project-structure-chapter-3.png "Tutorial-3 structure at a
glance")

# Tutorial

### Prelude

In the previous tutorial, we had arranged for 3 Actors, namely *Inquirer*, *PingAware* and *PongAware*. Ping & Pong Aware Actors exchange messages between themselves for a predefined number of times. Meanwhile, the Inquirer, knocks at their doors asking for some information. They courteously respond, to the Inquirer. This exchange also happens for a certain predefined number of times.

When we ran the application, the output were not the same every time. While that is not unexpected, we found that several times, the application didn't terminate gracefully; in fact, many times, it hanged.

Why?

### That matter of asynchronous communication

The key aspects of asynchronous message-passing between Actors are:

- Any one can send a message to an (target) Actor. However, when that (target) Actor will be able to take a desired action on that message and how, are not known to the sender. 

- It is possible that the target Actor is busy servicing another message, sent by someone else, at that moment, or it is possible that the target Actor has indeed received the message alright but has decided not to respond to it due to some logic of its own. The sender Actor must be prepared to deal with this.

- It is also possible that the act of 'sending' has been  successful and the sender believes that the target Actor must be working on it soon. However, in the meantime, the target Actor has gone out of existence. The sender is never going to know. 

### Revisiting the Actor trio in the tutorial

The PingAware Actor:

- Responds to `PingPongMessage::Inquire` message from the Inquirer Actor just after it receives it, with a sequentially increasing integer number.

- Responds to `PingPongMessage::Ping` with a `PingPongMessage:Pong` but only if the number of responses so far has not exceeded a limit. If it has, then the Actor terminates itself.

The PongAware Actor:

- Responds to `PingPongMessage::Inquire` message from the Inquirer Actor just after it receives it, with a sequentially increasing integer number.

- Responds to `PingPongMessage::Pong with a` PingPongMessage:Ping` but it doesn't keep track of number of such responses. In other words, it can keep responding as long as it receives a Pong message. It imposes no condition on itself to decide if and when to go out of existence.

The Inquirer Actor:

- Inquires both the Actors and keeps track of responses from both. If the number of responses from *both* Ping and Pon Aware Actors reach a pre-defined number, it terminates itself.

Let's think about the implications of all these, a bit. 

For example, the Inquirer Actor doesn't know that Ping | Pong Aware Actors know one another and they are conversing too. That is alright because functionally, that is the correct depiction of what needs to happen. However, the PingAware Actor can go out of existence, if the aforesaid limit of Ping messages is reached, and the Inquirer Actor remains unaware of the disappearance of its inquiry's target.

### What happens first?

Does the PingAware Actor hear from the Inquirer first or from the PongAware first? One doesn't know. 

Does the PingAware Actor receive *several* messages from PongAware Actor before it receives the *first* message from the Inquirer? Quite possible.

Let's say that the PingAware Actor is codified to exchange messages with PongAware Actor 5 times max and after that, it terminates itself. Given this and extending the preceding point, can the PingAware finish responding to all 5 Ping messages and then die, and as a result, the Inquirer **never even gets a chance** to inquire it? Quite possible, again!

A few other such possibilites exist in the application.

The point is that **sequentiality of interactions** is not guaranteed. Put simply, if Actor A sends 2 messages, one each to Actor B and Actor C, and expects responses from both of them, then Actor A must be ready for response from either of them arriving in **any order**. Actor A must not assume that because it sends the message to Actor B first lexically, the response from Actor B will arrive first. Moreover, it may never arrive!

### Effect on the application's behaviour

This is the partial output of 1 run:

```bash
# ... truncated
Inquirer-Actor, checking if all inquiries done: Actor PingAware-Actor, So far 1, max-allowed 5
PongAware-Actor <- Inquirer-Actor, inquiry 
Inquirer-Actor, checking if all inquiries done: Actor PongAware-Actor, So far 0, max-allowed 5
PongAware-Actor <- PingAware-Actor, a pong message
Inquirer-Actor, checking if all inquiries done: Actor PingAware-Actor, So far 1, max-allowed 5
Inquirer-Actor, checking if all inquiries done: Actor PongAware-Actor, So far 1, max-allowed 5
PingAware-Actor <- PongAware-Actor, a ping message
PingAware-Actor <- PongAware-Actor, a ping message
PongAware-Actor <- PingAware-Actor, a pong message
PingAware-Actor <- PongAware-Actor, a ping message
PongAware-Actor <- PingAware-Actor, a pong message
# .. more of these, total 10
PingAware-Actor handled max Ping messages already. No further! # PingAware is terminated after this!
Inquirer-Actor, checking if all inquiries done: Actor PingAware-Actor, So far 2, max-allowed 5
Inquirer-Actor, checking if all inquiries done: Actor PongAware-Actor, So far 1, max-allowed 5
thread 'tokio-runtime-worker' panicked at 'called `Result::unwrap()` on an `Err` value: SendErr', src/participants/object_can_inquire.rs:71:92
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
# hangs ...
```

Because PingAware Actor has exchanged  10 messages with PongAware Actor, it **terminates** itself. At this point, PongAware Actor is living on, but is in a limbo because it never gets any further message from PingAware Actor. Inquirer Actor fails to send next inquiry to PingAware Actor because it doesn't exist any more and panics! 

Here's the partial  output of another run:

```bash
# .... truncated
PingAware-Actor handled max Ping messages already. No further!
Inquirer-Actor <-- PingAware-Actor, Inquiry report, total 3 so far!
Inquirer-Actor, checking if all inquiries done: Actor PongAware-Actor, So far 2, max-allowed 5
Inquirer-Actor, checking if all inquiries done: Actor PingAware-Actor, So far 3, max-allowed 5
thread 'tokio-runtime-worker' panicked at 'called `Result::unwrap()` on an `Err` value: SendErr', src/participants/object_can_inquire.rs:71:92
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace\
```

PingAware Actor is terminating itself. However, Inquirer is processing PingAware Actor's last inquiry-report in its own time and then trying to send another inquiry to it. Obviously, it fails to do so and panics!

Let us reorganize the code and prepare ourselves for such eventualities, so that the application behaves predictably.

### Reorganisation

##### Set limits at the start of the application

There are two limits we are posing on the application: (a) maximum number of inquiry messages to be exchanged between Inquirer < -- > PingAware and Inquirer < -- > PongAware Actors and (b) maximum of number of Ping messages that the PingAware Actor responds to. Instead of hard-coding these values, we set them in the `main` at the beginning (ideally, we should pass in those values at the command-line but let's keep that for another time):

```rust
let start_arguments = MaxLimitArguments {
        mx_inquiry_allowed: 5u16,  // <-- how many inquiries
        mx_ping_messages_exchanged: 10u16 // <-- how many PING messages to respond to
    };
```

##### Use macro to send messages and return errors early

ractor provides a macro called `cast` (more [here]([cast in ractor - Rust](https://docs.rs/ractor/latest/ractor/macro.cast.html))) to handle the act of sending a message. We will replace all explicit `send_message` call with this macro. Additionally, we will *return early (using '?' operator)* when an error occurs, in keeping with Rust's norm.

```rust
myself.cast(PingPongMessage::CleanupIfAllDone)?;
```

##### Give Actors their own names

This sounds trivial but it is important to give user-defined and easily-readable names to each Actor that is created. This small step goes a long way in tracing and monitoring in non-trivial applications producing, working with and destroying large number of Actors:

```rust
 // PingAware Actor
 Actor::spawn(
        Some(String::from("PingAware-Actor")), // <-- Name!
        CanRecognizePing, 
        PingAwareArguments(start_arguments.mx_ping_messages_exchanged)
    ).await
    .expect("Failed to start actor");
```

##### Initialize the Inquirer Actor with references to other two Actors

As per the application's scope, the Inquirer Actor needs to know which other two Actors it has to interact with. Therefore, those two Actors (namely Ping | Pong Aware) have to pre-exist! So, it is alright to initialize the Inquirer Actor with the `ActorRef` of Ping | Pong Actors, instead passing these through messages (we did that  in the previous tutorial):

```rust
// In ./configuration/arguments.rs
pub struct InquirerArguments {
    pub mx_inquiry_allowed: u16,
    pub ping_actor_to_inquire: ActorRef<PingPongMessage>,
    pub pong_actor_to_inquire: ActorRef<PingPongMessage>,
}

// ...

// In ./participants/object_can_inquire.rs
impl Actor for CanInquire {
    type Msg = PingPongMessage;
    type State = PingPongCountInquiry;
    // No startup arguments for actor initialization, at the moment
    type Arguments = InquirerArguments;

    async fn pre_start(&self, myself: ActorRef<Self::Msg>, start_arguments: InquirerArguments) -> Result<Self::State, ActorProcessingErr> {
        Ok(PingPongCountInquiry:: new (
            start_arguments.mx_inquiry_allowed,
            start_arguments.ping_actor_to_inquire,
            start_arguments.pong_actor_to_inquire
        ))
    }
// ...

// In ./main.rs
let (actor_ref_inquire, actor_handle_inquire) = 
       Actor::spawn(
            Some(String::from("Inquirer-Actor")),
            CanInquire, 
            InquirerArguments {
                  mx_inquiry_allowed: start_arguments.mx_inquiry_allowed,
                  ping_actor_to_inquire: ping_actor_ref.clone(),
                  pong_actor_to_inquire: pong_actor_ref.clone()
            }
        ).await
        .expect("Failed to start actor");
// ...
```

##### Initialize PingAware Actor with the max PING messsages to handle

This is straightforward:

```rust
// ./participants/object_can_understand_ping.rs
pub struct ControlParameters {
        pub max_ping: u16,
        pub so_far_ping: u16,
        pub last_ping_serial: u16
}

// ..
async fn pre_start(&self, myself: ActorRef<Self::Msg>, args: PingAwareArguments) 
        -> Result<Self::State, ActorProcessingErr> {
        Ok(Parameters::ControlParameters { 
                max_ping: args.0, 
                so_far_ping: 0, 
                last_ping_serial: 0
        }) 
}

// ..
```

##### Prevent overflowing when keeping count of responses to inquiries

This is straightforward as well:

```rust
// In pingpong_inquiry/inquiry_block.rs
pub fn track_reports_of_inquiry(&mut self, actor_name: &str) -> u16 {
     let may_be_an_actor = self.actor_trackers.get_mut(actor_name).unwrap();
        // Prevent unneccessary increment; no overflowing allowed
        if may_be_an_actor.1 < self.max_inquiry_reports_expected {
            may_be_an_actor.1 += 1u16;
     }
     may_be_an_actor.1
}
```

##### Give PongAware Actor a chance to stop itself

PongAware Actor has a predicament. It responds to *Inquirer* Actor, with InquiryReport messages. It talks to PingAware Actor by sending *Ping* messages and receiving *Pong* messages. Now, either (or both) of Inquirer and PingAware may decide to stop themselves, and the PongAware Actor will not come to know of their disappearance, unless it tries to send another message to them. This means, it may keep waiting for the next message which never arrives!

Essentially, there are 2 ways in which an Actor can be stopped gracefully: ( a ) send a special message to it or ( b ) let it have its own logic to decide. In this particular case, we will go for the first.

--------------------------------------

An Actor can be _killed_ also, but that is not exactly a graceful stoppage. We are not considering that here.

---------------------------------------------

PongAware has no information about the existence of the other two Actors. Till either of them sends a message, it sits idle. But, for how long? Not for an indefinite period. 

It can send itself a reminder though, after a certain predefined duration. When the reminder arrives, it assumes that other two Actors are unlikely to knock at its door and it decides to stop itself. 

How do we do this?

An Actor can use `ractor`'s facilities to set a reminder for itself:

```rust
let reminder_handle = 
        ractor::time::send_after(
                     Duration::from_millis(10), // <-- when
                     myself.get_cell(),         // <-- whom       
                     || { 
                            PingPongMessage::NoInquiryReportReceivedSince(
                                Duration::from_millis(10)
                            ) // <-- what
                     }          
        );
```

`ractor` will send a `NoInquiryReportReceivedSince` message to the Actor after (*at least*, more accurately) 10 milliseconds. 

On receipt of this message, PongAware Actor decides that sufficient time has passed since it has heard from any of the other 2 Actors and that it is time to stop its own operation.

What about cascaded reminders, when one reminder is set even though the previous reminder is still brewing? 

Well, that is troublesome because the PongAware Actor will not be able differentiate between two reminders. Therefore, the preceding reminder has to taken off and let the latest be the only one alive.

When we initialize PongActor, we leave a place for an eventual reminder, in the `state`:

```rust
async fn pre_start(&self, myself: ActorRef<Self::Msg>, _: ()) 
    -> Result<Self::State, ActorProcessingErr> {
        // first: count of Pong messages; 
        // second: running serial for inquiry reports; 
        // third: placeholder for a reminder
        Ok((0u16, 0u16,None)) 
    }
```

When we are about to set a reminder, we should check if one exists and if so, then remove it:

```rust
// Remember, that we initialize this Actor with no reminder!
// Extract it!
let (_, _, may_be_handle) = counts; // the 'state'

// It is possible that that a reminder set earlier is still alive.
if let Some(existing_handle) = may_be_handle {
   existing_handle.abort(); // <-- remove
}
*counts = 
    (counts.0,counts.1,Some(reminder_handle)); // <-- install afresh!
```

When the reminder arrives, the PongAware Actor stops itself:

```rust
PingPongMessage::NoInquiryReportReceivedSince(duration) => {
 // ..
   myself.stop(None); // <-- stops itself!
},
```

#### The Inquirer needs to know if any further inquiry is of any use

Here is the situation, explained earlier in section "What happens first", once more: 

The Inquirer Actor initiates a conversation: it sends an `Inquiry` to both of Ping | Pong Aware Actors and receives `InquiryReport` from them. However, these two targets of inqury also converse between themselves, and may decide to stop based on a condition that is unknown to the Inquirer. In other words, the Inquirer is likely to fail to send messages to them, because of a condition that is beyond its knowledge. This is a key observation.

Till the time it tries to send a message to Ping | Pong Aware and fails, the Inquirer cannot decide if it makes sense to send further messages. Furthermore, recall that the PingAware Actor may have been gone (its own decision) but the PongAware may still be alive. The Inquirer has to deal with this partial availability too.

The Inquirer tracks the number of responses from either of these two Actors, in a Map:

```rust
// In src/conversation/pingpong_vocabulary.rs
pub struct InquiryReportTracker (pub ActorRef<PingPongMessage>, pub u16);

// ..
pub struct PingPongCountInquiry {
    max_inquiry_reports_expected: u16,
    actor_trackers: HashMap<String, InquiryReportTracker>,
}
```

and, makes use of some helper methods:

```rust
 // In src/conversations/pingpong_vocabulary.rs
pub fn are_max_inquiries_done_with_target(&self, actor_name: &str) -> bool {
    self.actor_trackers.get(actor_name)
        .is_some_and(|may_be_an_actor| 
                may_be_an_actor.1 == self.max_inquiry_reports_expected
        )
}

pub fn track_reports_of_inquiry(&mut self, actor_name: &str) -> u16 {
    let may_be_an_actor = self.actor_trackers.get_mut(actor_name).unwrap();
    // Prevent unneccessary increment; no overflowing allowed
    if may_be_an_actor.1 < self.max_inquiry_reports_expected {
            may_be_an_actor.1 += 1u16;
    }
    may_be_an_actor.1
}

pub fn remove_target_actor(&mut self, actor_name: &str) -> u16 {
    if let Some(may_be_existing_actor) =  self.actor_trackers.remove(actor_name) {
            may_be_existing_actor.1
    }
    else {
           0 as u16
    }
}
 // .. some more
```

When an `InquiryReport`` arrives from either of the Actors, the Inquirer Actor checks the conditions:

```rust
// In src/participants/objects_can_inquire.rs
PingPongMessage::InquiryReport(serial_received, from_actor) => {
   let sender_name = from_actor.get_name().unwrap();
   let last_report = state.track_reports_of_inquiry(sender_name.as_str());

   if !state.are_max_inquiries_done_with_target(&sender_name) { // Cond[1]
      // failing to send the next message means that the target may
      // not be alive anymore
      if let Err(_) = // Cond[2]
          from_actor.cast(PingPongMessage::Inquire(myself.clone())) {
               // We are not interested to track this target anymore!
               state.remove_target_actor(&sender_name);
               myself.cast(PingPongMessage::CleanupIfAllDone)?;
           }
           else { // Cond [2]
              send_after(
                   Duration::from_millis(10), 
                   myself.get_cell(),
                   || { PingPongMessage::CheckAreTargetsAlive }
              );   
           }
   }
   else { // Cond[1]
     // Maximum inquiries are made. Now, the target can be dismissed.
     state.remove_target_actor(&sender_name);
     myself.cast(PingPongMessage::CleanupIfAllDone)?;
   }
```

- Cond[1] is straightforward. The Inquirer is checking if all inquires (pre-configured) made to this target Actor have been responded to. If those have been, then it is alright to dismiss that target Actor (Ping Aware or Pong Aware); the `else` block does the needful. 

- Cond[2] is to check if the Inquirer can send the next message to the (same) target. If it fails, then it is assumed that the target Actor is non-existent (e.g., the PingAware Actor has decided to stop itself, meanwhile). in that case, the target can be dismissed because there is no point in trying to contact it any further.

The so-called _happy path_ can follow this. Remember though, that our intention is to make the application's behaviour predictable. Everytime we run it, we want it finish gracefully (no hanging etc.).

#### Unpredictability of existence of target Actors

Let us revisit the interaction between the Inquirer and Ping | Pong Aware Actors. Very importantly, the Ping | Pong Aware Actors **respond** to the Inquirer; by themselves, they send nothing to the Inquirer. Therefore, the only way for the Inquirer to know if they exist, is to try and send another `Inquiry` message to them and watch if that attempt, fails!

**Possibility 1**

Ping and Pong Aware Actors converse between themselves. Once it has responded to a pre-defined number of Ping messages, the Ping Aware Actor stops itself. This entire saga happens completely without Inquirer's knowledge!

**Possibility 2**

Pong Aware Actor sets up a timer for itself which it uses to guess if the Ping Aware Actor is not going to respond any more. Once the Pong Aware Actor infers the Ping Aware Actor's disappearance, it stops itself. Again, the Inquirer Actor never knows, when the Pong Aware goes out of existence.

**Possibility 3**

The Inquirer receives a few responses from Ping Aware Actor and suddenly. it receives nothing more. Because the Inquirer sends the next inquiry *only after* receving the previous response, it will not be in a position to send any further message to the Ping Aware Actor. It will be in a limbo: waiting for something which never comes!

There exist a few other similar scenarios. In all of these, the Inquirer may remain undecided and non-moving for ever. The only way for it not to be in that stupor, is to send a message to itself through a timer! 

Message to self `CheckTargetsAreAlive` :

On receipt of this message, the Inquirer Actor tries to ascertain if 2 target Actors are alive and if not, then it tries to wait for some more time and then, check again. 

```rust
 // In src/participants/object_can_inquire.rs
 PingPongMessage::CheckAreTargetsAlive => {
     // get the remaining targets
     let remaining: Vec<_> = 
           state
           .get_remaining_targets()
           .map(|next_t| &next_t.0)
           .cloned()
           .collect();

     // Try to reach, each of the targets. On failure, take that off the
     // tracker.
     for next_target in remaining.iter() {
         if let Err(reason) =
                next_target.cast(PingPongMessage::Inquire(myself.clone()))
         {
           state.remove_target_actor(
                 next_target.get_name().unwrap().as_str()
           );
         }
     }

     // Send a message to itself, to clean itself            
     myself.cast(PingPongMessage::CleanupIfAllDone)?;
}
```

Message to self `CleanUpIfAllDone` :

On receipt of this message, the Inquirer Actor confirms that none of the targets is alive any more and if so, stops itself.

```rust
// In src/participants/object_can_inquire.rs
PingPongMessage::CleanupIfAllDone => {
    if state.no_inquiry_target_still_remaining() {
           myself.stop(Some(String::from("All inquiries done!")));
    }
    else {
        // At least one target still seems to exist.
        // Check again after some time.
           send_after(
              Duration::from_millis(20), 
              myself.get_cell(),
              || { PingPongMessage::CheckAreTargetsAlive }
            ); 
         }
}
```

One message helps create a condition and another message decides on the condition! In message-driven design, this is a useful pattern. 

Because of such an arrangement, the Inquirer is guaranteed to be informed of non-existence of the two target ( Ping | Pong Aware ) Actors. It gets a chance to stop itself confidently and cleanly. 

The outputs of two successive runs may be very different but, the application will eventually terminate cleanly! Every time!

#### Asynchronousness and determinable behaviour

Let's refer to the section "That matter of asynchronous communication" at the beginning. Now that we have coded the behaviour, the implication of what that section says, is clearer.

The asynchronous messaging leads to situations where the *send* action, the 
*processing* action and the *receive* action are handled at different times 
and most possibly by different threads (therefore, not in a predictable 
sequence). Moreover, an Actor can receive messages in random order. 
Crucially, its behaviour is dependent on the order of processing  of the 
messages.

But even then, the Actors must be designed in such a way, that their 
behaviour remains deterministic. The **design** is the operative word here. Building an Actor-based application is primarily about modeling the whole set of interaction between Actors. Because each Actor is a single-threaded structure, we can discount the whole aspect of thread-contention, thread-race and resource-locking and focus on **what must happen when a message arrives** !

------------------------------------------------------------------------------------
