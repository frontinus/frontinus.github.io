---
layout: post
title:  "Vote for Pedro"
date:   2025-04-07 22:20:41 +0100
categories: ['CTF', 'cryptography']
tags : ['CTF', 'cryptography', 'Cryptohack', 'Cybersecurity']
author : "Francesco Abate"
---

Vote for Pedro is a RSA challenge from [CryptoHack](https://cryptohack.org/challenges/rsa/)

## 🗳️ CTF Writeup: Vote for Pedro

### If you want my flag, you better vote for Pedro! Can you sign your vote to the server as Alice?

---

## 🔍 Challenge Overview

You're given access to:

- A Python server script (`13375.py`)
- Alice’s RSA public key (`alice.key`)
- A prompt telling you to “sign your vote as Alice” and submit it to a socket server.

### 🧠 Goal

Forge a vote for Pedro and submit a **valid RSA signature** as if it came from Alice.

---

## 📄 Challenge Files

### `alice.key`

```text
N = 22266616657574989868109324252160663470925207690694094953312891282341426880506924648525181014287214350136557941201445475540830225059514652125310445352175047408966028497316806142156338927162621004774769949534239479839334209147097793526879762417526445739552772039876568156469224491682030314994880247983332964121759307658270083947005466578077153185206199759569902810832114058818478518470715726064960617482910172035743003538122402440142861494899725720505181663738931151677884218457824676140190841393217857683627886497104915390385283364971133316672332846071665082777884028170668140862010444247560019193505999704028222347577

e = 3
```

- Standard RSA public key

- Exponent e = 3 (very small)

- Very large modulus N

### `13375.py`

```python
from Crypto.Util.number import bytes_to_long, long_to_bytes
from utils import listener
```

Imports essential utilities: bytes_to_long and long_to_bytes help convert between integers and byte strings, crucial for RSA operations. listener is a helper provided by Cryptohack to handle incoming socket connections for CTF interactions.

```python
FLAG = "crypto{????????????????????}"
```

Placeholder for the flag that will be returned upon successful submission.

```python
class Challenge():
    def __init__(self):
        self.before_input = "Place your vote. Pedro offers a reward to anyone who votes for him!\n"
```

Defines the challenge class with an introductory message shown to the player before they send input.

```python
    def challenge(self, your_input):
        if 'option' not in your_input:
            return {"error": "You must send an option to this server"}
```

Main method handling user input. If the input doesn't include an "option" key, it immediately responds with an error.

```python
        elif your_input['option'] == 'vote':
            vote = int(your_input['vote'], 16)
```

Handles the "vote" option. Expects the input value to be a hex string representing an integer — this is supposed to be the signature forged by the attacker.

```python
            verified_vote = long_to_bytes(pow(vote, ALICE_E, ALICE_N))
```

Performs RSA signature verification by computing vote^e mod N, then converts the resulting integer to bytes.

```python
            vote = verified_vote.split(b'\x00')[-1]
```

Removes padding (very simplistically) by splitting the decrypted message at null bytes and taking the last chunk, assuming that is the meaningful message.

```python
            if vote == b'VOTE FOR PEDRO':
                return {"flag": FLAG}
            else:
                return {"error": "You should have voted for Pedro"}
```

Checks if the message matches the expected vote content. If so, it reveals the flag; otherwise, it returns an error.

```python
        else:
            return {"error": "Invalid option"}
```

Fallback if the provided option is not recognized (i.e., not "vote").

```python
import builtins; builtins.Challenge = Challenge
listener.start_server(port=13375)
```

Registers the challenge class globally so the server can find it, and starts the socket server on port 13375, waiting for connections from challengers.

## Solution

This script is an exploit for the "Vote for Pedro" CTF challenge. It leverages RSA’s low public exponent (e=3) vulnerability by using a pre-computed signature to forge a valid vote. Below is a block-by-block explanation of the code.

```python
from pwn import *
from json import *
from Crypto.Util.number import long_to_bytes, bytes_to_long
```

Imports:

- pwn provides networking functionality (like connecting to a remote server).

- json is used for serializing/deserializing the data exchanged with the server.

- Crypto.Util.number offers conversion utilities for handling RSA numbers.

```python
def send(hsh):
    return r.sendline(dumps(hsh))
```

Helper Function:

- The send function takes a Python dictionary (hsh), converts it to a JSON-formatted string using dumps, and sends it to the remote server with sendline.

```python
ALICE_N = 22266616657574989868109324252160663470925207690694094953312891282341426880506924648525181014287214350136557941201445475540830225059514652125310445352175047408966028497316806142156338927162621004774769949534239479839334209147097793526879762417526445739552772039876568156469224491682030314994880247983332964121759307658270083947005466578077153185206199759569902810832114058818478518470715726064960617482910172035743003538122402440142861494899725720505181663738931151677884218457824676140190841393217857683627886497104915390385283364971133316672332846071665082777884028170668140862010444247560019193505999704028222347577
ALICE_E = 3
```

RSA Key Constants:

- ALICE_N and ALICE_E define Alice’s RSA public key. Here, e is set to 3, making the RSA system vulnerable to cube-root attacks when used without proper padding.

```python
r = remote('socket.cryptohack.org', 13375)
print(r.recv())
```

Remote Connection:

- Establishes a connection to the challenge server at socket.cryptohack.org on port 13375 using remote from the pwntools library.

- Immediately prints the server’s welcome message.

```python
vote = 855520592299350692515886317752220783
```

Pre-computed Signature:

- The variable vote holds the forged signature. This number is the cube root of the padded message corresponding to "VOTE FOR PEDRO", calculated such that raising it to the power of 3 modulo ALICE_N reproduces the correctly padded vote.

```python
option = {
    'option': 'vote',
    'vote': hex(vote)
}
send(option)
```

Building and Sending the Payload:

- A JSON object is constructed with two keys:

  - 'vote' contains the hexadecimal representation of the forged signature.

  - The send helper function is used to transmit this payload to the server.

```python
get = loads(r.recv())
flag = get['flag']
print(flag)
```

Receives the server’s response, decodes the JSON string into a Python dictionary, extracts the flag from the response, and prints it. If the forged signature was valid, the server responds with the flag.

Overall, the script automates the process of connecting to the server, sending a precomputed forged RSA signature (that takes advantage of the low exponent and lack of padding), and printing the flag received as a result.

In the end you obtain the flag :

```python
crypto{y0ur_v0t3_i5_my_v0t3}
```
