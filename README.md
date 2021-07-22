# Building a Reliable Data Transfer protocol
Here we implement a simple reliable data transfer protocol, which is similar
to how TCP implements its reliable data transfer. We start by implementing
a stop-and-wait, alternating-bit protocol. Then, we extend this to a simple
Go-Back-N (GBN) protocol, where the sender can now pipeline multiple packets
into the network.

## Alternating-Bit Protocol
1. Implementing over a reliable data channel (no loss or bit errors)
2. A protocol that can handle bit errors
3. A protocol that handles bit errors & packet loss
4. Using duplicate ACKs as feedback (get rid of NAKs)

## Go-Back-N Protocol
[TODO]

The network here is a simulated network. Boilerplate and simulation code, as
well as the problem itself, is on the open web and provided by Kurose & Ross.
Adapted from Kurose & Ross
