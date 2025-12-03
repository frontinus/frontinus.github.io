---
layout: post
title:  "CTF Writeup: Cross-Key Reflection Attack with ProVerif"
date:   2025-12-02 22:24:46 -0600
categories: CTF 
tags : ['Reflection Attack', 'CTF', 'Crypto', 'Diffie-Hellman', 'ProVerif']
author: "Francesco Abate"
---

In this post, I'll walk through the solution to a recent CTF challenge involving a custom authentication protocol. We'll explore how a **Reflection Attack** can be used to bypass authentication without knowing the private key.

## The Challenge

We are presented with a server implementing a challenge-response authentication protocol. The server can act as both a **Challenger** (verifying a client) and a **Responder** (proving its identity).

### The Protocol

The protocol is based on discrete logarithms and shared secrets, similar to Diffie-Hellman but with a twist.

1.  **Public Key Exchange**:
    *   Server sends its public key $B = g^b$.
    *   Client sends its public key $A = g^a$.
2.  **Shared Secret**:
    *   Both parties compute $k = A^b = B^a = g^{ab}$.
    *   The shared secret is $s = \text{hash}(k)$.
3.  **Authentication**:
    *   **Challenger** sends a challenge $C = \text{hash}(\text{hash}(s))$.
    *   **Responder** must reply with $R = \text{hash}(s)$.

### The Vulnerability

The vulnerability lies in the fact that the server uses the **same private key** $b$ for both its Challenger and Responder roles. This allows us to perform a **Cross-Key Reflection Attack**.

If we open two parallel sessions with the server:
1.  **Session 1**: We act as the Client, Server is Challenger.
2.  **Session 2**: We act as the Client, Server is Responder.

We can trick the server into solving its own challenge!

## The Exploit Strategy

Here is the step-by-step attack:

1.  **Connect to Session 1**: Receive Server's public key $B_1$.
2.  **Connect to Session 2**: Receive Server's public key $B_2$ (should be same as $B_1$).
3.  **Cross-Key Exchange**:
    *   Send $B_2$ as our public key to Session 1. So $A_1 = B_2$.
    *   Send $B_1$ as our public key to Session 2. So $A_2 = B_1$.
    *   Now, both sessions compute the *same* shared secret: $k = B^b = (g^b)^b = g^{b^2}$.
4.  **Wait for Challenge**:
    *   Session 1 (Challenger) sends us a challenge $C_1 = \text{hash}(\text{hash}(s))$.
5.  **Reflect Challenge**:
    *   We forward $C_1$ to Session 2 (Responder).
    *   Session 2 verifies $C_1$ (since it has the same $s$) and replies with $R_2 = \text{hash}(s)$.
6.  **Solve Challenge**:
    *   We take $R_2$ from Session 2 and send it back to Session 1.
    *   Session 1 verifies $R_2$ and gives us the flag!

## The Code

Here is the Python script that implements this attack:

```python
#!/usr/bin/env python3
"""
Cross-Key Reflection Attack (Gentry-Halevi)
"""

import socket
import time
import re
import sys

HOST = 'svtctf.m0lecon.it'
PORT = 13418

def extract_number(text, prefix):
    pattern = prefix + r'\s*(\d+)'
    match = re.search(pattern, text)
    if match:
        return int(match.group(1))
    return None

def recv_until(sock, marker, timeout=20):
    sock.settimeout(timeout)
    data = b""
    try:
        while marker.encode() not in data:
            chunk = sock.recv(4096)
            if not chunk:
                break
            data += chunk
    except socket.timeout:
        pass
    return data.decode(errors='ignore')

def solve():
    print("[+] CROSS-KEY REFLECTION ATTACK")
    
    # --- Session 1 ---
    s1 = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s1.connect((HOST, PORT))
    
    # Read B1
    data1 = recv_until(s1, "Give me your public key:")
    B1 = extract_number(data1, "Here is my public key:")
    
    # --- Session 2 ---
    s2 = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s2.connect((HOST, PORT))
    
    # Read B2
    data2 = recv_until(s2, "Give me your public key:")
    B2 = extract_number(data2, "Here is my public key:")
    
    # --- Cross-Key Exchange ---
    # Send A1 = B2 to S1
    s1.sendall(str(B2).encode() + b"\n")
    
    # Wait 10s so S1 starts its timer earlier than S2
    time.sleep(10)
    
    # Send A2 = B1 to S2
    s2.sendall(str(B1).encode() + b"\n")
    
    # --- Key Confirmation Relay ---
    print("\n[1] Waiting for S1 to timeout and send Challenge C1...")
    
    data1 = ""
    while "Here is your challenge:" not in data1:
        try:
            chunk = s1.recv(4096)
            if not chunk: break
            data1 += chunk.decode(errors='ignore')
        except socket.timeout: continue
            
    # Extract C1
    match = re.search(r'Here is your challenge:\s*([0-9a-f]{64})', data1)
    C1 = match.group(1)
    print(f"[1] Got Challenge C1: {C1}")
    
    # --- Relay C1 to S2 ---
    print(f"[2] Sending C1 to S2...")
    s2.sendall(C1.encode() + b"\n")
    
    # --- Receive Response R2 from S2 ---
    s2.settimeout(5)
    data2 = ""
    while "\n" not in data2:
        try:
            chunk = s2.recv(4096)
            if not chunk: break
            data2 += chunk.decode(errors='ignore')
        except socket.timeout: break
            
    match = re.search(r'([0-9a-f]{64})', data2)
    R2 = match.group(1) if match else data2.strip()
    print(f"[2] Got Response R2: {R2}")
    
    # --- Relay R2 to S1 ---
    print(f"\n[1] Sending R2 to S1...")
    s1.sendall(R2.encode() + b"\n")
    
    # --- Get Flag ---
    flag_data = s1.recv(4096).decode(errors='ignore')
    print(f"\n[SUCCESS]\n{flag_data}")
    
    s1.close()
    s2.close()

if __name__ == "__main__":
    solve()
```

## ProVerif Model

We also verified this vulnerability using **ProVerif**, a formal verification tool. The model defines the Challenger and Responder processes and shows that the attacker can indeed derive the `flag`.

```proverif
(* ProVerif Model for CTF Challenge *)

free c: channel.
type G.
type exponent.

(* The Flag *)
free flag: bitstring [private].

(* Constants *)
const g: G.

(* Functions *)
fun exp(G, exponent): G.
fun hash(bitstring): bitstring.
fun G_to_bytes(G): bitstring.

(* Equations *)
equation forall x: exponent, y: exponent; exp(exp(g, x), y) = exp(exp(g, y), x).

(* Server Private Key *)
free b: exponent [private].

(* ... Processes omitted for brevity ... *)

(* Query *)
query attacker(flag).
```

Running this model confirms `RESULT attacker(flag) is true`, validating our attack strategy.

## Conclusion

This challenge demonstrates the importance of using different keys for different roles or sessions, and how formal verification can help identify such logical flaws in protocols.
