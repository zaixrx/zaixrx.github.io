+++
date = '2026-08-09'
draft = false
title = 'L3ak Omega MIPS Devirtualization'
+++

# Challenge
l3ak ctf 2026 had alot of interesting reverse challenges, I only really looked at this challenge and that was after the ctf ended because I couldn't play it in time, the challenge comes with a README.md and a binary called **executor**
```sh
$ ./executor
usage: ./executor <job.prx>
```

it's asking for a `prx` file which is a custom binary format :(, the README also says that the goal is to read `/challenge/flag.txt` in the remote instance running the executor, so probably I need to let the prx file do that

I am also given two files ./echo.local.prx and ./echo.remote.prx, running a small bindiff I see that they are mostly the same besides the part from 0x10 to 0x30... interesting
```sh
vimdiff <(xxd ./echo.local.prx) <(xxd echo.remote.prx)
```
![overview](/assets/l3ak_rev_omega/vimdiff.png)

I can also run the local file passing the secret in as an env variable, but I can't do that with the remote file, possibly because I am missing the remote secret, so the previous bindiff result is prolly cuz of the difference in the secrets
![overview](/assets/l3ak_rev_omega/echo_local_run.png)

# Main Function
I opened executor in ghidra, and the main function looks like it's reading a secret either from the environment or a file `.env` represented in hex, it also reads the `job.prx` you pass as an argument
![overview](/assets/l3ak_rev_omega/secret_3.png)

it then forks and opens a pipe, the child process calls a function `exec_job` that is passed a global variable and a quite large number `0xa80` which pretty much looks like bytecode, the parent process also sends the secret and the prx file to the child process via the previously opened pipe
![overview](/assets/l3ak_rev_omega/fork_4.png)

it then waits for the child to finish, extract the exit status and if it's zero it also runs `exec_job` on the prx file, if not it exits with a message `[-] Verification failed\n`
![overview](/assets/l3ak_rev_omega/exec_prx_5.png)

which means the binary is running some sort of validator passing in both the secret and the payload, if that suceeds it runs the provided prx file which is what I supposed to craft to read the flag

the `exec_job` function looks like this:
```c
u32 exec_job(u8 *job, u64 job_size) {
  i32 parse_stauts;
  u32 ret_val;
  u8 vm_handle[0xAC];
  
  vm_init(vm_handle);
  parse_stauts = vm_parse_prx(vm_handle,job,job_size);
  if (parse_stauts == 0) {
    vm_set_sp(vm_handle);
    vm_loop(vm_handle);
    vm_free(vm_handle + 0x90);
    ret_val = *(u8*)(vm_handle + 0xa8);
  }
  else {
    vm_free(vm_handle + 0x90);
    ret_val = -1;
  }
  return ret_val;
}

void vm_loop(u8 *vm_handle) {
  if (*(i32*)(vm_handle + 0xa4) != 0) {
    return;
  }
  do {
    vm_run(vm_handle);
  } while (*(i32*)(vm_handle + 0xa4) == 0);
  return;
}
```

there is a vm_handle allocated in the stack, that get's initialized, used to parse prx file(prolly loads it in memory), then executes the bytecode in `vm_loop`

# PRX Binary Format
understanding the binary format is really important as it will help us write the exploit later, to explain it I find it enough to only give a byte offset table, with a short description for each field:
```
0x0000000 magic          // b"PRX\x00\x01", 5 bytes (includes version)
0x0000005 section_count  // number of section entries
0x0000006 flags          // does some nieche shit
0x0000007 ...            // works with reg28(used for error indication ig)
0x0000008 entry_ip       // initial ip / next_ip-4
0x000000C reg28          // only written if flags & 1
...
0x0000030 phdrs[]        // section_count * 16-byte entries
```

```
0x0000000 start          // offset into file, section data start
0x0000004 page_base      // destination page address
0x0000008 size           // section size, 0xffffffff = rest of file
0x000000C something      // don't need that
```

# VM Architecture

the VM is register based, with a whooping 32 4-byte registers each, and here is what I defined the struct as:
```
0x0000000 [r0-r28]       // GPRs
0x0000074 r29            // Stack Pointer
0x0000078 r30            // GPR
0x000007C r31            // Link Register
0x0000080 ip             // Intruction Pointer
0x0000084 next_ip        // Next Intruction Pointer
...
0x0000090 pages_ptr      // 64-bit Pointer To An Array Of Pages
...
0x00000A4 is_stopped     // Is The VM Stopped
0x00000A8 exit_status    // VM `exit` Status, If 0 Success Else Failure
...
```
those are all the fields I needed to solve the challenge, `pages_ptr` is interesting because each page is allocated seperatly, and their pointers are stored in an array pointed to by `pages_ptr`(each page is 4096 bytes), `exit_status` is the value that is checked by the parent process to see if the validator succeeded

the VM comes with some very mid opcodes nothing special, and I didn't have to reverse them all, so I only took care of the ones that needed to be executed for the valiadtor to function correctly:

- { MOVB, MOVW, MOVD } { [expression], reg_dest }, { [expression], reg_src }:
either loads data from the address being the value of `expression` to `reg_dest`, or moves data from `reg_src` to the address being the value of `expression`, the load/move can be of a byte, a word or a dword respectively

- { ADD, SUB, AND, OR, XOR, NOR, SHL, SHR, LESS } reg_dest, reg_lhs, reg_rhs
perform the above binary operations on `reg_lhs` and `reg_rhs` and stores the result in `reg_dest`

- { ADD, OR, AND, LESS } reg_dest, reg_lhs, imm
perform the above binary operations on `reg_lhs` and `imm` and stores teh result in `reg_dest`, the immediate is constructed from it's corresponding bit range inside the opcode(which doesn't have to be contiguous), then besides for AND and OR it's masked to 16-bits and sign-extended 

- { SHL, SHR } reg_dest, reg_lhs, imm5
takes lowest bits 27-31 from opcode and performs a left or right shift of `reg_lhs` with `imm5`, the result is stored inside `reg_dest`

- MOV reg_dest, imm
loads a constructed immediate `imm`(depending on it's bit-range in the opcode) to the top 16-bits of `reg_dest`

- JMP reg
schedule a jump to instruction pointed to by `reg`

- JE, JNE, JLE imm
always following a CMP instruction, the result will deduce whether or not to jump to instruction pointed to by `imm`

- JMP imm
always following a `MOV r31 imm` instruction together they form a function call, `r31` represents what is known as **LR** or Link Register which holds the address the called function should return to when it exits with a `JMP r31` instruction

- IODispatch
you can think of it as a syscall, it traps out of the vm_loop into x64 to execute an IO call, more details later

# Emulator & Disassembler
I first started with the emulator as it made be discover a lot of nuances inside the ISA, details that I wouldn't even need to solve the challenge lol, so first I started with the `0x6` branch instructions as they looked the most easy to reverse and would be a good first step

take this mid-ass handler which compares two registers and stores the result in another register in case `opcode & 0x3e0000 != 0` otherwise it's just a NOP:
```c
    case 0x1c:
      retval = vm_instance[0x29];
      if ((opcode & 0x3e0000) != 0) {
        vm_instance[opcode >> 0x11 & 0x1f] =
             (uint)((uint)vm_instance[opcode >> 0x16 & 0x1f] < (uint)vm_instance[opcode >> 6 & 0x1f]
                   );
      }
      break;
```

and that, which sets the next_ip instruction effectively **scheduling** a jump, and I said **scheduling** cuz the mfs who designed MIPS made it so that any jump is delayed with one instruction:
```c
    case 0x29: // JMP
      retval = vm_instance[0x29];
      vm_instance[0x21] = vm_instance[opcode >> 0x16 & 0x1f];
      break;
```

moving forward in the `0x6` branch, I stubbmled upon case `0x11` that calls to a function I renamed `handle_syscall`, which provides some IO primitives, meaning I can use that later to open and read the flag... cool.
![overview](/assets/l3ak_rev_omega/handle_syscall.png)

the "syscalls" it provides are:
*(go see [man7](https://man7.org/linux/man-pages/man2) if you don't like my explanation)*
- `open(path=R4, flags=R5, perms=R6)` opens the file in `path` with `flags` and `perms`(depends on `flags`)
- `read(fd=R4, data=R5, data_size=R6)` reads `data_size` bytes from `fd` and stores them in `data`(in-VM pointer), returns number of bytes read
- `write(fd=R4, data=R5, data_size=R6)` writes `data_size` bytes from `data`(in-VM pointer) to the file descriptor `fd`, returns number of bytes written
- `exit(exit_status=r4)` exists with `exit_status`

the return value is stored inside `r2`, and an error indicator boolean inside `r28`

after that I went to the top-level branch and reversed the remaining instructions, which were mostly memory reads, writes and conditional jumps

here is an illustrative example:
```c
  case 0x11:
    write_byte(vm_instance,
               (int)(short)((ushort)((opcode >> 0x11) << 0xb) | (ushort)((opcode >> 0x1b) << 6) |
                           opcode_ & 0x3f) + vm_instance[opcode >> 0x16 & 0x1f],
               *(undefined1 *)(vm_instance + (opcode >> 6 & 0x1f)));
    retval = vm_instance[0x29];
    break;
```

the write_byte function takes the third argument `a3` and writes it into the second argument `a2`(in-VM address) as follows:
```c
memory = (*(u64**)vm.pages_ptr)[a2 >> 0xC];
if memory == NULL {
    (*(u64**)vm.pages_ptr)[a2 >> 0xC] = memory = malloc(0x1000);
}
*(u32*)(memory + (a2 & 0xfff)) = a3;
```

after I finished the vm loop, I went back and implmented prx parsing which handles loading the progam's sections in memory(e.g .data, .text), and boom a full emulator... of course it wasn't as easy as that there were some bugs, and to make sure I have **absolutely no bugs**(my way of saying I got carried away with programming), I built a gdb script that tracks the VMs state after each execution of any instruction in sync with the native executor, and stop whenever there is a mismatch, here it is:
```py
import gdb
import hashlib

from main import VM

inst_count = 0
pages = set()
vm = VM(open("./validator.bin", "rb").read())

class NewPageLogger(gdb.Breakpoint):
    def stop(self):
        addr = gdb.parse_and_eval("(unsigned int*)($rsi)")
        pages.add(int(addr) >> 0xC)
        return False

class OpcodeLogger(gdb.Breakpoint):
    def stop(self):
        global inst_count
        is_buggy = False

        try:
            vm.vm_exec()
            if vm.ret_val != 0:
                print("=== FINISHED ===")
        except Exception as e:
            print("CRASHED", e)
            is_buggy = True

        inst_count += 1
        ip = gdb.parse_and_eval("*(unsigned int*)($rbx + 0x80)")
        next_ip = gdb.parse_and_eval("*(unsigned int*)($rbx + 0x84)")
        regs = [gdb.parse_and_eval(f"*(unsigned int*)($rbx + {i * 4})") for i in range(0x20)] 

        bin_pages = {}
        inferior = gdb.inferiors()[1]
        for page in pages:
            memory = int(gdb.parse_and_eval(f"*(unsigned long*)(*(unsigned long*)($rbx+0x90)+{page}*8)"))
            assert memory != 0
            bin_pages[page] = hashlib.md5(inferior.read_memory(memory, 0x1000)).hexdigest()
        vm_pages = {}
        for page in vm.pages:
            vm_pages[page] = hashlib.md5(bytes(vm.pages[page])).hexdigest()

        expected = (hex(ip), hex(next_ip), " ".join([f"r{i}: {hex(regs[i])}" for i in range(0x20)]))
        found = (hex(vm.ip), hex(vm.next_ip), " ".join([f"r{i}: {hex(vm.regs[i])}" for i in range(0x20)]))

        is_buggy |= (expected != found) or (bin_pages != vm_pages)
        if is_buggy:
            print("=== BUG FOUND ===")
            print(f"after {inst_count} instructions")
            print(f"with opcode {vm.extract_opcode(ip):x}") # that may fail in the case opcode got overwritten with some garbage
            print(expected)
            print(found)

            print("=== PAGES ===")
            print(bin_pages)
            print(vm_pages)

        return is_buggy or vm.ret_val != 0

""" after I finished everything I ran that to make sure diff is none
class OpcodeLogger(gdb.Breakpoint):
    def stop(self):
        ip = gdb.parse_and_eval("*(unsigned int*)($rbx + 0x80)")
        next_ip = gdb.parse_and_eval("*(unsigned int*)($rbx + 0x84)")
        regs = [gdb.parse_and_eval(f"*(unsigned int*)($rbx + {i * 4})") for i in range(0x20)] 
        expected = (hex(ip), hex(next_ip), " ".join([f"r{i}: {hex(regs[i])}" for i in range(0x20)]))
        found = (hex(vm.ip), hex(vm.next_ip), " ".join([f"r{i}: {hex(vm.regs[i])}" for i in range(0x20)]))

        print(hex(ip), hex(next_ip), " ".join([f"r{i}: {hex(regs[i])}" for i in range(0x20)]))

        return False
"""

gdb.execute("set follow-fork-mode child")
gdb.execute("set logging file trace.log")
gdb.execute("set logging enabled")

# before an instruction get's executed
OpcodeLogger(f"*0x555555554000+{0x1d0c}")

# all places where a new page can be allocated
NewPageLogger(f"*0x555555554000+{0x2d50}")
NewPageLogger(f"*0x555555554000+{0x2dd0}")
NewPageLogger(f"*0x555555554000+{0x2ed0}")
NewPageLogger(f"*0x555555554000+{0x3030}")
NewPageLogger(f"*0x555555554000+{0x3130}")

gdb.execute("r")
```
as you can see I tracked all registers, memory and program counters, that made me able to catch a lot of bugs and achieve and 1:1 validator execution with the native executor... great

finally I wrote a `disasm_inst` method which was a bunch of prints so nothing special about that... which helped write a tracer and a disassembler.

*you can find all the code [here](TODO)*

# The Validation
finally I am in the validator, the entry point is at 0x400244 and it calls to two functions:
```
400234: ADD r2 0xfa1, r0
400238: IODispatch(op=r2) // calls exit
40023C: JMP r31 // ret
400240: NOP

// r29 = sp
ENTRY POINT:
400244: ADD r29 0xffffffe8, r29 // setup a stack frame
400248: MOVD [r29 + 0x14], r31 // save register
40024C: MOV r31, 0x400258; JMP 0x400080 // calls to main
400250: NOP
400254: MOV r31, 0x400260; JMP 0x400234  // tailcall exits the VM
400258: OR r4 r2 r0
40025C: MOVD r31, [r29 + 0x14] // recover saved register
400260: NOP
400264: JMP r31
400268: ADD r29 0x18, r29 // pop stack frame
40026C: NOP
```
just remember that jump instructions are delayed with one instruction.

the main function then takes control, it's also worth noting that:
- r0 is always zero
- calling convention: arguments are passed in order in r4, r5, r6... and return value in r2
```
sub_400080: main
// prologue
400080: ADD r29 0xfffffc28, r29
400084: MOVD [r29 + 0x3d4], r31
400088: MOVD [r29 + 0x3d0], r18
40008C: MOVD [r29 + 0x3cc], r17
400090: MOVD [r29 + 0x3c8], r16
// read secret_size
400094: ADD r5, 0x4, r0
400098: MOV r31, 0x4000a4; JMP 0x400000 // read secret size, 4-bytes
40009C: ADD r4 0x10, r29
4000A0: CMP r2 r0; JNE 0x400204 # if read == 0 exit
4000A4: ADD r2 0x2, r0
4000A8: MOVB r16, [r29 + 0x11]
4000AC: NOP
4000B0: SHL r16 r16 0x8
4000B4: MOVB r2, [r29 + 0x12]
4000B8: NOP
4000BC: SHL r2 r2 0x10
4000C0: OR r16 r16 r2
4000C4: MOVB r2, [r29 + 0x10]
4000C8: NOP
4000CC: OR r16 r16 r2
4000D0: MOVB r2, [r29 + 0x13]
4000D4: NOP
4000D8: SHL r2 r2 0x18
4000DC: OR r16 r16 r2 // construct secret size as 4-byte number
4000E0: LESS r2 r16 0x101
4000E4: CMP r2, r0; JE 0x400204 // make sure secret size is < 0x100
4000E8: ADD r2 0x2, r0
// read secret
4000EC: OR r5 r16 r0
4000F0: MOV r31, 0x4000fc; JMP 0x400000 // read the secret
4000F4: ADD r4 0x14, r29
4000F8: CMP r2 r0; JNE 0x400204 // if secret not read exit
// sha256 init context
4000FC: ADD r2 0x2, r0
400100: MOV r31, 0x40010c; JMP 0x400500 // sha256 init constants, in r20+0x358
400104: ADD r4 0x358, r29
400108: OR r6 r16 r0
// sha256 buffer secret (if it's not a full block)
40010C: ADD r5 0x14, r29
400110: MOV r31, 0x40011c; JMP 0x400578 // schedule sha256 until a full block
400114: ADD r4 0x358, r29
400118: ADD r5 0x4, r0
// read full file size
40011C: MOV r31, 0x400128; JMP 0x400000 // read file size, 4-bytes
400120: ADD r4 0x10, r29
400124: CMP r2 r0; JNE 0x400204
400128: ADD r2 0x2, r0
40012C: MOVB r16, [r29 + 0x11]
400130: NOP
400134: SHL r16 r16 0x8
400138: MOVB r2, [r29 + 0x12]
40013C: NOP
400140: SHL r2 r2 0x10
400144: OR r16 r16 r2
400148: MOVB r2, [r29 + 0x10]
40014C: NOP
400150: OR r16 r16 r2
400154: MOVB r2, [r29 + 0x13]
400158: NOP
40015C: SHL r2 r2 0x18
400160: OR r16 r16 r2 // construct file size a 4-bytes number BE
400164: CMP r16, r0; JE 0x4001b4
400168: ADD r18 0x200, r0
40016C: CMP r0, r0; JE 0x4001a4
400170: LESS r2 r16 0x201

// process the file 0x200 bytes per iteration until the end
// sha256 hash all previously buffered secret + file blocks until you process all the file (one block may remain -- needs padding)
400174: OR r5 r17 r0
400178: MOV r31, 0x400184; JMP 0x400000 // read the buffer itself
40017C: ADD r4 0x114, r29
400180: CMP r2 r0; JNE 0x400228 // make sure it has been read
400184: OR r6 r17 r0
400188: ADD r5 0x114, r29
40018C: MOV r31, 0x400198; JMP 0x400578 // sha256 hash all blocks, and buffer remaining blocks
400190: ADD r4 0x358, r29
400194: SUB r16 r16 r17 // subtract handled size
400198: CMP r16, r0; JE 0x4001b8 // if none remains exit the loop
40019C: ADD r5 0x314, r29
4001A0: LESS r2 r16 0x201
4001A4: CMP r2 r0; JNE 0x400174 // not a full block remains
4001A8: OR r17 r16 r0
4001AC: CMP r0, r0; JE 0x400174 // uncoditional jump back(loop)
4001B0: OR r17 r18 r0 // r17 = 0x200

// pad and hash remaining data
4001B4: ADD r5 0x314, r29
4001B8: MOV r31, 0x4001c4; JMP 0x4006D0 // pad and hash buffered data
4001BC: ADD r4 0x358, r29

// read prx header from 0x10 to 0x30
4001C0: ADD r5 0x20, r0
4001C4: MOV r31, 0x4001d0; JMP 0x400000 // read prx header into r29+0x334
4001C8: ADD r4 0x334, r29
4001CC: CMP r2 r0; JNE 0x400230
// comapres the hash with the header, if they match the validator returns sucess!!
4001D0: ADD r3 0x314, r29 // final hash
4001D4: ADD r5 0x334, r29 // prx header [0x10:0x30]
4001D8: OR r7 r5 r0
4001DC: MOVB r4, [r3 + 0x0]
4001E0: MOVB r6, [r5 + 0x0]
4001E4: NOP
4001E8: XOR r4 r4 r6
4001EC: OR r2 r2 r4
4001F0: AND r2 0xff r2 // r2 |= (r4 ^ r6); r2 &= 0xFF
4001F4: ADD r3 0x1, r3
4001F8: CMP r3 r7; JNE 0x4001e0
4001FC: ADD r5 0x1, r5

// epilogue
400200: LESS r2 r0 r2 // return comparision result
400204: MOVD r31, [r29 + 0x3d4]
400208: MOVD r18, [r29 + 0x3d0]
40020C: MOVD r17, [r29 + 0x3cc]
400210: MOVD r16, [r29 + 0x3c8]
400214: JMP r31
400218: ADD r29 0x3d8, r29
```

the main function looks clean, thank god my diassembly is correct, so it first calls to a function `0x400500`, so I looked at it with the tracer and hence the result
```
// sha256 IV
0x6a09e667, 0xbb67ae85, 0x3c6ef372, 0xa54ff53a
0x510e527f, 0x9b05688c, 0x1f83d9ab, 0x5be0cd19

400500: MOV r2=0x6a090000, 1778974720
400504: OR r2=0x6a09e667, r2=0x6a09e667, 0xe667
400508: MOVD [r4 + 0x0=0x7fffef68], r2=0x6a09e667
40050C: MOV r2=0xbb670000, 3144089600
400510: OR r2=0xbb67ae85, r2=0xbb67ae85, 0xae85
400514: MOVD [r4 + 0x4=0x7fffef6c], r2=0xbb67ae85
400518: MOV r2=0x3c6e0000, 1013841920
40051C: OR r2=0x3c6ef372, r2=0x3c6ef372, 0xf372
400520: MOVD [r4 + 0x8=0x7fffef70], r2=0x3c6ef372
400524: MOV r2=0xa54f0000, 2773417984
400528: OR r2=0xa54ff53a, r2=0xa54ff53a, 0xf53a
40052C: MOVD [r4 + 0xc=0x7fffef74], r2=0xa54ff53a
400530: MOV r2=0x510e0000, 1359872000
400534: ADD r2=0x510e527f, 0x527f, r2=0x510e527f
400538: MOVD [r4 + 0x10=0x7fffef78], r2=0x510e527f
40053C: MOV r2=0x9b050000, 2600796160
400540: ADD r2=0x9b05688c, 0x688c, r2=0x9b05688c
400544: MOVD [r4 + 0x14=0x7fffef7c], r2=0x9b05688c
400548: MOV r2=0x1f830000, 528678912
40054C: OR r2=0x1f83d9ab, r2=0x1f83d9ab, 0xd9ab
400550: MOVD [r4 + 0x18=0x7fffef80], r2=0x1f83d9ab
400554: MOV r2=0x5be00000, 1541406720
400558: OR r2=0x5be0cd19, r2=0x5be0cd19, 0xcd19
40055C: MOVD [r4 + 0x1c=0x7fffef84], r2=0x5be0cd19
400560: OR r2=0x0 r0=0x0 r0=0x0
400564: OR r3=0x0 r0=0x0 r0=0x0
400568: MOVD [r4 + 0x20=0x7fffef88], r2=0x0
40056C: MOVD [r4 + 0x24=0x7fffef8c], r3=0x0
400570: JMP r31=0x400108
400574: MOVD [r4 + 0x68=0x7fffefd0], r0=0x0
```

ooookay, the function looks like it's initializing some sha256 context based on the IV, values so there is atleast some sort of hashing let's suppose for now it's sha256.

so the algorithm is rather simple -- it's doing `sha256(secret+file)`, and because standard sha256 implementations usually process data in blocks of 0x40 bytes with little padding and `len * 8` postfix at the end, the validator's implementation is seperated to when you have a full 0x40 block buffered `0x400578`, and when you don't `0x4006D0`

so I spawned a REPL session testing out our hypothesis
```py
>>> data = open("echo.local.prx", "rb").read()
>>> payload = data[0x30:]
>>> target_hash = data[0x10:0x30]
>>> import hashlib
>>> secret = bytes.fromhex("00112233445566778899aabbccddeeff")
>>> hashlib.sha256(secret + payload).hexdigest()
'06b9b4ff772b501ff2a142d77c581a6fa56ac9a3c856f0eec6bc1093a5ac4973'
>>> target_hash.hex()
'06b9b4ff772b501ff2a142d77c581a6fa56ac9a3c856f0eec6bc1093a5ac4973'
```
ouuu shit that's it??... [clears throat] exuce me.

# The Vulnerability
*by now it may be clear to some of you what the next step should be, I felt lost atp and started brute-forcing the secert :), then I asked the author and because he is a giga-chad he just told me there is a length-extension vulnerability, so I felt stupid for a moment and continued*

so the idea here is pretty simple, because whenever a block is hashed, it's hash becomes the IV of the next block, I can take the very last hash(in the prx header) and make it the IV of our own data, I just need to make sure the base data has what ever padding and data sha256 inserts at the end

but I also need to make sure my data, get's loaded into memory and that I have access to control flow without modifying any file data beyond the offset `0x30`, so the idea was clear there is a dword at file offset `0x8` that initializes the `IP` register I need to point that at the address where my code starts, so that's the control flow problem solved

but for loading the code my first attempt was to increment the file offset `0x5` which is a byte indicating the number of sections, so that automatically loads my own section with my code inside. the only issue is.. look at what the section phdr is:
```
$ xxd -s 80 ./echo.local.prx | head -n 2
80a0 0101 3a30 0000 1080 8100 3b30 0a01  ....:0......;0..
```
the section start would have been at file offset `0x101a080` with a size of `0x818010` which would make the file HUGE, although I [tried](TODO) to do that and did mange to make it work locally, the file was so large that it passed the remote file size limit :(

but them I see something I already forgot about:
![overview](/assets/l3ak_rev_omega/set_until_eof.png)

and the second section falls in that exact case:
```
00000040: 7002 0000 0010 4000 ffff ffff a003 0000  p.....@.........
```

I don't have to introduce a new section, I can just extend the second section(.data) which already is variable length, and point IP in there

# The Light At The End Of The Tunnel

I first generated a payload that works with the local secret:
```py
import struct

MAGIC = b'PRX\x00\x01'
# ip = 0x304e
SECTION_HEADERS = b'\x01\x00\x00\xc4\x13\x40\x00\x00\x00\x00\x00'
HASH = b'\x9e0\xd6\xc4\xda\n\xcfP\x9f\xf5\xa5\x08S+\xc9i\xdc\x90\x86`2\n\xa3\xa0\r\xf8\x8a\x92\x80\xba\xd4\x82'
HEADER = MAGIC+SECTION_HEADERS+HASH
assert len(HEADER) == 0x30

SECTION_START = b'\x40\x00\x00\x00'
PAGE_BASE = b'\xb0\x13\x40\x00'
SECTION_SIZE = b'\x00\x01\x00\x00'
EMPTY_DWORD = b'\x00\x00\x00\x00'
HEADER += SECTION_START + PAGE_BASE + SECTION_SIZE + EMPTY_DWORD

assert len(HEADER) == 0x40

def gen_or_reg(dest, lhs, rhs):
    opcode = 0x6 << 0xb | 0x3b
    opcode |= dest << 0x11
    opcode |= lhs << 0x16
    opcode |= rhs << 0x6
    opcode &= (1 << 32) - 1
    return struct.pack("<I", opcode)

def gen_shl_cte(dest, lhs, cte):
    opcode = 0x3a
    opcode |= (lhs & 0x1f) << 6
    opcode |= 6 << 0xb
    opcode |= (dest & 0x1f) << 0x11
    opcode |= (cte & 0x1f) << 0x1b
    opcode &= (1 << 32) - 1
    return struct.pack("<I", opcode)

def gen_add_cte(dest, lhs, cte):
    opcode = cte & 0x3f
    opcode |= (dest & 0x1f) << 6
    opcode |= 0x35 << 0xb
    opcode |= ((cte >> 0xb) & 0x1f) << 0x11
    opcode |= (lhs & 0x1f) << 0x16
    opcode |= ((cte >> 6) & 0x1f) << 0x1b
    opcode &= (1 << 32) - 1
    return struct.pack("<I", opcode)

# ADD r5, r0, 0x401
# SHL r5, r5, 0xC
# ADD r4, r0, 0x3b0 // for remote 0x3b1
# OR r4, r5, r4
# ADD r2, r0, 0xfa5
# r2 = open("/challenge/flag.txt", 0)

# 0x6->0x3b|0x3e0000 opcode >> 0x11 & 0x1f OR r4, r2, r0
# OR r4, r2, r0
# OR r5, r29, r0
# ADD r6, r0, 0x100
# ADD r2, r0, 0xfa3
# r2 = read(r4, r5, r6)

# ADD r4, r0, 0x1
# OR r5, r29, r0
# ADD r6, r0, 0x100
# ADD r2, r0, 0xfa4
# write(r4, r5, r6)

nop = struct.pack("<I", 0x303a)

# for remote this works
# code = b'/app/flag.txt\x00'
code = b'/challenge/flag.txt\x00'
code = code.ljust(0x14, b'\x00')
assert len(code) == 0x14
code += gen_add_cte(9, 0, 0x401)
code += gen_shl_cte(9, 9, 0xC)
code += gen_add_cte(4, 0, 0x3b0) # for remote 0x3b1
code += gen_or_reg(4, 9, 4)
code += gen_add_cte(2, 0, 0xfa5)
code += b'\x110\x00\x00'

code += gen_or_reg(4, 2, 0)
code += gen_or_reg(5, 29, 0)
code += gen_add_cte(6, 0, 0x1000)
code += b'\xa3\xa8\x03\xf0'
code += b'\x110\x00\x00'

code += struct.pack("<I", 0x1a901)
code += gen_or_reg(5, 29, 0)
code += gen_add_cte(6, 0, 0x1000)
code += b'\xa4\xa8\x03\xf0'
code += b'\x110\x00\x00'

# remove for remote
code += (0x100 - len(code)) // 4 * nop
assert len(code) == 0x100

open("./payload.prx", "wb").write(HEADER + code)
```

when giving it to the executor, it get's verified because the hash is correct, and then it opens /challenge/flag.txt and prints it's content to stdout, that's what it looks like:
```sh
$ ./executor ./payload.prx
[+] Verified
L3AK{flag_for_local_testing}
```

perfect, now I can start the remote solver, here is the script for that:
```py
import struct
import sha256

secret_size = 0xf # secret_size is variable so fuck around until you find a working exploit

remote_data = open("./echo.remote.prx", "rb").read()
payload_data = open("./payload.prx", "rb").read()

remote_header = remote_data[0:0x10]
remote_hash = remote_data[0x10:0x30]
remote_program = remote_data[0x30:]

remain = 0x40 - ((len(remote_program) + secret_size) % 0x40)
final_payload = remote_program + b'\x80' + b'\x00'*(remain - 5) + struct.pack(">I", (len(remote_program) + secret_size) * 8)

assert (len(final_payload) + secret_size) % 0x40 == 0

# prints 0x4015f0 
# print(hex(len(final_payload)))

extend = payload_data[0x40:] # code starts at 0x40
# print(extend.hex())
final_payload += extend

def gen_hash(data, iv):
    mhash = b''
    while len(data) >= 0x40:
        block = data[:0x40]
        mhash = sha256.generate_hash(block, iv)
        iv = [struct.unpack(">I", mhash[i*4:(i+1)*4])[0] for i in range(8)]
        data = data[0x40:]

    assert len(data) > 0
    data += b'\x80'
    data += b'\x00' * (0x3C - len(data))
    data += struct.pack(">I", ((len(final_payload) + secret_size) * 8) & 0xFFFFFFFF)
    assert len(data) == 0x40
    mhash = sha256.generate_hash(data, iv)

    return mhash

iv = [struct.unpack(">I", remote_hash[i*4:(i+1)*4])[0] for i in range(8)]
final_hash = gen_hash(extend, iv)
print(f"FINAL HASH: {final_hash.hex()}")
# ip = 0x4013c4
final_header = b'\x50\x52\x58\x00\x01\x02\x00\x00\xc5\x13\x40\x00\x00\x00\x00\x00'

open("./final.prx", "wb").write(final_header + final_hash + final_payload)
```
the implementation of sha256 I found [here](https://raw.githubusercontent.com/keanemind/python-sha-256/refs/heads/master/sha256.py) I just removed the padding bs cuz I added it manually

there was a one offset difference because the local secret is one byte bigger than the remote, but anyways I run the script, base64 encode the payload, send it to the remote server aaaaaaaaaaaaaaaaaaaaaaaaaanddd........

```sh
$ ncat --ssl omega.instances.ctf.l3ak.team 1337
Ncat: TIMEOUT.
```

the infra is down lol, I took so long solving the challenge that even the time for which the infra stays up has fully passed... skill issue ig lol

I told the author and he was once again a GIGA-CHAD and gave me the files so I can spin up a local instance and after that.....

![overview](/assets/l3ak_rev_omega/FINALLY.png)

TaADAAAAAAA!!! 🎉🎉🎉🎉🎉🎉🎉
