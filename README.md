# minimal-state-tcp-syn-cookie-proxy

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                [ CLIENT ]                                    │
└──────────────────────────────────────────────────────────────────────────────┘
               │
               │  ① Sends TCP SYN to connect to target service
               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                [ XDP HOOK ]                                  │
│                         (eBPF program on ingress NIC)                        │
└──────────────────────────────────────────────────────────────────────────────┘
               │
               │ ② Intercepts TCP SYN packet
               │   - Parses Ethernet, IP, TCP headers
               │   - Verifies it's a SYN without ACK
               │   - Generates SYN cookie: hash(client IP, port, timestamp...)
               │   - Crafts SYN-ACK packet:
               │        * seq = cookie
               │        * ACK = client_seq + 1
               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          [ SYN-ACK with cookie SEQ ]                         │
│                             sent back to client                              │
└──────────────────────────────────────────────────────────────────────────────┘
               ▲
               │ ③ Client receives SYN-ACK, sends ACK
               │
               │
               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                [ XDP HOOK ]                                  │
│                         (Intercepts client's ACK)                            │
└──────────────────────────────────────────────────────────────────────────────┘
               │
               │ ④ Intercepts TCP ACK packet
               │   - Parses ACK number
               │   - Validates cookie from (ACK - 1)
               │   - If valid:
               │        → passes connection to userspace
               │        → optionally stores entry in BPF map for tracking
               │   - If invalid:
               │        → XDP_DROP
               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          [ USERSpace HANDLER/DAEMON ]                        │
│                         (via AF_XDP, perf ring, or BPF map)                  │
└──────────────────────────────────────────────────────────────────────────────┘
               │
               │ ⑤ Userspace receives connection metadata:
               │    - Client IP, port, validated seq number
               │
               │ ⑥ Establishes TCP connection to the real backend:
               │    - Connects to real server (e.g., backend:80)
               │
               │ ⑦ Optionally performs 3-way handshake with backend
               │
               │ ⑧ Injects client ACK to complete handshake bridging
               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                       [ BACKEND / REMOTE SERVER ]                            │
└──────────────────────────────────────────────────────────────────────────────┘
               ▲
               │
               │ ⑨ Full TCP connection is now established
               │    - Userspace forwards packets both ways
               │    - Can use zero-copy techniques (e.g., splice, io_uring)
               │
               ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        [ Client communicates with backend ]                  │
│                  (Through the userspace proxy or relay)                      │
└──────────────────────────────────────────────────────────────────────────────┘
```
