---
layout: post
title:  "Endless Emails"
date:   2025-04-12 22:20:41 +0100
categories: [CTF, cryptography]
---

Endless Emails is an RSA challenge from [CryptoHack](https://cryptohack.org/challenges/rsa/)

## 🗳️ CTF Writeup: Endless Emails

CTF Challenge Analysis: RSA Low Exponent Broadcast

## 🔍 Challenge Overview

You're given access to:

- A Python server script (`johan.py`)
- RSA combinations with modulus, exponent and ciphertext (`output.txt`)

### 🧠 Goal

This document analyzes the provided files (johan.py and output.txt) from the cryptography CTF challenge. The goal is to understand the setup and identify the intended vulnerability and solution path.

---

## 📄 Challenge Files

### `johan.py`

This Python script generated the challenge data found in output.txt.

```python
#!/usr/bin/env python3

from Crypto.Util.number import bytes_to_long, getPrime # Imports functions for number theory: converting bytes to integers and generating prime numbers.
from secret import messages # Imports a variable 'messages' from a file named secret.py. This likely contains the flag or plaintext messages to be encrypted.


def RSA_encrypt(message):
    m = bytes_to_long(message) # Converts the input message (bytes) into a large integer 'm'.
    p = getPrime(1024) # Generates a 1024-bit prime number 'p'.
    q = getPrime(1024) # Generates another 1024-bit prime number 'q'.
    N = p * q # Calculates the RSA modulus 'N' by multiplying p and q. N will be approximately 2048 bits long.
    e = 3 # Sets the public exponent 'e' to 3. This is a small, common value, often used in CTFs.
    c = pow(m, e, N) # Performs the RSA encryption: c = m^e mod N.
    return N, e, c # Returns the modulus N, public exponent e, and the ciphertext c.


for m in messages: # Loops through each message in the 'messages' list/tuple imported from secret.py.
    N, e, c = RSA_encrypt(m) # Encrypts the current message 'm'. Crucially, a *new* pair of primes (p, q) and thus a *new* modulus N are generated for *each* message.
    print(f"n = {N}") # Prints the generated modulus N.
    print(f"e = {e}") # Prints the public exponent e (which is always 3).
    print(f"c = {c}") # Prints the resulting ciphertext c.
```

Key Observations:

- Purpose: The script encrypts messages (likely including the flag) from a secret.py file using RSA.

- RSA Implementation: It follows the standard RSA encryption procedure.

- Key Generation: A new RSA key pair (specifically, new primes p, q and modulus N = p*q) is generated for each message it encrypts.

- Public Exponent: It uses a fixed, small public exponent e = 3 for all encryptions.

- Output: The script outputs the public modulus (n), the public exponent (e), and the ciphertext (c) for each encryption.

- Vulnerability Setup: The combination of encrypting the same message multiple times with different moduli (N) but the same small public exponent (e=3) directly sets up the conditions for Hastad's Broadcast Attack.

- Message Source: The messages are likely stored in the secret.py file, which is not provided. This file is crucial for the challenge as it contains the plaintexts that were encrypted.

### `output.txt`

The output.txt file contains the RSA modulus (n), public exponent (e), and ciphertext (c) for each message. The format is as follows:

```text
n = 22266616657574989868109324252160663470925207690694094953312891282341426880506924648525181014287214350136557941201445475540830225059514652125310445352175047408966028497316806142156338927162621004774769949534239479839334209147097793526879762417526445739552772039876568156469224491682030314994880247983332964121759307658270083947005466578077153185206199759569902810832114058818478518470715726064960617482910172035743003538122402440142861494899725720505181663738931151677884218457824676140190841393217857683627886497104915390385283364971133316672332846071665082777884028170668140862010444247560019193505999704028222347577
e = 3
c = 6965891612987861726975066977377253961837139691220763821370036576350605576485706330714192837336331493653283305241193883593410988132245791554283874785871849223291134571366093850082919285063130119121338290718389659761443563666214229749009468327825320914097376664888912663806925746474243439550004354390822079954583102082178617110721589392875875474288168921403550415531707419931040583019529612270482482718035497554779733578411057633524971870399893851589345476307695799567919550426417015815455141863703835142223300228230547255523815097431420381177861163863791690147876158039619438793849367921927840731088518955045807722225
```

Key Observations:

- Content: The file consists of multiple blocks, each containing an RSA modulus (n), the public exponent (e), and a ciphertext (c). There are 7 such blocks provided.

- Parameters:
  - The modulus n is different in each block.

  - The public exponent e is always 3.

  - The ciphertext c is the result of encrypting some message with the corresponding n and e=3.

### Solution

---

The core steps of the strategy are:

- Data Loading: Parse the output.txt file to extract all provided pairs of RSA moduli (n) and ciphertexts (c). The public exponent e=3 is known from the challenge description and the output.txt file itself.
   
- Combinations: Since the challenge might have encrypted multiple different messages, or the target flag might not correspond to the first three (n, c) pairs, the script iterates through all possible combinations of e=3 pairs from the loaded data. This ensures that the correct set of three encryptions corresponding to the same plaintext message m will eventually be processed.

- Chinese Remainder Theorem (CRT): For each combination of three (n, c) pairs (n_1, c_1), (n_2, c_2), (n_3, c_3), apply the Chinese Remainder Theorem. The goal is to find a single integer M that satisfies the system of congruences:
  - M ≡ c_1 (mod n_1)

  - M ≡ c_2 (mod n_2)

  - M ≡ c_3 (mod n_3) According to Hastad's attack, this combined value M will be congruent to m^3 modulo the product N = n_1 * n_2 * n_3.

- Integer Cube Root: Assuming m^3 < N (which is highly likely given standard message/flag lengths compared to the 2048-bit moduli), the congruence M ≡ m^3 (mod N) becomes the equality M = m^3. The script then calculates the integer cube root of M.

- Verification & Decoding: The script uses gmpy2.iroot which not only calculates the integer cube root (m) but also returns a boolean (exact) indicating if M was a perfect cube. If exact is true, it confirms that the correct combination of (n, c) pairs was found and M is indeed m^3. The resulting integer m is then converted back into bytes to reveal the original plaintext flag.

- Libraries Used:

  - gmpy2: For high-performance arbitrary-precision arithmetic, specifically the efficient iroot function for integer roots.

  - Cryptodome.Util.number: Provides number theoretic functions like modular inverse (inverse) and conversions (long_to_bytes).

  - itertools.combinations: Used to efficiently generate the combinations of (n, c) pairs needed for the attack.

```python
  # Import necessary libraries
import gmpy2 as gmpy # Used for high-precision integer arithmetic, especially the integer root function
from Cryptodome.Util import number # Used for number theoretic functions like modular inverse and byte/long conversion
from itertools import combinations # Used to generate combinations of (n, c) pairs

# Function to load n and c values from the output file
def load_output():
    # Initialize a dictionary to store lists of 'n' and 'c' values
    ret = {'n':[], 'c':[]}
    # Open the specified output file in binary read mode
    # NOTE: Ensure the filename "output_0ef6d6343784e59e2f44f61d2d29896f.txt" matches your actual file.
    with open("output_0ef6d6343784e59e2f44f61d2d29896f.txt", 'rb') as fd:
        # Read the file line by line
        while True:
            line = fd.readline()
            # Break the loop if end of file is reached
            if not line: break
            # Remove leading/trailing whitespace and decode from bytes to string
            line = line.strip().decode()
            # Skip empty lines
            if not line: continue
            
            # Split the line into key ('n', 'e', or 'c') and value parts based on '='
            k, v = line.split('=')
            # Remove whitespace from the key
            k = k.strip()
            # Skip the line if the key is 'e' (we already know e=3)
            if k == 'e':
                continue
            # Convert the value part to an integer and append it to the corresponding list in the dictionary
            ret[k].append(int(v))

    # Return the dictionary containing the lists of n and c values
    return ret

# Function to perform Hastad's attack
def decrypt(grps, e):
    # Use itertools.combinations to iterate through all unique combinations of size 'e' (which is 3)
    # from the available (n, c) pairs. zip(grps['n'], grps['c']) creates pairs like (n1, c1), (n2, c2), ...
    for grp in combinations(zip(grps['n'], grps['c']), e):
        # grp will be a tuple like ((n1, c1), (n2, c2), (n3, c3)) for e=3

        # Calculate the product of the moduli in the current group
        N = 1
        for x in grp: N *= x[0] # N = n1 * n2 * n3

        # Apply the Chinese Remainder Theorem (CRT) to find M such that M = c_i mod n_i for all i
        # M = sum( c_i * N/n_i * inverse(N/n_i, n_i) ) mod N
        M = 0
        for x in grp:
            # x[0] is n_i, x[1] is c_i
            term = N // x[0] # This is N/n_i
            # Calculate modular inverse: inverse(N/n_i, n_i)
            inv = number.inverse(term, x[0])
            # Add the component for this congruence to the total sum M
            M += x[1] * inv * term
        
        # Ensure M is within the range [0, N-1]
        M %= N

        # Calculate the integer e-th root (cube root since e=3) of M using gmpy2
        # gmpy.mpz(M) converts M to a gmpy2 multi-precision integer for efficiency/correctness
        # gmpy.iroot returns the integer root 'm' and a boolean 'exact'
        # 'exact' is True if M is a perfect e-th power, False otherwise
        m, exact = gmpy.iroot(gmpy.mpz(M), e)

        # Check if M was a perfect e-th power
        if exact:
            # If it was, 'm' is our original message integer m = M^(1/e)
            # Convert the integer 'm' back to bytes
            flag = number.long_to_bytes(m)
            # Print the recovered flag
            print(flag)


# --- Main execution ---
# Load the n and c values from the file
grps = load_output()
# Call the decrypt function with the loaded groups and the known public exponent e=3
decrypt(grps, 3)
```

Eventually you obtain the flag :

```text
 crypto{1f_y0u_d0nt_p4d_y0u_4r3_Vuln3rabl3}
```
