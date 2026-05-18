# TCP

## What is TCP?

Transmission Control Protocol is a connection-oriented protocol that provides reliable, ordered, and error-checked delivery of data between applications on an IP network.
Operates at Layer 4 (transport layer) of the OSI model, acts as a shield against unreliable nature of the internet such as lost, corrupted, or out-of-order packets.
HTTP, SMTP (email), SSH, and WebSocket all rely on TCP to ensure data integrity.

# Core Concepts

## Three-Way Handshake - Establishing connection

1. LISTEN: The server is in a LISTEN state, waiting for a connection on a specific port.
2. SYN (Synchronize): The client sends a SYN packet with its Initial Sequence Number (ISN). This tells the server the starting point for counting bytes in the client-to-server direction.
3. SYN-ACK: The server acknowledges the client's SYN by adding one to the client's ISN (the "ghost byte"). Simultaneously, it sends its own ISN to the client.
4. ACK (Acknowledgment): The client acknowledges the server's SYN by adding one to the server's ISN. The connection is now ESTABLISHED and ready for duplex data flow.

# Sequence Number

Using Wireshark, we can check client and server sequence and acknowledgment numbers. It is how we determine what is happening deep down in TCP.

    Byte Tracking: Every byte of data sent is assigned a sequence number.
    Relative vs. Raw: Wireshark often displays Relative Sequence Numbers (starting at 0) because raw ISNs are large, 32-bit random numbers that are difficult for humans to track.
    Increments: If a client sends a 501-byte payload starting at sequence 1, the next packet it sends will start at sequence 502. The server acknowledges this by sending back an ACK number of 502, meaning "I received up to 501 and am ready for 502".

## Retransmission

TCP ensures reliability through a "positive acknowledgment with retransmission" mechanism.

    Timers: When a sender transmits data, it holds a copy in a send buffer. If a timer expires before an ACK is received, the sender assumes the packet was lost and retransmits it.
    Checksums: Every TCP segment contains a checksum. If the receiver calculates a different sum than what is in the header, the data is considered corrupted and discarded, eventually triggering a retransmission.

...

## Flow Control

This mechanism prevents a fast sender from overwhelming a slow receiver.

    Receive Window: The receiver advertises its available buffer space (the Window field in the header). The sender is not allowed to send more data than this window size without receiving an ACK.
    Sliding Window: As the application reads data from the buffer, the window "slides," opening up more space for the sender.
    Window Scaling: Because the original TCP window field is only 16 bits (max 65,535 bytes), modern high-speed networks use a Window Scale option negotiated during the handshake to multiply this value, allowing for buffers up to 1GB.

## TCP Socket and Queues

The Socket Abstraction: A TCP socket is an endpoint identified by the pair of an IP address and a port number. In programming, this is often exposed as a file descriptor, allowing you to "read" and "write" to a network connection just like a file on a disk.
The Listen Queue: When a server starts listening, it specifies a queue size for pending connections. If the server receives more simultaneous connection requests than this queue can handle, it will begin dropping them.
Kernel Buffers (Send-Q & Recv-Q): The operating system maintains buffers to hold data. The Recv-Q holds bytes received but not yet read by the application, while the Send-Q holds bytes waiting to be acknowledged by the remote host.

## Port Routing (Multiplexing)

Simultaneous Applications: TCP uses 16-bit source and destination ports to allow many network applications to run on a single machine simultaneously.
The 5-Tuple: Every connection is uniquely identified by five elements: the source IP, destination IP, source port, destination port, and the protocol (TCP/UDP).

## TCP Segment and Header Structure

A TCP Segment is the minimal unit of data, consisting of a header and a payload.

    Fixed vs. Variable Fields: The header has fixed-position fields (like ports and sequence numbers) for fast parsing, and a variable-length Options field.
    Data Offset: This 4-bit field tells the receiver the size of the header, which is necessary because the Options field can vary in size.
    Checksum: This field is used for data integrity; if the calculated sum of the segment’s words doesn't match the checksum, the segment is considered corrupted and is discarded.

## Advanced Flags

Beyond SYN, ACK, and FIN, several other flags manage the connection:

    PSH (Push): Tells the receiver to immediately pass the data to the application instead of waiting for the buffer to fill.
    RST (Reset): Used for abrupt or unrecoverable errors. It tells the receiver to immediately terminate the connection.
    URG (Urgent): Indicates that specific data in the payload should be prioritized for processing.

## 4-Way Handshake (Closing Connection)

TCP uses a graceful goodbye handshake to ensure both sides have finished sending data:

    FIN: Side A sends a FIN flag to indicate it wants to disconnect.
    ACK: Side B acknowledges the FIN.
    FIN: Side B sends its own FIN flag once it is finished sending its own data.
    ACK: Side A acknowledges Side B’s FIN, and the connection is closed.

## Congestion Control

While Flow Control protects the receiver, Congestion Control protects the network itself from overload.
TCP interprets packet loss as a sign of congestion and reduces its sending rate accordingly.

    Slow Start: TCP begins cautiously, increasing the congestion window exponentially until loss is detected.
    Congestion Avoidance (AIMD): After reaching a threshold, growth becomes linear to avoid overwhelming the network.
    Fast Retransmit: Multiple duplicate ACKs indicate likely packet loss, allowing retransmission before a timeout occurs.
    Backoff: When congestion is detected, TCP reduces throughput to stabilize the network.

Without congestion control, excessive retransmissions can lead to congestion collapse, where the network spends more time retransmitting packets than delivering useful data.

## TCP is a Byte Stream

TCP does not preserve application message boundaries.
It provides a continuous ordered stream of bytes rather than discrete packets/messages.

- one send() call may arrive in multiple recv() calls
- multiple sends may arrive in a single recv()

Applications must therefore implement their own framing protocols if message boundaries matter.
Protocols like HTTP, WebSocket, and gRPC all define framing rules on top of TCP.

## Blocking vs Non-Blocking Sockets

By default, TCP sockets are blocking:

- read() waits until data arrives
- accept() waits until a connection arrives

Non-blocking sockets return immediately instead of waiting to efficiently manage thousands of concurrent connections.
High-performance servers often use:

- epoll (Linux)
- kqueue (BSD/macOS)
- IOCP (Windows)

---

# Deep Dives

## TCP State Machine

TCP connections transition through specific states to manage the lifecycle of a session:

    ESTABLISHED: Data can flow bidirectionally.
    Graceful Shutdown (Four-Way Handshake).
    Abrupt Shutdown (RST): If a process crashes or a port is closed, a Reset (RST) flag is sent to immediately terminate the connection.

## Ephemeral Ports and TIME_WAIT

Ephemeral Ports: These are high-numbered, short-lived ports (usually >1023) automatically assigned by the operating system to client applications for a single connection.
Port Multiplexing: Ports allow a single IP address to host multiple applications simultaneously. The 5-tuple (Source IP, Source Port, Destination IP, Destination Port, Protocol) uniquely identifies each connection.

---

# Production Relevance

Troubleshooting: Network analysts use TCP health to isolate problems. If TCP segments show high retransmissions or out-of-order packets, the issue is likely the network; if TCP is "clean" but the application is slow, the issue is likely the server or code.
Modern Limitations: TCP headers have a finite amount of space for Options. This is one reason for the move toward QUIC, as TCP is reaching its limits for new feature extensions.

---

# Failure Cases

Congestion Collapse: In 1986, the internet nearly failed because systems kept retransmitting lost packets into an already clogged network, dropping speeds to 40 bps. Modern TCP uses Congestion Control to "back off" when loss is detected.
Zero Window: If a receiver's application is stuck and cannot clear its buffer, the window size drops to zero. This effectively halts the sender, resulting in a "stuck" application.
TCP Reset Attack: A security case where an intermediary (like a firewall) injects a spoofed RST packet to force-terminate a connection between two parties.

---

# Labs

## TCP Echo Server

Build a basic server using Berkeley Socket primitives in Go:

    socket(): Create the endpoint.
    bind(): Attach the socket to a port.
    listen(): Set up a queue for incoming requests.
    accept(): Wait for and accept a connection, returning a file descriptor that can be read from or written to like a file.

First step is use the net pakcage though.

## Packet Loss and Retransmission

To observe this, we will use Wireshark on an unreliable link.
We will look for TCP Retransmissions (highlighted in black/red by default), observing the Delta Time column to see the delay introduced by waiting for timers to expire.

---

# Observations

...

---

# Questions

...
