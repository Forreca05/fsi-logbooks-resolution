# Hash Length Extension Lab

## Setup

### **Docker Configuration**

Inside the *Labsetup* directory, the environment was built using the configuration provided in the `docker-compose.yml` file.

The following commands were executed:

```
dcbuild
dcup
```

These commands built and started all the containers required for the lab.

---

### **Updating the Hosts File**

To allow the server to be accessed through the hostname defined for this lab, an entry had to be added to the `/etc/hosts` file of the VM.

**Added line:**

```
10.9.0.80  www.seedlab-hashlen.com
```
 
![Server-added](Images/hosts-added.png)

This ensures that any HTTP request to `www.seedlab-hashlen.com` is correctly routed to the web server container.

---
 
## Task 1- Send Request to List Files

### **Objective**
The goal of this task is twofold:  
1. Send a benign request to the server to list the available files.  
2. Use the `download` command to retrieve the contents of one of those files.  

This demonstrates both the listing functionality and the ability to access file contents when the MAC is correctly calculated.

---

### **Procedure**

1. **Choose UID and secret key**  
   - In the file `key.txt` located in the `LabHome` directory, each line contains pairs in the format `uid:key`.  
   ![key.txt](Images/key.txt.png)

   - For this task, we select **UID 1002** with the associated key **983abe**.

2. **Construct the parameter string for listing (R)**  
   - The request must include our name, the UID, and the command `lstcmd=1`.  
   - Example:  
     ```
     myname=JoaoFerreira&uid=1002&lstcmd=1
     ```

3. **Concatenate with the secret key**  
   - Format:  
     ```
     <key>:<R>
     ```
   - Example:  
     ```
     983abe:myname=JoaoFerreira&uid=1002&lstcmd=1
     ```

4. **Calculate the MAC (SHA-256)**  
   - The MAC is computed using the following command:  
    ![lst-mac](Images/lst-mac.png)
     
   - Result:  
     ```
     cb9ca57a2c3da76c4f9e85cf54618ae42deb734471793dba4860bf2efa9e5332
     ```

5. **Send the list request**  
   - Final URL:  
     ```
     http://www.seedlab-hashlen.com/?myname=JoaoFerreira&uid=1002&lstcmd=1&mac=cb9ca57a2c3da76c4f9e85cf54618ae42deb734471793dba4860bf2efa9e5332
     ```
   - The server responds with the **list of available files**.  
   ![lst-result](Images/lst-result.png)

6. **Construct the parameter string for download (R)**  
   - To download a file, we just had to add `download=<filename>`.  
   - Example:  
     ```
     myname=JoaoFerreira&uid=1002&lstcmd=1&download=secret.txt
     ```

7. **Concatenate with the secret key and calculate MAC**  
   - Command:  
    The MAC is computed using the following command:
    ![download-mac](Images/download-mac.png)

   - Result:  
     ```
     76b069a0d6082d7ca421c234ec2a03ba49c02efe6921e7b7da1d9e97c748b284
     ```

8. **Send the download request**  
   - Final URL:  
     ```
     http://www.seedlab-hashlen.com/?myname=JoaoFerreira&uid=1002&lstcmd=1&download=secret.txt&mac=76b069a0d6082d7ca421c234ec2a03ba49c02efe6921e7b7da1d9e97c748b284
     ```
   - The server validates the MAC and returns the **contents of `secret.txt`**.  
   ![download-result](Images/download-result.png)

---

### **Results**
- The server successfully validated the MAC for both requests.  
- First, the **list of available files** was displayed.  
- Then, using the `download` command, the **contents of `secret.txt`** were retrieved.  

---

### **Conclusion**
Task 1 demonstrated how to:  
- Select a UID and secret key from the `key.txt` file.  
- Construct parameter strings for both listing and downloading.  
- Calculate the MAC using SHA-256.  
- Send requests to the server and obtain valid responses.  

This shows the importance of the **MAC** in authenticating requests and ensuring integrity, both for listing available files and for securely downloading their contents.

## Task 2- Create Padding

First we needed to calculate the length of the message:
![echonrwords](Images/echonrwords.png)

---

### Message and length
- **Message:** `983abe:myname=JoaoFerreira&uid=1002&lstcmd=1`
- **Length (bytes):** 44 (confirmed via wc)
- **Length  Field(bits):** \(44 * 8 = 352\) → `0x160`

---

### SHA-256 padding for 44 bytes
SHA-256 pads to 64-byte blocks with:
- **1 byte:** `0x80`
- **Zeros:** until total length before the length field is 56 bytes
- **Length field (8 bytes, big-endian):** `00 00 00 00 00 00 01 60`

For a 44-byte message:
- **After adding 0x80:** 45 bytes
- **Zeros needed to reach 56:** 11 bytes of `0x00`
- **Then the 8-byte length field:** `00 00 00 00 00 00 01 60`

Full padding bytes:
- `80` + `00` × 11 + `00 00 00 00 00 00 01 60`

---

### URL encoding of padding
Convert each byte to `%NN`:
- **Encoded padding:**  
  `  %80%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%01%60`

Breakdown:
- `%80`  
- `%00` repeated 11 times (zeros)  
- `%00%00%00%00%00%00%01%60` (length field)

## Task 3- The Length Extension Attack