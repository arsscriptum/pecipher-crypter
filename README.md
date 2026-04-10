# Portable Executable Crypter reasearch and Development

For high-security, self-implemented software protection and distribution control (akin to CodeMeter but DIY), your technology choice depends on your *risk model*, target audience, and how much effort you want to invest in anti-tamper/anti-reversing. Here’s a practical breakdown:

---

### 1. **Windows, C++, or .NET C#?**

* **C++ (native, unmanaged):**

  * **Pros:** Native code is harder to reverse-engineer than .NET bytecode. Supports custom loaders, packing, encryption, anti-debug, obfuscation at binary level. Greater flexibility for custom cryptography and runtime decryption.
  * **Cons:** More complex memory and resource management. Building robust, non-trivial protection is time-consuming. Compatibility issues (different Windows versions/architectures).

* **.NET / C# (managed):**

  * **Pros:** Faster development, easier integration with Windows features (WPF, WinForms). Tools exist for obfuscation, but real security is limited—MSIL (bytecode) can always be reverse-engineered with tools like dnSpy or ILSpy.
  * **Cons:** Even with obfuscation, managed assemblies are much easier to decompile and modify. Runtime decryption and execution is harder to secure, as the JIT process can be hooked.

* **Windows Platform:**

  * Both C++ and C# can be tightly integrated with Windows APIs, including secure storage (DPAPI, certificates), user/PC binding, etc. No significant difference in protection options here.

---

### 2. **Protecting the EXE & Datablock (Private Key)**

#### a) **EXE Encryption & Runtime Decryption**

* Implement a custom loader/stub: The real payload is encrypted, and a tiny stub decrypts and loads it into memory at runtime.
* **C++:** You can write a loader in C++ that:

  * Decrypts the EXE section or resource in memory
  * Uses APIs like `VirtualAlloc`, `WriteProcessMemory`, `CreateThread` to launch from memory (process hollowing or reflective loading)
* **C#:** Technically possible but much less secure, since all memory is managed and debuggers/IL decompilers can easily access raw memory.

#### b) **Datablock (Resource) Protection**

* Store the private key/data encrypted in the resource section.
* At runtime, decrypt only when needed, and keep it in memory as short as possible.
* Use strong symmetric encryption (AES-256), key derived from hardware/user/machine fingerprint.
* **C++:** Can access resources at binary level, decrypt and use directly in native code.
* **C#:** Use `ResourceManager`, but again, less secure against memory inspection.

#### c) **Anti-Reverse Engineering**

* Obfuscation: Use commercial or open-source obfuscators.
* Anti-debug: Check for debuggers, breakpoints, timing checks, hardware breakpoints, etc.
* Anti-dump: Encrypt memory, use guard pages, self-modifying code.
* White-box cryptography for key handling.

#### d) **User-Binding & Distribution Control**

* Bind license/keys to hardware IDs (MAC, CPU, motherboard, TPM, etc.)
* Activate via server challenge/response, with short-lived tokens.
* Consider code-signing and verifying signature at runtime.

---

### 3. **Recommendation**

**If your priority is maximum control over security and distribution:**

* **Go with native C++ on Windows.**

  * Allows lowest-level tricks and the hardest possible protection you can roll yourself.
  * You can implement custom packers/loaders, integrate deep anti-debugging, and use all Windows-native anti-reverse engineering techniques.
  * Much easier to do robust runtime EXE/resource decryption.

**If rapid development is more important than peak security (and you accept the risks):**

* **C#/.NET is fine**, but understand that any secret can be reverse-engineered with enough effort.

---

### 4. **Useful Tools/Keywords to Research**

* PE file format, process hollowing, reflective DLL injection
* VMProtect, Themida, Enigma Protector (as reference for techniques)
* Open-source packers/cryptors on GitHub
* White-box cryptography for key obfuscation
* Windows DPAPI for storing keys (as an extra layer)

---

**Summary:**
For serious anti-tamper, encrypted runtime loading, and resource protection, native C++ is the best foundation. .NET can be obfuscated but is fundamentally less secure. No software-only solution is perfect, but custom C++ gives you the highest barrier against unauthorized access and distribution.

Let me know if you want example skeletons for loaders or specific C++ anti-reverse engineering strategies.


Here’s a minimal C++ “crypter” skeleton: it takes a payload (binary data, e.g. another EXE or DLL, or even a data block), encrypts it (for this demo, with AES-256), and at runtime decrypts it in-memory for use. This example is for educational purposes—production-grade versions add much more anti-debug/anti-dump/obfuscation logic.

**Dependencies:**

* Use a simple AES library, e.g. [tiny-AES-c](https://github.com/kokke/tiny-AES-c) or Crypto++.
* Example below uses [tiny-AES-c](https://github.com/kokke/tiny-AES-c), which is easy to drop into any project.

---

### 1. **Encrypting Your Payload (One-time step)**

```cpp
#include <fstream>
#include <vector>
#include "aes.h"  // from tiny-AES-c

void encrypt_payload(const char* infile, const char* outfile, const uint8_t* key, const uint8_t* iv) {
    std::ifstream fin(infile, std::ios::binary);
    std::vector<uint8_t> data((std::istreambuf_iterator<char>(fin)), std::istreambuf_iterator<char>());
    fin.close();

    // Pad to multiple of 16 bytes
    size_t pad = 16 - (data.size() % 16);
    data.insert(data.end(), pad, pad);

    // Encrypt
    struct AES_ctx ctx;
    AES_init_ctx_iv(&ctx, key, iv);
    AES_CBC_encrypt_buffer(&ctx, data.data(), data.size());

    std::ofstream fout(outfile, std::ios::binary);
    fout.write(reinterpret_cast<const char*>(data.data()), data.size());
    fout.close();
}

// Example key/IV (NEVER hardcode in real app)
uint8_t key[32] = {0}; // 256-bit key
uint8_t iv[16] = {0};  // 128-bit IV

int main() {
    encrypt_payload("payload.bin", "payload.enc", key, iv);
    return 0;
}
```

---

### 2. **Crypter Skeleton – Decrypt at Runtime**

```cpp
#include <fstream>
#include <vector>
#include "aes.h"  // from tiny-AES-c

// Decrypt function
std::vector<uint8_t> decrypt_payload(const char* infile, const uint8_t* key, const uint8_t* iv) {
    std::ifstream fin(infile, std::ios::binary);
    std::vector<uint8_t> enc_data((std::istreambuf_iterator<char>(fin)), std::istreambuf_iterator<char>());
    fin.close();

    struct AES_ctx ctx;
    AES_init_ctx_iv(&ctx, key, iv);
    AES_CBC_decrypt_buffer(&ctx, enc_data.data(), enc_data.size());

    // Remove padding
    uint8_t pad = enc_data.back();
    enc_data.resize(enc_data.size() - pad);
    return enc_data;
}

int main() {
    uint8_t key[32] = {0}; // Your 256-bit key here
    uint8_t iv[16] = {0};  // Your 128-bit IV here

    // Decrypt payload
    std::vector<uint8_t> payload = decrypt_payload("payload.enc", key, iv);

    // At this point, "payload" contains the original data.
    // You can, for example, write it to a temp file, load as DLL, execute, or parse as resource.

    // Example: write to disk
    std::ofstream fout("payload_out.bin", std::ios::binary);
    fout.write(reinterpret_cast<const char*>(payload.data()), payload.size());
    fout.close();

    return 0;
}
```

---

### 3. **Integrate in-memory execution (optional, advanced)**

* For EXE: Use process hollowing or reflective loading.
* For DLL: Use [Reflective DLL Injection](https://github.com/stephenfewer/ReflectiveDLLInjection) or similar technique.
* For custom data: Just use the decrypted buffer as needed.

---

### **Tips for Real-World Use**

* Never hardcode your key/IV; derive from user/machine info or obfuscate it.
* Delete all decrypted material from memory ASAP.
* Add anti-debugging and tamper-resistance.
* Hide payload in resources or as a section in your EXE, not as a separate file.

---

Let me know if you want this tailored for a specific payload type or in-memory loading.



Technically, this is called **runtime code encryption/decryption** or **“on-demand code decryption”**. The general method is to store sensitive routines (functions, code blocks, modules) in an encrypted state inside the binary. When the function is needed, it’s decrypted into memory, executed, and then re-encrypted or wiped from memory.

Here’s how it’s done at the low level (in C/C++ on Windows, but similar concepts apply elsewhere):

---

## 1. **Encrypting the Function’s Machine Code**

* Identify the function’s address range in the binary (from disassembly or linker map).
* Before building the final executable, encrypt the raw machine code bytes of that function (AES, etc.).
* The encrypted blob replaces the original code in the binary, or is stored as a resource/section.
* Add a small “stub” at the function’s entry, which is not encrypted, that triggers the decrypt/reencrypt routine.

---

## 2. **At Runtime: Decryption, Execution, Re-encryption**

**Step-by-step:**

1. **Intercept the Function Call**
   The function entry is a stub or trampoline that checks if the function is encrypted. If so, it decrypts the function code into the same memory location.

2. **Change Memory Protection**
   Use `VirtualProtect` (Windows) to mark the memory region as writable/executable (`PAGE_EXECUTE_READWRITE`).

3. **Decrypt in Place**
   Decrypt the encrypted code bytes directly into the memory region where the function resides.

4. **Execute Function**
   Call/jump into the now-decrypted function code.
   Optionally, you may patch the return or exit with a re-encryption stub.

5. **Re-encrypt or Wipe**
   After execution, re-encrypt the memory region, or fill with zeros/random data if security is critical.
   Reset memory protections with `VirtualProtect` as needed.

---

## 3. **Code Example Skeleton**

**This is a concept, not a drop-in library:**

```cpp
// Assume function to protect is called "SensitiveFunc"
void SensitiveFunc() {
    // ... code ...
}

// Metadata for the function's address/size
uintptr_t func_addr = (uintptr_t)&SensitiveFunc;
size_t func_size = /* size of function in bytes (disassembly/linker map) */;

void decrypt_function(uint8_t* addr, size_t size, uint8_t* key, uint8_t* iv) {
    VirtualProtect(addr, size, PAGE_EXECUTE_READWRITE, &oldProtect);
    AES_CBC_decrypt_buffer(&ctx, addr, size); // use your crypto
}

void reencrypt_function(uint8_t* addr, size_t size, uint8_t* key, uint8_t* iv) {
    AES_CBC_encrypt_buffer(&ctx, addr, size);
    VirtualProtect(addr, size, PAGE_EXECUTE_READ, &oldProtect);
}

// At runtime:
decrypt_function((uint8_t*)func_addr, func_size, key, iv);
((void(*)())func_addr)();  // Call function
reencrypt_function((uint8_t*)func_addr, func_size, key, iv);
```

---

## 4. **Advanced Implementation Details**

* **Automatic Trampoline**:
  Replace function entry with a small assembly stub that decrypts and jumps.
* **Code Section Manipulation**:
  Most OSs require executable memory to be read-only for security. Temporarily change protections with `VirtualProtect`.
* **Obfuscation**:
  Combine with control flow obfuscation to defeat static analysis.
* **Self-Deleting Code**:
  After execution, wipe or randomize the code region if not re-encrypting.

---

## 5. **Security Implications**

* While in memory (even briefly), the function can be dumped by a skilled attacker.
* Combine with anti-debug, anti-dump, and regular re-keying for best results.

---

## **Summary**

This technique encrypts machine code for sensitive functions and decrypts them in memory only when needed. It requires detailed control of memory layout, code section access, and cryptography, typically in native languages (C/C++, sometimes ASM). This is a classic advanced anti-reversing method used in high-security applications and commercial protectors like CodeMeter, VMProtect, etc.


You’re asking about **self‑modifying / self‑decrypting code** at the function level. That’s not magic; it’s just very low‑level control over what bytes end up in the executable and how they’re transformed at runtime.

I’ll walk through the *actual mechanics* step by step, from compile time to runtime, without hand‑waving.

This is not about “security by obscurity.” It’s about making reverse‑engineering expensive.

---

## 0) First: what “encrypting a function” really means

A function in C++ becomes **machine code bytes** inside the `.text` section of your EXE.

So when you say:

```cpp
int main() {
    SecureFunction();
}
```

What exists in the final binary is:

```
[ main machine code bytes ]
[ SecureFunction machine code bytes ]
```

Encrypting a function means:

You **replace the raw bytes of SecureFunction in the executable with encrypted bytes**, and at runtime you:

1. change the memory protection
2. decrypt those bytes back into executable code
3. call it
4. re‑encrypt (or wipe)

Nothing more mystical than that.

---

## 1) What the compiler/linker normally does

When you build in Visual Studio:

1. **Compiler (cl.exe)**
   Turns each function into object code (`.obj`).

2. **Linker (link.exe)**
   Places the functions into the final EXE:

   * `.text` → code
   * `.rdata` → constants
   * `.data` → globals

At this point, `SecureFunction()` is just a block of bytes in `.text`.

You **cannot ask the compiler directly** to “encrypt this function” in standard C++. You must either:

* post‑process the executable, or
* design your code so that the sensitive bytes are treated as data and executed manually.

CodeMeter and VMProtect do both.

---

## 2) The high‑level strategy

There are two practical ways:

### Method A — **In‑place encryption of the real function**

* The function is compiled normally.
* After linking, you locate its byte range in the EXE and encrypt it.
* At runtime, you decrypt it in memory, execute, re‑encrypt.

### Method B — **Function stored as encrypted data blob**

* The function is compiled separately or extracted.
* Stored as encrypted data.
* At runtime, you allocate executable memory, decrypt into it, call it.

Commercial protectors usually use Method A because it preserves normal call flow and is harder to isolate.

Let’s go through **Method A**, since that’s what you asked.

---

## 3) Step 1: Make the function’s boundaries deterministic

You must know **exactly where the function starts and ends in memory**.

In C++, this is not guaranteed unless you force it.

You do that by:

* disabling inlining
* disabling optimization for that function
* optionally placing it in its own section

Example:

```cpp
__declspec(noinline)
__declspec(allocate(".secure"))
void SecureFunction() {
    // sensitive logic
}
```

And in your linker options:

```
/SECTION:.secure,ERW
```

This puts `SecureFunction` into its own section named `.secure`.

Now:

* start address = address of `SecureFunction`
* size = size of the `.secure` section

You’ve created a clean target.

---

## 4) Step 2: Build once, then encrypt the function in the EXE

After compilation, you have:

```
MyApp.exe
  .text
  .rdata
  .secure   <-- contains SecureFunction machine code
```

Now you write a **post‑build tool**:

1. Open the EXE as a binary file.
2. Parse the PE headers.
3. Find the `.secure` section.
4. Read its raw bytes.
5. Encrypt them (AES, XOR, whatever).
6. Write the encrypted bytes back into the file.

At rest on disk:

```
.secure = ENCRYPTED MACHINE CODE
```

Your EXE will crash if you try to call `SecureFunction()` now — because the CPU will try to execute garbage. That’s expected.

---

## 5) Step 3: Add a runtime decrypt‑execute‑reencrypt wrapper

Now you modify `main()` (or wherever it’s called) so it never jumps directly into encrypted bytes.

Instead:

```cpp
void CallSecure() {
    DecryptSecureSection();
    SecureFunction();
    ReEncryptSecureSection();
}
```

And:

```cpp
int main() {
    CallSecure();
}
```

---

## 6) Step 4: Runtime mechanics in memory

At runtime, Windows loads your EXE:

* `.secure` is mapped into memory as **executable but read‑only**.
* But its contents are still encrypted.

So your loader does:

### A) Change memory protection

```cpp
VirtualProtect(secure_addr, secure_size, PAGE_EXECUTE_READWRITE, &old);
```

Now the code region is writable.

### B) Decrypt in place

You run your AES/XOR over the bytes at that address:

```cpp
decrypt(secure_addr, secure_size);
```

Now the encrypted garbage becomes real machine code again.

### C) Execute

```cpp
SecureFunction();
```

### D) Re‑encrypt

```cpp
encrypt(secure_addr, secure_size);
VirtualProtect(secure_addr, secure_size, PAGE_EXECUTE_READ, &old);
```

At this point, memory no longer contains readable machine code.

---

## 7) What actually happens on the CPU

Before call:

```
Address 0x401000: 9F 2A C4 11 ...  (encrypted garbage)
```

After decrypt:

```
Address 0x401000: 55 8B EC 83 EC 08 ... (valid x86 instructions)
```

CPU executes it normally.

After re‑encrypt:

```
Address 0x401000: 9F 2A C4 11 ... (back to garbage)
```

Disassemblers see junk unless they break at exactly the right time.

---

## 8) How CodeMeter and VMProtect go further

What I described is the **minimal** version.

Commercial systems add:

* **Instruction‑level encryption** (not whole function at once)
* **Execution virtualization** (custom bytecode VM)
* **Anti‑dump hooks** (detect memory scans)
* **Call‑stack verification**
* **Self‑checksum / integrity checks**
* **Per‑call re‑keying**
* **Inline decryption stubs** at basic‑block level

But the core idea is still:

> encrypted machine code → decrypted briefly in memory → executed → erased

---

## 9) Important reality check

This protects against:

* static disassembly
* casual dumping
* basic cracking

It does **not** protect against:

* kernel debuggers
* hardware breakpoints
* runtime memory tracing
* dynamic binary instrumentation

Anyone with enough time and skill can still capture the decrypted code.

You’re **raising the cost**, not creating absolute secrecy.

---

## 10) Summary in one sentence

You “encrypt a function” by **taking its compiled machine code bytes inside the EXE, encrypting them after build, and at runtime temporarily decrypting them in executable memory, calling the function, then re‑encrypting or wiping the bytes**.

No mystery. Just controlled self‑modifying code.

---

If you want, next I can walk through:

* the exact PE section layout in Visual Studio
* how to locate and encrypt a section in a post‑build tool
* or how to hide the key using per‑machine derivation instead of hardcoding

No fluff. Just mechanisms.
Great question. With **modern ciphers like AES** (instead of simple XOR), ciphertext length is typically the **same as plaintext**, except for *padding* required by the block size (e.g., 16 bytes for AES). Here's the technical breakdown for your use-case:

---

## 1. **Block Cipher Padding:**

* AES and similar ciphers operate on fixed-size blocks (16 bytes for AES).
* If your function size is already a multiple of 16 bytes, **no padding** is needed, and encrypted data size == original.
* If not, you pad the plaintext to the next 16-byte boundary (commonly with PKCS7 padding, which can be safely stripped after decryption).

**Result:**

* Encrypted code is at most (block size − 1) bytes longer than the original.

**Practical implication:**

* You must ensure your “function region” is sized/padded to a multiple of the cipher block size (preferably at compile/link time).
* This way, the encrypted bytes will fit exactly into the region reserved for code, and you overwrite the original code with the encrypted version without overflow.

---

## 2. **How to do this safely:**

**a) Force function region alignment and size:**

* Use compiler/linker attributes to place the function in its own section.
* Round up the section size to the nearest 16 bytes.
* If the function is, say, 115 bytes, pad to 128 bytes (8 blocks).
* If you encrypt 128 bytes, decrypted code will occupy 128 bytes (even if only 115 are “real” instructions).

**b) Post-build tool logic:**

* Encrypt the entire aligned region (including the padding).
* At runtime, decrypt the region, and execution proceeds as normal.
* Extra padding at the end is harmless, as code execution never reaches it (function ends with `ret` or jumps out).

---

## 3. **What about non-block cipher (e.g., stream cipher)?:**

* With a stream cipher, no padding is needed. Output is same length as input, one-to-one.

---

## 4. **What commercial protectors do:**

* They pad the encrypted region to block size, so decryption in-place always fits.
* Some inject harmless “junk” instructions as padding.
* Function boundaries are always aligned and slightly over-provisioned.

---

## 5. **Summary table:**

| Cipher      | Output size              | Handling                                     |
| ----------- | ------------------------ | -------------------------------------------- |
| AES-128/256 | = input + up to 15 bytes | Pad function region to 16 bytes (block size) |
| XOR         | = input                  | No change                                    |
| Stream      | = input                  | No change                                    |

---

## 6. **Code Example:**

**Align function region and pad:**

```cpp
__declspec(allocate(".secure"))
__declspec(noinline)
void SecureFunction() {
    // ... code ...
    // Optionally add: __asm { nop } // for padding
}
```

**Pad section in linker:**

```
/SECTION:.secure,ERW
```

**Encrypt whole region (padded to 16-byte boundary).**

---

**TL;DR:**
Always **pad the encrypted code region** to the cipher's block size (usually 16 bytes). The encrypted and decrypted code will then always fit in the same memory space, and in-place overwrite is safe. No size mismatch issues will occur.
