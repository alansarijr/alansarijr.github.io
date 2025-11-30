---
layout: post
title: m0leCon 2026 CTF - Login_System Writeup
categories: [general]
tags: [general]
---
m0leCon 2026 CTF - Login_System Writeup
Category: Reverse Engineering / Cryptography
Difficulty: Hard
Objective: Reverse engineer a custom database authentication system, identify hidden administrative accounts, and crack the password to login and get the flag.
1. Reconnaissance & File Analysis
The challenge provided two files:
login: A 64-bit ELF binary executable.
db.dat: A binary database file containing user credentials.

I started by opening the binary in Ghidra to decompile the key functions: check (login), reg (register), and custom_hash.
A. Analyzing check (The Login Function)
The login function reads the db.dat file in chunks to validate credentials.
undefined8 check(char *input_username, undefined8 input_password, undefined8 *output_struct) {
  FILE *__stream;
  undefined8 local_38; // Buffer for reading file
  // ... variables ...

  __stream = fopen("./.db.dat","rb");
  if (__stream == (FILE *)0x0) { /* Error handling */ }
  else {
    custom_hash(input_password, local_18); // Hash the input password
    do {
       // Read 0x1d (29) bytes from the file
       sVar3 = fread(&local_38, 0x1d, 1, __stream);
       if (sVar3 == 0) { /* End of file loop */ }
       
       // Compare Username (first part of the chunk)
       iVar1 = strcmp((char *)&local_38, input_username);
    } while (iVar1 != 0);
    
    // Compare Hash (offset +4 from the struct middle, effectively bytes 20-27)
    iVar1 = memcmp(auStack_28 + 4, local_18, 8);
    // ...
  }
  return uVar2; // Returns 1 if successful
}
Key Finding: The database (db.dat) structure is fixed at 29 bytes per record:
Bytes 0–19: Username (Null-terminated string).
Bytes 20–27: Password Hash (8 bytes).
Byte 28: Unknown/Padding (at this stage).

B. Analyzing reg (The Registration Function)
To understand the mysterious 29th byte, I analyzed the registration function.
void reg(void) {
  // ... inputs ...
  custom_hash(local_28, local_34); // Hash password
  
  local_2c = 0; // <--- CRITICAL: Sets the 29th byte to 0
  
  __s = fopen("./.db.dat","ab");
  fwrite(local_48, 0x1d, 1, __s); // Write 29 bytes to file
  
  printf("\nRegistration complete for user %s.\n", local_48);
  puts("All new users receive privilege level 0x00.");
}
Key Finding: The 29th byte is the Privilege Level. Regular users have 0x00. This implies that an Administrator would have a non-zero value (likely 0x01).
C. Analyzing custom_hash (The Crypto)
The binary uses a custom hashing algorithm instead of a standard one like SHA-256.
void custom_hash(char *password, void *output_buffer) {
  size_t pass_len;
  byte hash;
  ulong i, j;

  pass_len = strlen(password);
  memset(output_buffer, 0, 8); // Initialize 8-byte buffer

  for (i = 0; i < 8; i++) { // Generate 8 bytes of hash
    hash = 0;
    for (j = 0; j < pass_len; j++) {
      // XOR operation dependent on char value, outer index (i), and inner index (j)
      hash = hash ^ password[j] + (char)i + (char)j + 170U;
    }
    *(byte *)((long)output_buffer + i) = hash;

    // Dependency on previous byte
    if (i != 0) {
       *(byte *)(i + output_buffer) = 
          (*(char *)(output_buffer + i - 1) + *(char *)(output_buffer + i)) ^ 0xd;
    }
  }
}
2. Vulnerability & Exploitation
Step 1: Finding the Admin
I wrote a Python script to parse db.dat and look for any user where Byte 28 != 0.
import os

def find_admin(filename="db.dat"):
    """
    Scans a binary database file for users with non-zero privilege levels.
    
    Database Structure (29 bytes per record):
    - [0-19]  : Username (20 bytes, null-padded string)
    - [20-27] : Password Hash (8 bytes)
    - [28]    : Privilege Level (1 byte)
    """
    
    if not os.path.exists(filename):
        print(f"[-] Error: File '{filename}' not found.")
        return

    try:
        with open(filename, "rb") as f:
            data = f.read()
            
        record_len = 29
        total_records = len(data) // record_len
        print(f"[*] Scanning {total_records} records in '{filename}'...")
        
        found = False
        
        # Iterate through the file in 29-byte chunks
        for i in range(0, len(data), record_len):
            chunk = data[i : i + record_len]
            
            # Ensure we have a full record
            if len(chunk) < record_len:
                break
            
            # Extract the Privilege Byte (Byte 28)
            privilege = chunk[28]
            
            # Check if this user is special (Privilege != 0)
            if privilege != 0:
                # Extract Username (Bytes 0-19)
                # Decode as ASCII/UTF-8 and strip null bytes/spaces
                raw_name = chunk[:20].split(b'\x00')[0]
                try:
                    username = raw_name.decode('utf-8', errors='ignore').strip()
                except:
                    username = raw_name.hex()

                # Extract Hash (Bytes 20-27)
                target_hash = chunk[20:28].hex()
                
                print(f"\n[+] ADMIN FOUND!")
                print(f"    Username:  {username}")
                print(f"    Privilege: {privilege}")
                print(f"    Hash:      {target_hash}")
                found = True
        
        if not found:
            print("\n[-] No privileged users found.")
            
    except Exception as e:
        print(f"[-] An error occurred: {e}")

if __name__ == "__main__":
    find_admin()
Result: Found hidden admin user_2135 with hash 1028376b728691a9.
Step 2: Cracking the Hash (Z3 Solver)
Since the hash function relies on basic math (XOR/ADD) and the password length is limited to 19 chars, I used Z3 to reverse it mathematically.
# Z3 Solver Script
from z3 import *

target_hex = "1028376b728691a9"
target_bytes = bytes.fromhex(target_hex)

s = Solver()
# Create 5 byte variables (guessed length based on simple password)
p = [BitVec(f'p{i}', 8) for i in range(5)] 

# Constraints: Printable ASCII
for x in p: s.add(x >= 32, x <= 126)

# Re-implement Hash Logic
buf = [BitVec(f'b{i}', 8) for i in range(8)]
for i in range(8):
    h = BitVecVal(0, 8)
    for j in range(5):
        h = h ^ (p[j] + i + j + 170)
    
    if i == 0: s.add(buf[0] == h)
    else:      s.add(buf[i] == (buf[i-1] + h) ^ 0x0D)

# Match Target
for i in range(8): s.add(buf[i] == target_bytes[i])

if s.check() == sat:
    print(s.model())
Result: Password is 3wDVn.
Step 3: Login and get the Flag
3. Final Flag
ptm{Th15_L0g1n_Sy5tem_15_S0_B4d}