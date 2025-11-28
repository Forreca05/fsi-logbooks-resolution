# Secret-Key Encryption

## Task 1- Frequency Analysis

### **1. Task Description**

In this task, we are given a ciphertext that was produced using a **monoalphabetic substitution cipher**.
The goal was to **recover the full plaintext** and **determine the substitution key** by applying **frequency analysis** and manual pattern recognition.

We are told that:

* All letters were converted to lowercase.
* All punctuation and digits were removed.
* Spaces were kept to simplify the task.
* A randomized permutation of the alphabet was used as the encryption key.
* The encryption was performed using:

```
tr 'abcdefghijklmnopqrstuvwxyz' '<random_permutation>' < plaintext.txt > ciphertext.txt
```

We are also provided with a Python tool `freq.py`, which outputs frequency statistics from the ciphertext.
The given output was:

```
$ ./freq.py
-------------------------------------
1-gram (top 20):
n: 488
y: 373
v: 348
...
-------------------------------------
2-gram (top 20):
yt: 115
tn: 89
mu: 74
...
-------------------------------------
3-gram (top 20):
ytn: 78
vup: 30
mur: 20
...
```

Our task is to use these frequency patterns to infer the plaintext.

---

### **2. Methodology**

**Replace the incorrect paragraph with this corrected version**

Based on the 3-gram and 2-gram frequencies, I made initial assignments using the most common English trigrams and bigrams as anchors:
>
* The ciphertext trigram **ytn** (most frequent) strongly suggested **“the”**, so I assigned `y → t`, `t → h`, `n → e`.
* The bigrams **yt** and **tn** then became **th** and **he**, which confirmed the trigram hypothesis.
* Next, the frequent bigram **mu** suggested **“in”** (`m → i`, `u → n`).
* The trigram **vup** matched **“and”** (`v → a`, `p → d`) and **mur** matched **“ing”** (`r → g`).
  To test these hypotheses I applied partial substitutions with `tr` and inspected the result, using uppercase to mark confirmed plaintext letters. For example:

```
tr 'nytvupmr' 'ETHANDIG' < ciphertext.txt > partial.txt
```

This produced obvious words like **THE**, **AND**, **IN**, and **ING**, which provided strong context for discovering the remaining letters.


**Iterative Refinement**

As I identified more patterns and recognizable English words,
I gradually expanded the mapping.

Examples:

* “EkTRA” in partial plaintext clearly corresponds to **EXTRA**
  ⇒ ciphertext `k` → plaintext `X`

* “SEkUAL” corresponds to **SEXUAL**
  ⇒ confirms `k` → X, `s` → S, etc.

* “PRIwE” corresponds to **PRIZE**
  ⇒ ciphertext `w` → Z

Each discovery was added to the evolving translation table using new `tr` commands.

---

### **3. Final Key Recovery**

After completing the frequency-analysis process, the full substitution key was recovered.

**Cipher Alphabet → Plain Alphabet**

```
nytvupmrqhfecxbaidlgzjsokw 
ETHANDIGSRVPMOFCLYWBUQKJXZ
```

**Final Decryption Command**

```
tr 'nytvupmrqhfecxbaidlgzjsokw' 'ETHANDIGSRVPMOFCLYWBUQKJXZ' < ciphertext.txt > partial.txt
```

This produced a **fully readable English article** discussing:

* The Oscars ceremony
* MeToo and Times Up
* Hollywood awards-season politics
* Statistical details of Oscar voting
* Predictions and controversies

![Task1-Text](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/Task1-text.png?ref_type=heads)
---

## Task 2- Encryption using Different Cyphers and Models

In this task, I explored AES-128 encryption in three modes: **ECB**, **CBC**, and **CTR**. I generated a plaintext file with 1200 bytes and encrypted it with all three modes.

### **Step 1: Create the plaintext file**

I created a file named `plain.txt` with 1200 bytes using the following command:

![plain.txt](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/Task2-plain.txt-creation.png?ref_type=heads)


This generates 1200 random bytes as input for encryption.

---

### **Step 2: Encrypt the file**

**AES-128-ECB**

![ecb-encrypting](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ecb-encryption.png?ref_type=heads)

**Note:** ECB does not use an IV; only the key (`-K`) is required.

#### **AES-128-CBC**

![cbc-encrypting](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/cbc-encryption.png?ref_type=heads)

**Note:** CBC requires both the key (`-K`) and IV (`-iv`).

**AES-128-CTR**

![ctr-encrypting](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ctr-encryption.png?ref_type=heads)

**Note:** CTR also requires the key and IV; it functions as a stream cipher.

---

### **Step 3: Decrypt the files**

**AES-128-ECB**

![ecb-decrypting](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ecb-decryption.png?ref_type=heads)

**AES-128-CBC**

![cbc-encrypting](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/cbc-decryption.png?ref_type=heads)

**AES-128-CTR**

![ctr-encrypting](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ctr-decryption.png?ref_type=heads)

---

### **Step 4: Verify correctness**

I verified the decrypted files against the original using `diff`:

![diff-ecb](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/diff.png?ref_type=heads)

**Result:** No differences were reported. This confirms that encryption and decryption worked correctly for all three modes.

---

### **Step 5: Answering to the Questions**


**1. When encrypting, which flags did I need to specify?**

To encrypt using OpenSSL, I needed to use:

* `-e` — tells OpenSSL to **encrypt**
* `-in` — input file (plaintext.txt)
* `-out` — output encrypted file
* `-K` — AES key in hexadecimal (**required for all modes**)
* `-iv` — **required only for AES-128-CBC and AES-128-CTR**
  (AES-128-ECB does *not* use an IV)

---

**2. What is the difference between the AES-128-ECB, AES-128-CBC, and AES-128-CTR modes?**

* **AES-128-ECB**
  Encrypts each 16-byte block independently; **no IV** is used.
  Patterns in the plaintext remain visible in the ciphertext.

* **AES-128-CBC**
  Each block is XORed with the previous ciphertext block; uses an **IV** for the first block.
  More secure than ECB but error propagation occurs.

* **AES-128-CTR**
  Converts AES into a **stream cipher**: encrypts a counter combined with an IV.
  No padding required, blocks can be processed independently.

---

**3. When decrypting, which flags did I need to specify?**

For decryption, the required flags were:

* `-d` — tells OpenSSL to **decrypt**
* `-in` — encrypted input file
* `-out` — output plaintext file
* `-K` — same key used during encryption
* `-iv` — **needed for CBC and CTR**, not used in ECB

---

**4. What is the main difference between AES-128-CTR and the other modes?**

The key difference of **AES-128-CTR** is that it operates as a stream cipher, not a block cipher mode. But there are other differences, such as:
* No padding is used
* Encryption and decryption are essentially the same operation
* A change in one byte only affects that byte
* Blocks can be processed in parallel

In contrast, **AES-128-ECB and AES-128-CBC** are **block-based modes**, require padding, and operate strictly on 16-byte blocks.

### **Step 6 : Summary Table**

| Mode    | Encryption Flags     | Decryption Flags     | IV Required? | Notes                                            |
| ------- | -------------------- | -------------------- | ------------ | ------------------------------------------------ |
| **ECB** | `-e -in -out -K`     | `-d -in -out -K`     | ❌ No         | Blocks independent, patterns visible             |
| **CBC** | `-e -in -out -K -iv` | `-d -in -out -K -iv` | ✔ Yes        | Blocks chained, errors propagate                 |
| **CTR** | `-e -in -out -K -iv` | `-d -in -out -K -iv` | ✔ Yes        | Stream cipher, no padding, byte-level encryption |

**Key takeaway:** AES-128-CTR behaves differently from ECB and CBC because it is stream-based, not block-based, allowing flexible and parallel processing.

## Task 5- Error Propagation- Corrupted Cypher Text

### **1. Introduction**

The goal of this task was to analyze how different AES encryption modes behave when the ciphertext is partially corrupted.

Using the previously generated `plain.txt` file (≈1200 bytes), I encrypted it using three AES-128 modes and then manually corrupted **byte 200** of each ciphertext using the **Bless** hex editor.

After corruption, each ciphertext was decrypted using the correct key and IV, and the resulting plaintexts were compared to the original to evaluate information loss.

The AES modes tested:

* **AES-128-ECB**
* **AES-128-CBC**
* **AES-128-CTR**

Because AES operates on **16-byte blocks**, error propagation heavily depends on the mode.

---

### **2. Block Calculation**

The corrupted byte is the **200th byte** of the ciphertext.

AES block size = **16 bytes**, so:

200/16 = 12

Therefore:

* The corrupted byte lies in **Block 12** (0-based indexing)
* That block corresponds to plaintext offsets **192–207**

---

### **3. Expected Behavior (Before Conducting the Experiment)**

Below are the theoretical predictions for each AES mode.

---

#### **3.1 AES-128-ECB — Block Independence**

**Prediction:**

* Only the corrupted block (block 12) becomes unreadable.
* No propagation to other blocks.

**Expected information loss:**  
➡️ **16 bytes**

---

#### **3.2 AES-128-CBC — Chained Blocks**

**Prediction:**

1. **Block 12 (192–207): fully corrupted**  
   Because `Dec(C₁₂)` becomes random.

2. **Block 13 (208–223): 1 corrupted byte**  
   Because the corrupted byte in `C₁₂` is XORed into the next block.

**Expected information loss:**  
➡️ **16 bytes destroyed + 1 byte corrupted**

---

#### **3.3 AES-128-CTR — Stream Cipher Behavior**

**Prediction:**

* Only the modified ciphertext byte affects the plaintext.
* No error propagation.

**Expected information loss:**  
➡️ **1 byte**

---

### **4. Experimental Procedure**

1. Encrypted the plaintext using AES-ECB, AES-CBC and AES-CTR.

![ecb-encrytion](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ecb-encryption2.png)
![cbb-encrytion](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/cbc-encryption2.png)
![ctr-encrytion](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ctr-encryption2.png)

2. Opened each ciphertext using **Bless**.

![ecb-byte200](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ecb-byte200.png)
![cbc-byte200](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/cbc-byte200.png)
![ctr-byte200](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ctr-byte200.png)

3. Modified **byte 200** in each ciphertext.  
4. Decrypted all corrupted ciphertexts. 
    
![ecb-decrytion](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ecb-decryption2.png)
![cbc-decrytion](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/cbc-decryption2.png)
![ctr-decrytion](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ctr-decryption2.png)

5. Compared them with the original plaintext using `cmp`, `diff`, and `hexdump`.

---

### **5. Experimental Results**

The results fully matched the theoretical predictions.

---

### **ECB – Only the Corrupted Block Is Affected**

* Block **192–207** was completely corrupted.  
* All remaining bytes were correct.

![ecb-comparation](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%239/Images/ecb-comparation.png)

---

### **CBC – One Full Block + One Byte Corrupted**

* Block **192–207**: fully corrupted  
* Block **208–223**: exactly **one byte** corrupted  

![cbc-comparation](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%239/Images/cbc-comparation.png)

---

### **CTR – Only One Byte Affected**

* Only the plaintext byte corresponding to ciphertext byte 200 was corrupted.  
* No additional errors.

![ctr-comparation](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/blob/main/Semana%20%239/Images/ctr-comparation.png)

---

### **6. Summary Table**

| AES Mode | Fully Lost Data | Partially Corrupted Data | Error Propagation |
|----------|------------------|---------------------------|--------------------|
| **ECB**  | 16 bytes         | 0 bytes                   | Only the modified block |
| **CBC**  | 16 bytes         | 1 byte                    | One block + one byte in next block |
| **CTR**  | 0 bytes          | 1 byte                    | No propagation |

---

### **7. Conclusion**

This experiment demonstrated clear differences in how AES modes propagate errors:

* **ECB** has no chaining, so corruption is limited to a single 16-byte block.
* **CBC** chains blocks, causing one full block to be destroyed and one additional byte to be corrupted.
* **CTR** behaves like a synchronous stream cipher, so only the modified byte is affected.

The experimental results confirmed the theoretical predictions perfectly.
