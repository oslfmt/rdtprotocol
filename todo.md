# TODO
1. Why do we need acknums? review seqnums use in textbook
2. Figure out acknums & seqnums (packets being delivered to receiver in correct order!)

## A (stop-and-wait) protocol that can handle bit errors
Two core assumptions:
1. Since this is stop-and-wait, the sender only send one packet at a time into the network, and must receive
some sort of feedback (ACK or NAK or garbled) before it can take another action. What this means is that
ACK/NAKs do NOT have to have acknums, since the sender can assume that whatever response it receives is
generated in response to the packet *it just sent*.
2. We assume no packet loss. This means all packets sent *will* be received. The only "bug" that can occur
is bit corruption.

### Sender states:
- Wait for call 0 from above
    1. Rcvd pkt 0 => send pkt 0
- Wait for ACK/NAK (for pkt 0)
    1. Rcvd ACK => send next pkt
    2. Rcvd NAK => resend pkt 0
    3. Rcvd garbled => resend pkt 0
- Wait for call 1 from above
    1. Rcvd pkt 1 => send pkt 1
- Wait for ACK/NAK (for pkt 1)
    1. Rcvd ACK => send next pkt
    2. Rcvd NAK => resend pkt 1
    3. Rcvd garbled => resend pkt 1

### Receiver states:
- Wait for pkt 0
    1. Rcvd pkt 0 => pass to layer5, send ACK (implicitly for 0)
    2. Rcvd pkt 1 => discard, send ACK (implicitly for 1)
    3. Rcvd garbled => send NAK
- Wait for pkt 1
    1. Rcvd pkt 1 => pass to layer5, send ACK
    2. Rcvd pkt 0 => discard, send ACK
    3. Rcvd garbled => send NAK

- note that the actions when waiting for packet 0/1 are the same, only the handling of packet
sequence numbers differs

## A (stop-and-wait) protocol that can handle packet loss
- Pkts are still sent one at a time, but in addition to bit corruption, packets can be lost
- This means that pkts with data can be lost AND response pkts can also be lost
- Note: we still do not need ACK/NAK numbers here because even though pkts can now be lost, still, only 1 pkt is
being sent into the network at a time. In the case that a data pkt is lost, NO response will be sent, and so acknums
do not matter here. The sender simply resends that last pkt. If the ack/nak is lost, then NO response will be sent
(obviously) and the sender action is the same. At the receiver side, duplicate pkts are handled by seqnums.
Acknums are necessary for the sender to know whether a specific pkt it sent was rcvd or not. In this case, since only 1 pkt
is sent, obviously any ACK/NAK pkts received indicate that the last sent pkt was rcvd (whether duplicate or not) and the
sender can move on to sending the next pkt.
- To reiterate--acknums are used by the sender to know whether a specific packet it sent was received. If only 1 pkt is sent
at a time, the ACK/NAK pkt itself serves as the acknowledgement, and can implicitly thought to have an acknum of 1--but
hopefully it can be seen that the 1 is duplicate info.

### Sender states
- Wait for call 0 from above
    1. Got pkt 0 => send pkt 0; start timer
    2. rcvd pkt => drop it
- Wait for ACK/NAK (for 0)
    1. Rcvd garbled OR rcvd NAK => do nothing (wait for timeout)
        ---->>> in the prev. implementation we would resend. Here we don't, since we're reliant on the timer. So if
        the timer times out then we resend 2x. We could however resend immediately and stop timer, cutting delay
    2. Timeout => resend pkt 0; restart timer
    3. Rcvd ACK => stop timer; send next pkt
- Wait for call 1 from above
    1. Got pkt 1 => send pkt 1
    2. rcvd pkt => drop it
        --->>> a pkt may be received here due to premature timeout, which causes a 2nd pkt to be sent. Both may arrive
        and 2 responses might be sent back. One can be discarded (whatever it is) since we already got the ack for the last
        pkt
- Wait for ACK/NAK (for 1)
    1. Rcvd garbled OR rcvd NAK => do nothing
    2. Timeout => resend pkt; restart timer
    3. Rcvd ACK => stop timer; send next pkt

### Receiver states
- Wait for pkt 0
    1. Rcvd pkt 0 => deliver; send ACK
    2. Rcvd pkt 1 => drop; send ACK
    3. Rcvd garbled => drop; send NAK
- Wait for pkt 1
    1. Rcvd pkt 1 => deliver; send ACK
    2. Rcvd pkt 0 => drop; send ACK
    3. Rcvd garbled => drop; send NAK

Note: the receiver FSM is exactly the same as the previous receiver FSM. This is because if a pkt is lost, it is
entirely up to the sender to handle it. The sender has timers to detect pkt loss, and recovers via retransmission. This
greatly simplifies the receiver design (and load).
To convince: if a data pkt is lost, the receiver will never see it and thus never send a response. The sender thus never
sees a response, and thus retransmits.
If a response pkt is lost, the sender *also* never sees a response, and thus retransmits. This causes a duplicate pkt,
but the receiver can detect this with seqnums and handle it.

## Getting Rid of NAKs
- In a simple stop-and-wait protocol, we can easily eliminate NAKs
- Consider this: a NAK is only sent when a received packet has been corrupted
- This indicates that the packet was not correctly received, and thus signals to the sender that it needs to retransmit
the last sent packet
- Instead of sending a NAK, however, we could just send another ACK, and somehow have this ACK function as an implicit NAK

- so lets say sender sends pkt 0. rcvr sends back an ACK. sender sends pkt 1. it is corrupted, so rcvr sends back an ACK.
sender can't tell the difference, so it sends 0, and now we have a problem.
- now lets say sender sends pkt 0. it is corrupted. rcvr sends back an ACK (as that is the only msg it can send). sender
views this as green light to send pkt 1. again, we have the same problem.

- Clearly, we need a way to differentiate the ACKs. One way to do this is by adding "acknums" which functionally serve the
the same purpose as seqnums, ie, they are the seqnums for acknowledgments.
- In stop-and-wait, we will only have acknums of 0 and 1, since those are the only possible seqnums. An acknum of 0 ACKs
a pkt with seqnum=0, and acknum=1 acks a pkt with seqnum=1

- Now, lets say a sender sends pkt 0, and it's corrupted. Normally, the rcvr sends a NAK. Here, it sends an ACK with acknum=1,
thus implicitly telling the sender: "I did not rcv pkt w/seqnum=0, I am acking for a pkt with seqnum=1 (which in this case
is nonexistent since this is the first msg sent)
- Now, lets say sender sends pkt 0. Rcvr sends ACK with acknum=0. This tells sender the pkt was correctly received. So sender
moves on to send pkt 1. This is corrupted, so rcvr sends ACK with acknum=0. What this tells the sender is that: "I did not
correctly rcv pkt1. Instead, the last correctly rcvd pkt I got had seqnum=0, so I am acking for that pkt". This then
prompts the sender to retransmit pkt1.

- Thus, by introducing acknums for acknowledgements, we can now entirely get rid of NAKs.
- This changes a few things in the implementation:
    - on the sender side, we now need to check explicitly for the acknum.
    - on the receiver side, we now need to add acknums to the acknowledgement pkts

### Sender states
- Wait for call 0 from above
    1. Got pkt 0 => send pkt 0; start timer
    2. rcvd pkt => drop it
- Wait for ACK w/acknum=0
    1. Rcvd garbled OR rcvd ACK w/acknum=1 => wait for timeout
    2. Timeout => resend pkt 0; restart timer
    3. Rcvd ACK w/acknum=0 => stop timer; send next pkt
- Wait for call 1 from above
    1. Got pkt 1 => send pkt 1
    2. rcvd pkt => drop it
- Wait for ACK w/acknum=1
    1. Rcvd garbled OR rcvd ACK w/acknum=0 => wait for timeout
    2. Timeout => resend pkt; restart timer
    3. Rcvd ACK w/acknum=1 => stop timer; send next pkt

### Receiver states
- Wait for pkt 0
    1. Rcvd pkt 0 => deliver; send ACK w/acknum=0
    2. Rcvd pkt 1 => drop; send ACK w/acknum=1
    3. Rcvd garbled => drop; send ACK w/acknum=1
- Wait for pkt 1
    1. Rcvd pkt 1 => deliver; send ACK w/acknum=1
    2. Rcvd pkt 0 => drop; send ACK w/acknum=0
    3. Rcvd garbled => drop; send ACK w/acknum=0

Note: see that none of the cases changed. all that we did was replace NAKs with ACKs that now have acknums

## Go-Back-N (GBN) Protocol Implementation
- Unidirectional transfer: data only transferred from A to B
- WindowSize=8

Basic Idea: This is a pipelined protocol. Whereas in the previous stop-and-wait protocol, the sender only sent one msg
into the network at a time. In order to send the next pkt, the sender MUST receive some sort of feedback, or a timeout
occurs. The problem with this protocol is that it is extremely slow, since the sender will be idle most of the time, not
utilizing the full bandwidth available to send pkts.

In order to utilize more of the bandwidth, the obvious solution is for the sender to send multiple packets into the
network, ie, pipelining them, WITHOUT having to wait for acknowledgements. This allows more data to be sent at once, and is
similar to batching. This is essentially what GBN does--it allows the sending of a batch of pkts (up to a limit) into the
network at once, without waiting for acknowledgements. In "go-back-N", the N is the limit of packets that can be sent
into the network at any given time.

This introduces some complications:
- Since we are sending batches of pkts at once now, we're going to need a larger range of seqnums, in order to know which
pkts have been acked, as well as for the receiver to differentiate pkts. If we only had 0 and 1, and could send 10 pkts
at once, we would have 5 0s and 5 1s, and it would make it more complicated for the receiver to know the difference between
pkts.
- Secondly, since we can send more pkts into the network at once, this means that any one of these pkts may get lost
or corrupted, and thus, we may need to possibly retransmit any of the n pkts we send. Thus, for transmitting n pkts at once,
we need to have a buffer of at least size n to properly retransmit
- Also, since any of these pkts can now be lost, each needs a timer to track timeouts. However, as we will see later, GBN
takes a more simplified approach with timers.

Note that since we are now sending n pkts at once, we need a buffer of at least size n. In the stop-and-wait protocol,
when sending one pkt, we needed a buffer of size 1. Thus, stop-and-wait can be viewed as a "special case" of the GBN
protocol, where n=1.

To reiterate, GBN allows the sender to transmit many packets without waiting for acknowledgements. However, GBN constrains
this number to no more than N. Thus, at any given time, there has to be N or less transmitted but unack'd pkts.
This number N is known as the window size.

The first important fact to establish is the range of sequence numbers. Earlier, we said the range can be increased from just
0 and 1. So, what should this range be set to exactly? In practice, I don't think it really matters, since used sequence
numbers can later be used, since we use modulo arithmetic to wrap back around the seqnum space. At minimum, it needs to be
larger than N. In practice, it can be set to 2^k, where k is the number of bits in the seqnum field. Thus, the range of seqnums
becomes [0, 2^k-1].

The window (possible pkts to be sent) is a slice of this sequence space. The sender has a "view" of this sequence space--think
of it as a linear array, where the indexes are the seqnums. The window starts at 0, and goes up to N. The window has 2 bounds,
at the lower end, we call it the *base*: this is where the window begins. Since the window represents the total possible number
of pkts to be pipelined (not necessarily the number that are transmitted), we have an index into the window space, called
*nextseqnum*, which is the seqnum of the next pkt to be sent.

Now, at any given time, there are broadly speaking 4 sections of the sequence number space:
1. [0, base-1]: these are the seqnums for pkts that have been sent and acked
2. [base, nextseqnum-1]: seqnums for pkts that have been sent but not yet acked
3. [nextseqnum, base+N-1]: seqnums for pkts that can be sent (immediately) but have not been sent yet
4. >=base+N: seqnums that are unusable (until base has been acked)

Thus, as pkts are acked, the window essentially slides forward, thus allowing more pkts to be sent. However, the window
keeps a hard maximum on the number ot transmitted but unack'd packets. One final thing to address is that since there is a
maximum seqnum, what happens if we have more packets than seqnums? Well, this is not an issue since arithmetic is done with
mod 2^k. Thus, the seqnums wrap around to the beginning of the space when the max is reached.

### Sender Events
GBN can be viewed as an event-based protocol. The sender is essentially always in a "wait" state, listening to certain events
to occur. When an event occurs, the sender performs an action. This is in contrast to the previous protocols that had
various states. There are 3 main events:

- Wait for call from above
    1. if window is full, drop; else, send pkt
- Wait for acknowledgement
    1. On receipt of an (uncorrupted) ACK, check seqnum
- Timeout
    1. resend all transmitted but unack'd pkts--at maximum, this can be N pkts, which is why protocol is called "go-back-N"

- Additionally the sender must keep track of the following:
    - A buffer (for retransmissions, and to hold pkts)
    - base: the window base
    - nextseqnum: the seqnum to use for the next sent pkt
    - N: the windowsize (is 8 in this case)

### Receiver Events
The receiver is also in a wait state and waits for events to occur:

- Uncorrupted, in-sequence pkt received
    1. deliver msg; send ACK with proper acknum; increment expectedseqnum
- All other events
    1. drop pkt; send ACK with acknum=last_rcvd_pkt_acknum