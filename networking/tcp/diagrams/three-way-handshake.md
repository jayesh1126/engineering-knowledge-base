```mermaid
sequenceDiagram
Client->>Server: SYN(seq=x)
Server->>Client: SYN-ACK(seq=y, ack=x+1)
Client->>Server: ACK(ack=y+1)
```