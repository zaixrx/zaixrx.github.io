+++
date = '2026-06-06'
draft = false
title = 'Cheeseme'
+++

# cheeseme.org - Full Writeup

## 1. Overview

We are given a Java binary `challenge.jar`. Opening it in JADX reveals two packages: `com.update.ctf` and `internal`. The main entry point is the `com.update.ctf.Challenge`

![overview](/assets/first.png)

Looking at the `Challenge` class, we find a few interesting details:
- In the static initializer, it loads a native DLL called `challenge_native.dll`, which can be found under `Resources/native`.
- It defines three checker methods: `checkPart1`, `checkPart2`, and `checkPart3`, alongside a lot of string obfuscation and dead code.

The main validation logic looks like this:

```java
// in com.update.ctf.Challenge
public static void main(String[] strArr) {
    Caf54bdb423.dispatch_0(strArr);
}

// in com.update.ctf.Caf54bdb423
public static void dispatch_0(String[] args) {
    if (args.length != 1) {
        System.exit(2);
    }
    byte[] bytes = args[0].getBytes(StandardCharsets.UTF_8);
    if (bytes.length == 47 && 
        Challenge.checkPart1(Arrays.copyOfRange(bytes, 0, 16)) && 
        Challenge.checkPart2(Arrays.copyOfRange(bytes, 16, 32)) && 
        Challenge.checkPart3(Arrays.copyOfRange(bytes, 32, 47))) {
        System.exit(0);
    } else {
        System.exit(1);
    }
}
```

From here we can see the flag length needs to be 47 characters:
- `0..15` is checked by `Challenge.checkPart1`
- `16..31` is checked by `Challenge.checkPart2`
- `32..46` is checked by `Challenge.checkPart3` (a native function linked to the DLL)

## 2. Part 1: Simple Reversible Obfuscation

Starting with `checkPart1`, after ignoring the bogus code that wraps the core call, we find `_s$6d77aa71(byte[] bArr)`. This method applies a sequence of bitwise XORs, shifts, and modular additions on the 16-byte input array.

```java
private static boolean _s$6d77aa71(byte[] bArr) {
    int[] iArr = new int[16];
    int i = 137;
    for (int i2 = 0; i2 < 16; i2++) {
        int i3 = ((bArr[i2] ^ i) ^ 87) & 255;
        iArr[i2] = ((i3 << 1) | (i3 >>> 7)) & 255;
        i = (i + 23) & 255;
    }
    // ... further reversible transformations ...
    return ((((((((((((((((0 | (iArr[0] ^ 167)) | (iArr[1] ^ 210)) | (iArr[2] ^ 183)) | 
           (iArr[3] ^ 213)) | (iArr[4] ^ 204)) | (iArr[5] ^ 26)) | (iArr[6] ^ 54)) | 
           (iArr[7] ^ 103)) | (iArr[8] ^ 216)) | (iArr[9] ^ 169)) | (iArr[10] ^ 251)) | 
           (iArr[11] ^ 83)) | (iArr[12] ^ 72)) | (iArr[13] ^ 101)) | (iArr[14] ^ 62)) | 
           (iArr[15] ^ 183)) == 0;
}
```

Instead of reversing these operations manually, we can simply rewrite the Java logic in Python and let Z3 solve it constraint by constraint:

```python
from z3 import *

bArr = [BitVec(f'b{i}', 8) for i in range(16)]

s = Solver()

for b in bArr:
    s.add(b >= 0x20, b <= 0x7e)

iArr = [BitVec(f'i{i}', 8) for i in range(16)]

i = BitVecVal(137, 8)
for i2 in range(16):
    i3 = (bArr[i2] ^ i) ^ BitVecVal(87, 8)
    iArr[i2] = RotateLeft(i3, 1)
    i = i + BitVecVal(23, 8)
i4 = BitVecVal(16, 8)
for i5 in range(16):
    iArr[i5] = iArr[i5] + i4
    iArr[i5] = RotateLeft(iArr[i5], 2)
    i4 = i4 + iArr[i5] + BitVecVal(1, 8)
i6 = BitVecVal(11, 8)
for i7 in range(15, -1, -1):
    iArr[i7] = iArr[i7] ^ i6
    i6 = i6 + iArr[i7] + BitVecVal(i7 + 1, 8)
i8 = BitVecVal(167, 8)
for i9 in range(16):
    iArr[i9] = iArr[i9] + i8
    iArr[i9] = RotateLeft(iArr[i9], 5)
    i8 = i8 + iArr[i9] + BitVecVal(8, 8)
i10 = BitVecVal(220, 8)
for i11 in range(15, -1, -1):
    iArr[i11] = iArr[i11] ^ i10
    i10 = i10 + iArr[i11] + BitVecVal(i11 + 1, 8)
i12 = BitVecVal(255, 8)
for i13 in range(16):
    iArr[i13] = iArr[i13] + i12
    iArr[i13] = RotateLeft(iArr[i13], 4)
    i12 = i12 + iArr[i13] + BitVecVal(15, 8)
i14 = BitVecVal(131, 8)
for i15 in range(15, -1, -1):
    iArr[i15] = iArr[i15] ^ i14
    i14 = i14 + iArr[i15] + BitVecVal(i15 + 1, 8)
i16 = BitVecVal(69, 8)
for i17 in range(16):
    iArr[i17] = iArr[i17] + i16
    iArr[i17] = RotateLeft(iArr[i17], 1)
    i16 = i16 + iArr[i17] + BitVecVal(22, 8)
i18 = BitVecVal(7, 8)
for i19 in range(15, -1, -1):
    iArr[i19] = iArr[i19] ^ i18
    i18 = i18 + iArr[i19] + BitVecVal(i19 + 1, 8)
i20 = BitVecVal(31, 8)
for i21 in range(16):
    iArr[i21] = iArr[i21] + i20
    iArr[i21] = RotateLeft(iArr[i21], 5)
    i20 = i20 + iArr[i21] + BitVecVal(29, 8)
i22 = BitVecVal(86, 8)
for i23 in range(15, -1, -1):
    iArr[i23] = iArr[i23] ^ i22
    i22 = i22 + iArr[i23] + BitVecVal(i23 + 1, 8)

target = [167, 210, 183, 213, 204, 26, 54, 103, 216, 169, 251, 83, 72, 101, 62, 183]
for idx in range(16):
    s.add(iArr[idx] == BitVecVal(target[idx], 8))

if s.check() != sat:
    print("failed to find solution")
    exit(1)

model = s.model()
result = bytes(model[bArr[i]].as_long() for i in range(16))
print(result.decode())
```

Running this gives us the first part of the flag: **`THEM{h0p3_7h15_7`**

## 3. Part 2: Custom JIT and Virtual Machine

For `checkPart2`, the execution flow passes to `internal.j64mft.obmwy._s7n9c()`, which relies on `_mh_ex`, a `MethodHandle` that isn't statically defined in the class.

![part2_init](/assets/part2_init.png)

Inspecting the static initializer (`<clinit>`) of `obmwy` by dumping it's bytecode using cuz for some reason(prolly obfuscation) jadx failed to decompile it we get:

```java
.method static <clinit>()V
    .max stack 7
    .max locals 0

    invokestatic java/lang/invoke/MethodHandles lookup ()Ljava/lang/invoke/MethodHandles$Lookup;
    ldc "ajyiv7/ome04d1e/ff8zoj/bts.bin"
    ldc "internal.j64mft.obmwyxuh"
    bipush 18
    anewarray java/lang/Object
    dup
    iconst_0
    ldc "_mh_ex"
    aastore
    dup
    iconst_1
    ldc "_s7n9c"
    aastore
    dup
    iconst_2
    ldc .methodtype (Ljava/lang/Class;Ljava/lang/String;Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;
    aastore
    dup
    iconst_3
    ldc "_mh_cb"
    aastore
    dup
    iconst_4
    ldc "_nzrf"
    aastore
    dup
    iconst_5
    ldc .methodtype (Ljava/lang/Object;)Z
    aastore
    // ... (rest of the method entries)
    invokestatic internal/j64mft/a8xxf bootAndInstall (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/Class;
    pop
    return
.end method
```

what that does is:
1) build an Object array of 6 method entries where each entry is a tuple containing:
  - method name (e.g "_mh_ex", "mh_cb", ...)
  - method handle (e.g "_s7n9c", "_nzrf", ...)
  - method signature (e.g (Class, String, Object, Object[]) -> Object, ...)
2) call `internal.j64mft.a8xxf.bootAndInstall` passing `"ajyiv7/ome04d1e/ff8zoj/bts.bin"`, `"internal.j64mft.obmwyxuh"` and the `methodEntries` array, seeing this we can directly assume it uses the binary `bts.bin` to JIT compile/load custom class files under the package `internal.j64mft.obmwyxuh`

### Decrypting the Class Bundle

taking a look at `internal.j64mft.a8xxf.bootAndInstall` we can see:

![part2_loader](/assets/part2_loader.png)

it calls `boot(loader, "ajyiv7/ome04d1e/ff8zoj/bts.bin", "internal.j64mft.obmwyxuh")` that returns a class that it presumably loads from the binary, then install the previously undifined MethodHandles in `obmwy` to their correponding method handles in the `methodEntries` array (e.g _mh_ex in obmy gets assaigned _s7n9c in obmwyxuh -- the newly loaded class)

now what we should do is get the implementation of loaded class `obmwyxuh` for that we look at `boot` which does the following:

```java
byte[] binary = _msy2fq(lookup.lookupClass().getClassLoader(), "ajyiv7/ome04d1e/ff8zoj/bts.bin");
if (binary == null || binary.length < 28) {
    throw new IllegalStateException("rt.bin missing or truncated: " + "ajyiv7/ome04d1e/ff8zoj/bts.bin");
}
byte[] header = new byte[12];
System.arraycopy(binary, 0, header, 0, 12);
byte[] payload = new byte[binary.length - 12];
System.arraycopy(binary, 12, payload, 0, payload.length);
byte[] key = new byte[16];
System.arraycopy(ajyiv._bbav2("vm-pack-v1".getBytes(StandardCharsets.UTF_8)), 0, key, 0, 16);
Class<?> cls = null;
Iterator<Map.Entry<String, byte[]>> it = _mzcrq6(o2oah.aesGcmDecrypt(key, header, payload)).entrySet().iterator();
while (it.hasNext()) {
    Class<?> clsDefineClass = lookup.defineClass(it.next().getValue());
    if (clsDefineClass.getName().equals("internal.j64mft.obmwyxuh")) {
        cls = clsDefineClass;
    }
}
if (cls == null) {
    throw new IllegalStateException("packed host not found: " + "internal.j64mft.obmwyxuh");
}
return cls;
```

basically it loads the binary which is an encrypted archive containing all class objects, decrypts it, and defines each class it contains returning the one named `internal.j64mft.obmwyxuh`, here is the script I used to extract all those class objects for further analysis:

```java
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.lang.reflect.Method;
import java.io.InputStream;

public class Dumper {
    public static void main(String[] args) throws Exception {
        InputStream is = Dumper.class.getClassLoader().getResourceAsStream("ajyiv7/ome04d1e/ff8zoj/bts.bin");
        byte[] bts = is.readAllBytes();
        
        byte[] header = new byte[12];
        System.arraycopy(bts, 0, header, 0, 12);
        
        byte[] payload = new byte[bts.length - 12];
        System.arraycopy(bts, 12, payload, 0, payload.length);
        
        Class<?> ajyivClass = Class.forName("internal.j64mft.ajyiv");
        Method bbav2 = ajyivClass.getDeclaredMethod("_bbav2", byte[].class); bbav2.setAccessible(true);
        byte[] keyBase = (byte[]) bbav2.invoke(null, "vm-pack-v1".getBytes(StandardCharsets.UTF_8));
        byte[] key = new byte[16];
        System.arraycopy(keyBase, 0, key, 0, 16);
        
        Class<?> o2oahClass = Class.forName("internal.j64mft.o2oah");
        Method aesGcmDecrypt = o2oahClass.getDeclaredMethod("aesGcmDecrypt", byte[].class, byte[].class, byte[].class); aesGcmDecrypt.setAccessible(true);
        byte[] decrypted = (byte[]) aesGcmDecrypt.invoke(null, key, header, payload);
        
        Files.write(Paths.get("rt.bin"), decrypted);
        System.out.println("Wrote " + decrypted.length + " bytes to .\\rt.bin");

        java.nio.ByteBuffer bb = java.nio.ByteBuffer.wrap(decrypted);
        bb.getInt(); // checksum 1246712321
        int size = bb.getInt();
        System.out.println("Total classes: " + size);
        for (int i = 0; i < size; i++) {
            int nameLen = bb.getShort() & 0xFFFF;
            byte[] nameBytes = new byte[nameLen];
            bb.get(nameBytes);
            String name = new String(nameBytes, StandardCharsets.UTF_8);
            
            int classLen = bb.getInt();
            byte[] classBytes = new byte[classLen];
            bb.get(classBytes);
            
            String[] parts = name.split("/");
            String outClassName = parts[parts.length - 1] + ".class";
            Files.write(Paths.get(outClassName), classBytes);
            System.out.println("Wrote " + classLen + " bytes to .\\" + outClassName);
        }
    }
}
```

which yield the following class files which we add to jadx for further analysis:
```
Wrote 61709 bytes to .\rt.bin
Total classes: 7
Wrote 30230 bytes to .\obmwyxuh.class
Wrote 13816 bytes to .\obmwyxuh$ffl.class
Wrote 4203 bytes to .\obmwyxuh$zpo.class
Wrote 1941 bytes to .\obmwyxuh$apw.class
Wrote 607 bytes to .\obmwyxuh$a42.class
Wrote 9943 bytes to .\obmwyxuh$hyw.class
Wrote 727 bytes to .\obmwyxuh$msk.class
```


### The Part 2 Custom VM

Now that we have successfully decrypted and dumped the class files, the next logical step in our reverse-engineering process is to analyze the `obmwyxuh.class` file. We know from `bootAndInstall` that this class contains the real implementation of the previously undefined method handles.

Opening `obmwyxuh.class` in our decompiler, we look for the implementations of `_s7n9c` and `_nzrf`, which were the handlers mapped for `checkPart2`. The decompilation reveals:
```java
private static final /* synthetic */ Map<String, ffl> _fy7aqc = new ConcurrentHashMap();

public static /* synthetic */ Object _s7n9c(Class<?> cls, String str, Object obj, Object[] input) {
    return _fy7aqc.computeIfAbsent(cls.getName() + '|' + "ajyiv7/ome04d1e/ff8zoj/gmv3/175018dca521bdad4d58.bin", str2 -> {
        return _mau4l3(cls, "ajyiv7/ome04d1e/ff8zoj/gmv3/175018dca521bdad4d58.bin");
    })._mafig8(cls, null, input);
}

public static /* synthetic */ boolean _nzrf(Object obj) {
    if (obj instanceof Boolean) {
        return ((Boolean) obj).booleanValue();
    }
    if (obj instanceof Number) {
        return ((Number) obj).intValue() != 0;
    }
    throw new ClassCastException("tx625832");
}
```

As seen in `_s7n9c`, if the method hasn't been cached in `_fy7aqc`, it first reads `ajyiv7/ome04d1e/ff8zoj/gmv3/175018dca521bdad4d58.bin` by calling `_mau4l3`.

the `_mau4l3` method acts as a parser/loader for this binary file. It reads various sections such as constants (`_fk60s7`), exception handlers (`_faap7d`), and most importantly, bytecode instructions, initializing them as an array of `a42` objects (`_fapwvn`). It packages all this into an instance of the inner class `ffl` and returns it.

Once the `ffl` object is created and cached, `_s7n9c` immediately invokes `ffl._mafig8(cls, null, input)`, passing the input (the second chunk of the flag) to it.

If we examine `obmwyxuh$ffl._mafig8` in the decompiled code, we find a classic Virtual Machine interpreter loop:

![custom_vm](/assets/part2_custom_vm.png)

```java
hyw hywVar = new hyw(this._fwau0n, this._fafgs3);
// ... [loads arguments into hywVar] ...
while (hywVar._fqzgw7 >= 0 && hywVar._fqzgw7 < this._fapwvn.length) {
    // ...
    a42 a42Var = this._fapwvn[hywVar._fqzgw7];
    // ...
    switch (((a42Var._fa9yk8 * 1247066437) ^ 2095681396) & 255) {
        case 1:
            return hywVar._mla67o(a42Var._fango5[0]);
        case 5:
            hywVar._msxy4c(hywVar._mqkeq7() ^ hywVar._mqkeq7());
            break;
        case 7:
            hywVar._mszhjd(hywVar._moh9cy(a42Var._fango5[0]) & ((Integer) this._fk60s7[a42Var._fango5[1]]).intValue());
            break;
        // ... [over 200 cases representing various VM opcodes] ...
    }
    hywVar._fqzgw7++;
}
```

This reveals that the `ffl` class is a custom virtual machine evaluating the second chunk of the flag. The `hyw` class is the VM context (acting as the operand stack and local variables), and the huge `switch` statement dispatches the opcodes parsed from the `.bin` file.

To understand the flag validation logic, I traced the VM execution of the instructions loaded from `175018dca521bdad4d58.bin`. The VM implements 5 rounds of transformations. Each round consists of forward and backward passes over the 16-byte state array. The passes mix the array through:
- Rotations (using a custom `rotr`/`rotl` bitwise shift on 8 bits)
- Modular additions
- XOR operations

These operations use a hardcoded array of constants (`CTES`). Because all the operations are perfectly reversible, we can write a direct solver in Python that implements the inverse functions (`inv_bwd` and `inv_fwd`) and runs the rounds in reverse:

```python
CTES = [16, 0, 182, 26, 255, 3, 5, 20, 157, 2, 6, 1, 21, 15, 105, 4, 8, 94, 227, 223, 237, 22, 42, 35, 29, 61, 185, 221, 253, 73, 130, 201, 138, 7, 18, 9, 184, 10, 100, 11, 74, 12, 13, 14, 86, 178]
TARGET = [185, 221, 253, 73, 130, 201, 138, 18, 61, 184, 100, 74, 5, 13, 86, 178]

ROUNDS = [
    ((CTES[8], CTES[9], CTES[10], CTES[11]), CTES[12]),
    ((CTES[14], CTES[15], CTES[15], CTES[16]), CTES[17]),
    ((CTES[18], CTES[10], CTES[9], CTES[13]), CTES[19]),
    ((CTES[20], CTES[15], CTES[15], CTES[21]), CTES[22]),
    ((CTES[23], CTES[6], CTES[5], CTES[24]), CTES[25]),
]

def rotr(v, n):
    return (((v << (n & 7)) | (v >> (8 - (n & 7)))) & 0xFF) if (n & 7) else v & 0xFF

def inv_fwd(s, a, rl, add):
    s, acc = list(s), a
    for i in range(16):
        orig = (rotr(s[i], rl) - acc) & 0xFF
        acc = (acc + s[i] + add) & 0xFF
        s[i] = orig
    return s

def inv_bwd(s, a):
    s, acc = list(s), a
    for i in range(15, -1, -1):
        out_i = s[i]
        s[i] = (out_i ^ acc) & 0xFF
        acc = (acc + out_i + i + 1) & 0xFF
    return s

s = list(TARGET)
for (a, rl, rr, add), ba in reversed(ROUNDS):
    s = inv_bwd(s, ba)
    s = inv_fwd(s, a, rl, add)

flag = []
key = CTES[2]
for i in range(16):
    flag.append((rotr(s[i], CTES[5]) ^ key ^ CTES[3]) & 0xFF)
    key = (key + CTES[7]) & 0xFF
print("Part 2:", "".join(chr(b) for b in flag))
```

Running this solver yields the second part: **`1m3_7h3_1nfr4_15`**

## 4. Part 3: Native DLL

The last 15 bytes are checked by:

```java
static native boolean checkPart3(byte[]);
```

The DLL registers the native method through `JNI_OnLoad`. After importing a [jni.h](https://gist.githubusercontent.com/Jinmo/048776db75067dcd6c57f1154e65b868/raw/89af26807eaa8bb31e35da63e102b0abfa311580/jni_all.h) header which was a fucking life saver, here:

```c
uint JNI_OnLoad(JavaVM *vm) {
    JNIEnv *env = NULL;
    (*(*vm)->GetEnv)(vm, &env, JNI_VERSION_1_6);
    jclass clazz = (*(*env)->FindClass)(env, "com/update/ctf/Challenge");
    (*(*env)->RegisterNatives)(env, clazz, &nativeMethod, 1);
}
```

The `JNINativeMethod` struct at `0x202804030` maps:

```
checkPart3([B)Z  ->  0x202801390
```

That function only jumps into the real checker:

```asm
0x202801390:  JMP  0x2028b7800
```

The native checker is another VM. Following the thunk at `0x2028B7800`, it loads three key addresses into registers before jumping to the main prologue:

```asm
2028b7800:  LEA  R10, [0x2028B9000]    ; bytecode stream
2028b7807:  LEA  R11, [0x2028B7000]    ; handler table (256 QWORDs)
2028b780e:  LEA  RAX, [0x202820000]    ; dispatcher function
2028b7815:  JMP  0x2028AF217           ; prologue
```

A useful 15-byte constant appears in `.rdata` at `0x202804048`, immediately after the `JNINativeMethod` struct:

```
4d 3f c7 f5 3b 16 75 48 91 ab 67 ed 06 22 f4
```

This length matches the missing suffix length exactly 15 bytes, so in case the transformation doesn't change the size of the input, it could probably be a comparison target.

### Prologue and VM State Setup

the prologue at `0x2028AF217` sets up the main VM state. It allocates a large chunk of memory and sets `R12` as the base pointer for the VM's internal context (which holds the virtual registers, flags, and configuration). 

it also dervies a byte key that's used later to decrypt bytecode:

```asm
; Compute image base via RIP-relative addressing
LEA  RCX, [RIP]              ; RCX = current PC
MOV  R8,  0xAF59B
SUB  RCX, R8                 ; RCX = image base (PE_BASE)

; Derive key from bytecode/handler table offsets
MOV  R8,  R10                ; R10 = bytecode VA
SUB  R8,  RCX                ; R8  = bytecode RVA = 0xB9000
MOV  RAX, R11                ; R11 = handler table VA
SUB  RAX, RCX                ; RAX = handler table RVA = 0xB7000
SHL  RAX, 0x20               ; shift to upper 32 bits
XOR  R8,  RAX                ; combine
XOR  R8,  0x18F8A05A733562A1 ; constant mix
ROL  R8,  0x11               ; rotate left 17
XOR  R8,  0xD6E8FEB86659FD93 ; R8 = 0x74

MOVZX R8, R8B                ; zero-extend to byte = 0x74
MOV  RAX, 0x0101010101010101
IMUL R8,  RAX                ; replicate across QWORD
MOV  [R12+0x19A8], R8        ; Store 0x7474747474747474 in VM state
```

### Tracing the Native VM

With the VM state initialized, execution enters the main dispatcher loop at `0x202820087`:

```asm
0x202820087: MOV EAX, 0
0x20282008C: MOV AL, byte ptr [RSI]       ; fetch opcode byte
0x20282008E: INC RSI                      ; skip opcode
0x202820091: INC RSI                      ; skip operand
0x202820094: CMP EAX, 0x100
0x202820099: JNC ud2                      ; crash on invalid opcode
0x20282009F: MOV R11, qword ptr [R12]     ; reload handler table pointer
0x2028200A3: MOV RDX, qword ptr [R11+RAX*8] ; fetch handler address
0x2028200A7: CALL RDX                     ; execute handler
0x2028200A9: JMP 0x202820087              ; loop to next instruction
```

I loaded the DLL into x64dbg and generated an instruction trace of the VM execution with `abcdefghijklmno`, and the trace logged:
- the bytecode offset (`RSI-PIE_BASE-0xB9000`)
- The opcode and operand bytes (`[RSI-2]`)
- Which handler address was called (`RDX`)
- The values of all VM registers (stored within the `R12` state block)
- the xor key at `[R12+0x19A8]`

![vm_trace](/assets/part3_vm_trace.png)

### On-the-Fly Bytecode Decoding

One thing I noticed from the trace is that the bytecode is encrypted:

1. **Opcodes** are not decrypted; the dispatcher reads the raw byte and indexes into the handler table directly. The table is pre-shuffled so `table[raw_byte]` already points to the right handler.

2. **Operands** are XORed with a single-byte key at read time. Every handler that needs an operand does `XOR reg, byte ptr [R12+0x19A8]` before using it.

### Handler Classification

256 entries in the handler table at `0x2028B7000`, each an 8-byte pointer into the handler code section. I wrote a script to disassemble every unique handler and classify it by instruction patterns:

```python
import struct, os
from capstone import Cs, CS_ARCH_X86, CS_MODE_64
import pefile

DLL_PATH = os.path.join(os.path.dirname(__file__), "native", "challenge_native.dll")
PE_BASE = 0x202800000
HANDLER_TABLE = 0x2028B7000

pe = pefile.PE(DLL_PATH)
img = pe.get_memory_mapped_image()
rva = lambda va: va - PE_BASE

handler_table = {}
ht_raw = img[rva(HANDLER_TABLE):]
for opcode in range(256):
    handler_table[opcode] = struct.unpack_from("<Q", ht_raw, opcode * 8)[0]

md = Cs(CS_ARCH_X86, CS_MODE_64)

def disasm_handler(va, max_bytes=0x300):
    code = img[rva(va):rva(va) + max_bytes]
    insns = []
    for i in md.disasm(code, va):
        insns.append(i)
        if i.mnemonic in ("ret", "ud2", "jmp"):
            break
    return insns

def classify_handler(va):
    insns = disasm_handler(va)
    text = "\n".join(f"{i.mnemonic} {i.op_str}" for i in insns)

    add_rsi = [i for i in insns if i.mnemonic == "add" and "rsi" in i.op_str]
    if len(insns) <= 10 and add_rsi:
        return "NOP"

    has_rdtsc = any(i.mnemonic == "rdtsc" for i in insns)
    vm_reg_write = "[r12 +" in text and "*8 + 8]" in text
    has_lock = any(i.bytes[0] == 0xF0 for i in insns if len(i.bytes))
    has_jcc = "popfq" in text and any(k in text for k in ["jo ","jb ","jae ","je ","jne ","ja ","js "])
    has_imul = any(i.mnemonic == "imul" for i in insns)
    has_cmp = any(i.mnemonic in ("test","cmp") and "*8" in i.op_str for i in insns) and "1a08" in text

    if has_jcc: return "JCC"
    if has_lock: return "ATOMIC"
    if has_imul and vm_reg_write: return "IMUL"
    if has_cmp: return "CMP"
    if has_rdtsc and vm_reg_write: return "MOV_KEY"
    if "[r14]" in text: return "LOAD"
    if vm_reg_write: return "ARITH"
    if has_rdtsc: return "RDTSC_NOP"
    return "UNKNOWN"

cats = {}
for va in sorted(set(handler_table.values())):
    cats.setdefault(classify_handler(va), []).append(va)
for cat, vas in sorted(cats.items()):
    print(f"  {cat:10s}: {len(vas):3d} handlers")
```

Output:

```
  ARITH     :   1 handlers
  IMUL      :   7 handlers
  NOP       :  81 handlers
  RDTSC_NOP :  72 handlers
  UNKNOWN   :  95 handlers
```

Most of the 256 handlers are junk, NOPs that just advance `RSI` or RDTSC anti-debug wrappers that do nothing useful. The actual work (ARITH, IMUL, LOAD, CMP, JCC) is buried in the UNKNOWN bucket and the few classified ones, but only a handful fire on any given execution path.

Every handler is also bloated with obfuscation:

- **Opaque predicates** : XOR/XOR pairs that cancel, followed by a `CMP` + `JE` that always jumps to the real code
- **RDTSC timing checks** : measure elapsed cycles, `PAUSE` if too slow (anti-debug but doesn't actually halt)
- **Identity transforms** : `NOT/NOT`, `NEG/NEG`, `ROL N/ROR N` on scratch registers

All of this is irrelevant to the actual computation.

### Simplifying the whole VM

From the trace, the VM's actual execution path for the 15 input bytes follows a repeating pattern:
1. **LOAD** : read one byte from the input buffer into a VM register
2. **XOR** : XOR that register with another register holding a constant
3. **CMP** : compare the result against a target value from `.rdata`
4. **JCC** : branch to fail if mismatch

The transform is just `input[i] ^ key[i] == target[i]` for each of the 15 bytes. The per-byte XOR constants can be read directly from the VM register state in the trace at the point of each XOR instruction:

```
XOR key:  12 0a f0 87 0b 78 43 7b e3 f4 56 de 35 15 89
```

The 15-byte comparison target from `.rdata` at `0x202804048`:

```
target:   4d 3f c7 f5 3b 16 75 48 91 ab 67 ed 06 22 f4
```

### Solving

Since it's a straight XOR, the flag suffix is just `target[i] ^ key[i]`:

```python
from z3 import *

target = bytes.fromhex("4d3fc7f53b16754891ab67ed0622f4")
key    = bytes.fromhex("120af0870b78437be3f456de351589")

n = 15
x = [BitVec(f"x{i}", 8) for i in range(n)]

s = Solver()
for c in x:
    s.add(c >= 0x20, c <= 0x7e)
s.add(x[-1] == ord("}"))

for i in range(n):
    s.add((x[i] ^ key[i]) == target[i])

assert s.check() == sat
m = s.model()
print(bytes([m.evaluate(x[i]).as_long() for i in range(n)]))
```

**Part 3 suffix: `_57r0n63r_1337}`**

## 5. Final Flag

Combining the three recovered chunks:

```
part1 = THEM{h0p3_7h15_7
part2 = 1m3_7h3_1nfr4_15
part3 = _57r0n63r_1337}
```

**`THEM{h0p3_7h15_71m3_7h3_1nfr4_15_57r0n63r_1337}`**

