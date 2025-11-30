---
layout: post
title: m0leCon 2026 CTF - Astro-Logic Writeup
categories: [general]
tags: [general,m0leCon 2026 CTF]
---
**m0leCon 2026 CTF - Login_System Writeup**

In this CTF competition, I tackled a fascinating Reverse Engineering challenge called "Astro-Logic" What started as a simple "astrology calculator" turned into a deep dive involving logic inversion, custom checksum verification, and dynamic memory analysis. Here is a technical breakdown of how I solved it.


# 1. Initial Reconnaissance
The challenge provided a 64-bit ELF binary (astro). Upon execution, it presented a "Compatibility Test" interface asking for two specific inputs:
- Natal Coordinates: A date in DD MM YYYY format.
- Resonant Frequency: A "Favorite Number" (integer).

The program would process these inputs through three distinct "phases" and, if successful, verify a "Cosmic Blueprint." My goal was to find the specific input combination that unlocked the flag.

# 2. Static Analysis (Ghidra)
I loaded the binary into Ghidra to analyze the control flow. The main function revealed three critical gates that the input had to pass.

## Gate 1: The Logic Trap (calculate_cosmic_alignment)
This function calculated an "alignment score" (float) based on the date provided. It summed values from a hardcoded integer array (constellation) using the date as an index offset.

```cpp
// Simplified Decompilation
float score = (sum_week_2 + sum_week_1 * day + month) % (sum_week_3 + year);
score = score / (sum_week_3 + year);

if (score <= 0.7) { 
    // The decompiler shows this as the "Success" path...
    // BUT runtime testing proved this wrong.
}
```

The Twist: The decompiler output was misleading (likely due to compiler optimization or a logical inversion in assembly). While the code looked like it wanted a low score (<= 0.7), running the binary with inputs that produced 0.55 resulted in failure. Fix: I deduced the condition was actually inverted. The program required a high score (> 0.7). I calculated that the date 02/01/1990 produced a score of ~0.87, bypassing this gate.

## Gate 2: The Boost (calculate_celestial_boost)

This function took the "Favorite Number" and added a bonus to the alignment score.

The Math: It performed a modulo operation: (Sum * Count) % fav_num.

The Objective: The total score (Alignment + Boost) needed to be >= 1.0.

The Solution: To maximize the modulo remainder (and thus the score), I needed a divisor that left a large remainder. Using a prime number like 97 or 69 maximized this boost, pushing the total score well above 1.0.

## Gate 3: The Decryption (unveil_cosmic_blueprint)
This was the core cryptographic challenge. The function didn't just check the inputs; it used them as a Seed Key to decrypt the flag in memory.

```cpp 
void unveil_cosmic_blueprint(int day, int month, int year, int fav_num) {
    // 1. Copy encrypted data to Heap
    void *heap_buffer = malloc(0x80);
    memcpy(heap_buffer, constellation, 0x80); 
    
    // 2. Decrypt using inputs as Seed
    for (int i = month; i > 0; i--) {
        int seed = fav_num * j + year * i;
        // ... XOR decryption loop ...
    }
    
    // 3. Verify Integrity (CRC32 Check)
    if (verify_integrity(heap_buffer) == 0xB309B2F7) {
        puts("Perfect match!");
    }
}
```

The Problem: Just "passing the math" wasn't enough. I needed the exact date and number that, when used as XOR seeds, transformed the encrypted data into a valid byte array matching the CRC32 checksum 0xB309B2F7.

# 3. The Solution: Brute-Force Solver

Since mathematically reversing a CRC32 checksum into specific XOR seeds is computationally infeasible, I analyzed the key space:

Dates: ~365,000 possibilities (Years 1990-3000).

Numbers: 100 possibilities (0-99).

Total: ~37 million combinations.

```cpp 
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <stdbool.h>

/* * ASTRO CHALLENGE SOLVER
 * ----------------------
 * 1. Brute-forces the Date and Fav Number to satisfy the CRC32 checksum.
 * 2. Decrypts the memory buffer using the correct seed.
 * 3. Prints the flag.
 */

// The encrypted data extracted from the binary's memory (constellation array)
uint32_t constellation[32] = {
    112, 68, 189, 251, 55, 128, 103, 127,
    145, 0, 128, 197, 63, 3, 132, 212,
    50, 15, 211, 111, 109, 224, 49, 20,
    81, 18, 185, 172, 73, 7, 41, 93
};

// Standard CRC32 Implementation used by the binary
uint32_t verify_integrity(uint32_t *data, int count) {
    uint32_t crc = 0xffffffff;
    for (int i = 0; i < count; i++) {
        crc = crc ^ data[i];
        for (int j = 0; j < 8; j++) {
            if ((crc & 1) == 0) {
                crc = crc >> 1;
            } else {
                crc = crc >> 1 ^ 0xedb88320;
            }
        }
    }
    return ~crc;
}

// Simulates the 'unveil_cosmic_blueprint' decryption logic
bool check_candidate(int day, int month, int year, int fav_num) {
    
    uint32_t buffer[32];
    uint32_t ptr[32];
    uint32_t ptr_00[32]; // Key buffer
    uint32_t ptr_01[32]; // Index buffer

    // Reset buffer with encrypted data
    memcpy(buffer, constellation, sizeof(constellation));

    // Decryption Loop (Reversed from Ghidra)
    for (int i = month; i > 0; i--) {
        for (int j = 0x1f; j >= 0; j--) {
            
            // 1. Generate Permutation Indices
            for (int k = 0; k < day; k++) {
                if (k >= 32) break;
                int idx = (j + k) % 32;
                ptr_01[k] = idx;
                ptr[k] = buffer[idx];
            }

            // 2. Generate XOR Key (Seed)
            // Seed depends on Fav Num, Year, Month(i), and Loop(j)
            int seed = fav_num * j + year * i;
            
            for (int k = 0; k < day; k++) {
                if (k >= 32) break;
                seed = (seed * 0xd + 0x11) % 0x100;
                ptr_00[k] = seed;
            }

            // 3. XOR Step
            for (int k = 0; k < day; k++) {
                if (k >= 32) break;
                ptr[k] = ptr_00[k] ^ ptr[k];
            }

            // 4. Shuffle Write Back
            for (int k = 0; k < day; k++) {
                if (k >= 32) break;
                buffer[ptr_01[k]] = ptr[k];
            }
        }
    }

    // Check Integrity against the target checksum
    // Target: -0x4cf64d09 (signed) -> 0xB309B2F7 (unsigned)
    if (verify_integrity(buffer, 32) == 0xB309B2F7) {
        
        printf("\n[+] SUCCESS! Key Found.\n");
        printf("    Date: %d/%d/%d\n", day, month, year);
        printf("    Num:  %d\n", fav_num);
        
        printf("\n[+] Decrypted Flag:\n    ");
        for(int k=0; k<32; k++) {
            char c = (char)(buffer[k] & 0xFF);
            if (c >= 32 && c <= 126) {
                printf("%c", c);
            }
        }
        printf("\n");
        return true;
    }
    return false;
}

int main() {
    printf("Starting Brute-Force Solver...\n");
    printf("Target Checksum: 0xB309B2F7\n\n");

    // Loop Years (1990 - 3000)
    for (int y = 1990; y <= 3000; y++) {
        
        // Progress Indicator
        if (y % 100 == 0) { printf("Checking Year: %d...\r", y); fflush(stdout); }

        for (int m = 1; m <= 12; m++) {
            for (int d = 1; d <= 31; d++) {
                
                // Loop Favorite Numbers (0-99)
                for (int f = 0; f < 100; f++) {
                    if (f % 7 == 0) continue; // Skip numbers divisible by 7 (Program constraint)

                    if (check_candidate(d, m, y, f)) {
                        return 0; // Exit on success
                    }
                }
            }
        }
    }

    printf("\n[-] No key found. Check constellation array data.\n");
    return 0;
}
```

Result: The solver found the key in under 2 seconds.

- Date: 15/08/2022
- Number: 69

# 4. Dynamic Analysis: The "Invisible Flag" Trap
Entering these values into the binary triggered the success message: "Unbelievable... A perfect match!" However, the flag was not printed.

I had to use GDB (GNU Debugger) to inspect the memory at the exact moment of decryption.

The Trap: Inspecting the global variable constellation showed encrypted garbage. The Reason: The code decrypts the data into a temporary malloc buffer on the Heap, verifies it, and then immediately free()s it. The global variable remains touched.

The Fix: I set a breakpoint on the checksum function (verify_constellation_integrity) to catch the pointer to the heap buffer before it was destroyed.

```bash
$ gdb ./astro
(gdb) break verify_constellation_integrity
(gdb) run
# Inputs: 15 8 2022, 69
(gdb) x/32s $rdi  <-- The register holding the heap pointer
```

**Memory Dump:** 0x555555559ac0: "ptm{W0w_100%_St4r_coMpAt1BiLi7y}"

# 5. Conclusion
This challenge was a great exercise in:

- Logic Analysis: Identifying inverted conditions in decompiled code.
- Cryptanalysis: Recognizing small key spaces suitable for exhaustive search.
- Memory Forensics: Understanding Heap vs. Global memory to locate transient decrypted data.

**Final Flag:** `ptm{W0w_100%_St4r_coMpAt1BiLi7y}`