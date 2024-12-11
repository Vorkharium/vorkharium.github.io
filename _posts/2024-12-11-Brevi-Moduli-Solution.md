---
layout: post
title:  HTB Brevi Moduli Solution (Without SageMath)
description: HTB Cryptography Challenge
date:   2024-12-11 21:00:00 +0300
image:  '/images/htb.png'
tags:   [Solution, HTB, Challenges, Python, Script, Cryptography, RSA]
---
This is the Python script I created to solve the cryptography Brevi Moduli challenge without using SageMath (I used Msieve through PowerShell instead to get the values of p and q from the given n):

```python
"""
HTB Challenge: Brevi Moduli - Solution

Author: Raúl (Aka. Vorkharium)
Website: https://vorkharium.com
HTB Profile: https://app.hackthebox.com/users/1791371

Description:
This Python script solves the "Brevi Moduli" challenge from Hack The Box (HTB).
The challenge involves interacting with a remote service that provides an RSA public key and expects the user to supply two factors, p and q, that multiply to form the modulus n. The script does not factorize n but instead requires the user to provide the factors manually when asked to while the script is running.

Usage:
Run the script with the following syntax:
    python3 solve.py <IP>:<PORT>
Where <IP>:<PORT> is the address and port of the challenge service.

Note:
- The script assumes that n = p * q and that the user factorizes n externally (e.g., using a tool like msieve).
- The script is designed to handle the interaction with the server through multiple rounds, sending the factors p and q to solve the challenge.
- The final flag is retrieved after completing all the rounds of the challenge.

External Tools Used:
- Msieve (Available on GitHub or Sourceforge) for factorization (example usage on PowerShell: .\msieve153.exe -q <n>)
- Example: .\msieve153.exe -q 1463098089349801395141390157834302632433387077508478210190289697027
1463098089349801395141390157834302632433387077508478210190289697027
p33: 1191213103055447538419975585021327 (p = first pumpkin)
p34: 1228242105125415493207865587809101 (q = second pumpkin)

Changelog:
    11.12.2014 - HotFix: Increased timeout to 5 seconds to prevent problems when receiving the final server response, which includes the flag.

Dependencies:
- pwn (for interacting with the remote service)
- pycrypto (for RSA key handling)
- re (for regular expression matching)
- sys (for handling command-line arguments)

"""

from pwn import *
from Crypto.PublicKey import RSA
import re
import sys

print("Imported all required modules.")

def get_process():
    print("Entering get_process()...")
    try:
        host, port = sys.argv[1].split(':')
        print(f"Parsed host: {host}, port: {port}")
        return remote(host, int(port))
    except IndexError:
        print(f"Usage: python {sys.argv[0]} <ip:port>")
        exit(1)

def solve_challenge():
    print("Entering solve_challenge()...")
    conn = None
    try:
        print("Calling get_process()...")
        conn = get_process()
        print("Connection established.")
        for round_num in range(5):
            print(f"\nStarting Round {round_num + 1}/5")
            conn.recvuntil(b'Round')
            print("Received round indicator.")
            line = conn.recvline().decode()
            print(f"Current round: {line.strip()}")
            data = conn.recvuntil(b'enter your first pumpkin = ').decode()
            print("Received challenge data.")
            match = re.search('-----BEGIN PUBLIC KEY-----(.*?)-----END PUBLIC KEY-----', data, re.DOTALL)
            if not match:
                print("Could not find PUBLIC KEY in response.")
                raise ValueError("Could not find PUBLIC KEY in response")
            print("Found PUBLIC KEY.")
            pem = match.group(0)
            print(f"Extracted PEM:\n{pem}")
            key = RSA.importKey(pem.encode())
            print("Imported RSA key.")
            n = key.n
            print(f"Modulus n = {n}")
            
            # Prompt user for p and q
            try:
                print("Prompting user for factors...")
                p = int(input("Enter the first factor (p): "))
                q = int(input("Enter the second factor (q): "))
            except ValueError:
                print("Invalid input for p or q. Please enter valid integers.")
                return
            
            print(f'Factors provided: p = {p}, q = {q}')
            print(f'Verifying: {p} * {q} == {n}')
            if p * q != n:
                print("Factorization verification failed!")
                raise ValueError("Factorization verification failed!")
            print("Verification succeeded.")
            
            conn.sendline(str(p).encode())
            print("Sent first factor")
            conn.recvuntil(b'enter your second pumpkin = ')
            print("Server asked for second factor.")
            conn.sendline(str(q).encode())
            print("Sent second factor")
            response = conn.recvline(timeout=5)
            print(f"Server response: {response.decode().strip()}")
        
        print("All rounds completed. Sending final newline...")
        conn.sendline(b'')  # Send a final newline
        remaining = conn.recvall(timeout=5)
        if remaining:
            print(f"Final server response: {remaining.decode().strip()}")
    except Exception as e:
        print(f'Error occurred: {str(e)}')
    finally:
        print("Closing connection...")
        if conn:
            conn.close()
        print("Connection closed.")

if __name__ == '__main__':
    print("Executing main...")
    solve_challenge()
```
