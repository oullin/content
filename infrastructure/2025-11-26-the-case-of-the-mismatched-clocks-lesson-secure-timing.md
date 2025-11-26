![cartoon-illustration-bank-vault-metallic-iron-safe-door_1441-1915](https://github.com/user-attachments/assets/eccc7db1-70fe-42da-97e6-ff80391dd9c5)


# The Case of the Mismatched Clocks: A Lesson in Secure Timing

Building secure software is a lot like building a bank vault. You need thick walls, complex locks, and stringent procedures for who gets in and when.

At [Oullin](https://oullin.io/), I take the security of your data incredibly seriously. I use advanced techniques to ensure that every single message the app sends to the servers is authentic and hasn't been tampered with.

Recently, however, I ran into a tricky little bug. It wasn't a security hole—the vault was actually working _too_ well. It started rejecting legitimate requests from the app itself.

I want to share what happened in plain English because it offers a fascinating look at how precise digital security needs to be.

> 🔧 API Changes: [https://github.com/oullin/api/pull/171](https://github.com/oullin/api/pull/171)
> 
> 📡 Web Changes: [https://github.com/oullin/web/pull/178](https://github.com/oullin/web/pull/178)


## The "Digital Check" Analogy

To understand the bug, imagine you are writing a physical bank check to pay a bill. To make it valid, you need three essential things:

1. **The Amount** (the data).
2. **The Date** you wrote it (the timestamp).
3. **Your Signature** at the bottom proving you authorised it.
    
When the bank receives the check, they examine all three pieces. Your signature is valid _only_ for that specific amount on that particular date.

```text
graph TD
    A[The Check]
    B[Date: Tuesday]
    C[Amount: $100]
    D[Signature: Valid only for Tuesday + $100]
    
    A --> B
    A --> C
    A --> D
    
    B -.-> D
    C -.-> D
```

> If you wrote a check for $100 dated **Tuesday**, signed it, but then crossed out "Tuesday" and wrote **Wednesday** just before handing it over, the bank teller would reject it. Why? Because your signature was promising that the check was good on Tuesday, not Wednesday. The details no longer match up.

## Where I Got Tripped Up

In the digital world, the Oullin app does something very similar every time it talks to the server.

1. It checks the exact time.
2. It creates a "digital seal" (signature) locking that time to the data.
3. It sends the message.
    

Here is where my code got a little too clever for its own good. In an effort to be as precise as possible, the app was preparing the signature, but just milliseconds before sending the message, it would think: _"Wait, a fraction of a second has passed. Let me refresh the time so it's super accurate."_

It updated the timestamp on the **outside** of the message, but it didn't update the signature on the **inside**.

**The result:** The server received a message saying, "This was sent at **12:00:05**," but the signature inside was saying, "I only authorise this if it is **12:00:04**."


### The Broken Flow

```text
sequenceDiagram
    participant App
    participant Server
    
    Note over App: 1. Get Time: 12:00:04
    Note over App: 2. Create Signature for 12:00:04
    Note over App: ... slight delay ...
    Note over App: 3. Update Time to 12:00:05
    App->>Server: Send Message (Header: 12:00:05 | Sign: 12:00:04)
    Note over Server: CHECK: Does Header match Signature?
    Server-->>App: REJECTED (Mismatch)
```

The server, doing precisely what it is supposed to do to prevent fraud, saw the mismatch and slammed the door shut.


## The Fix: Locking It Down

Once I realised the mistake—that the app was acting like the person changing the date on a signed check—the fix was straightforward.

I reorganised the code to follow a strict sequence: define the exact time _once_, generate the signature based on that time, and then send the message. I removed the step that tried to update the time at the last second.

Now, the time on the outside of the message always perfectly matches the time locked into the signature on the inside.


## The Fixed Flow

```text
sequenceDiagram
    participant App
    participant Server
    
    Note over App: 1. Lock Time: 12:00:04
    Note over App: 2. Create Signature for 12:00:04
    Note over App: 3. Set Header to 12:00:04
    App->>Server: Send Message (Header: 12:00:04 | Sign: 12:00:04)
    Note over Server: CHECK: Does Header match Signature?
    Server-->>App: APPROVED (Match!)
```


## Why This Matters

Bugs like this can be frustrating because they temporarily break things, but they are also reassuring. They prove that the security measures are working. The Oullin system is designed to reject anything that looks even slightly suspicious—even if it’s just the app being a little disorganised with its timing.

I’ve deployed the fix, the clocks are synchronised, and everything is running smoothly again.

Thanks for your patience while we synchronised our watches!





