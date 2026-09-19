| | |
|---|---|
| **Author (of CrackMe)** | [Pitou](https://crackmes.one/user/pitou) |
| **Language** | C/C++ |
| **Platform** | Windows (x86-64) |
| **Difficulty** | 3.0 |
| **Quality** | 5.2 |
| **Upload Date** | 2026-05-06 |
| **Labels** | String / data encryption, XOR , Packer, Other named(Morphine/Neolite/PEtite…)|

---

omar far from home, and today we tackle an evasive exe.

for once i felt like my beloved dragon ghidra was completely useless, so time to bring out the bug: x64dbg. i don't think this exe was even made to run properly, it was just made to hide a flag. 

## Initial Analysis

meh, we run it and wipe out all the useless breakpoints. looking clean now. 

let's pull up the memory map to see our boundaries and check if there are actual signs of a packer. in real-life malware analysis, the binary won't be called `packer.exe`, you actually have to look at the layout:

![Memory Map Comparison](images/Screenshot%202026-09-19%20014726.png)

```text
Address          Size             Page Information   Current Protection   Allocation Protection
00007FF719380000 0000000000001000 evasive.exe        -R---                ERWC-
00007FF719381000 0000000000002000 ".text"            ER---                ERWC-
00007FF719383000 0000000000001000 ".rdata"           -R---                ERWC-
00007FF719384000 0000000000001000 ".data"            -RW--                ERWC-
00007FF719385000 0000000000001000 ".pdata"           -R---                ERWC-
00007FF719386000 0000000000009000 ".rsrc"            -R---                ERWC-
00007FF71938F000 0000000000001000 ".reloc"           -R---                ERWC-
```

looking at this, .text is around 0x2000 in size while .rsrc is sitting fat at 0x9000. classic.

## Breaking on VirtualProtect

time for a standard play: slap a breakpoint on VirtualProtect. why? because to unpack whatever payload is compressed inside .rsrc, the binary needs to adjust memory permissions so it can write and execute it without throwing an access violation. skill issue on the os's part, thankfully the cpu handles changing protections so it can write.

we hit our VirtualProtect breakpoint. i hit ctrl + f9 to execute until return because i am NOT digging through 20k lines of assembly inside kernelbase.

![VirtualProtect Entry Breakpoint](images/Screenshot%202026-09-19%20023404.png)

now we are here:

```x86 Assembly
00007FFCFECCE49A | C3 | ret
```

nobody cares about this, step over.

![Stepping into VirtualProtect](images/Screenshot%202026-09-19%20023754.png)

now we're back in the pe code looking for anything interesting like a call rax or jmp rax. nothing.

magic trick again: ctrl + f9 and step over. back in kernelbase bullshit. no call rax.

magic trick again, step over, another ret. still no call rax.

repeat. repeat. this is getting boring, just more mov rax and crap... and then i realize i've hit this loop twenty damn times because i forgot to remove the breakpoint on VirtualProtect. take notes kids. i certainly did this on purpose, i just wanted to see the function twenty times to memorize it for sure.

![Unpacker Protection Loop](images/Screenshot%202026-09-19%20023714.png)

![Flag Reference in Registers](images/Screenshot%202026-09-19%20023907.png)

i disable the breakpoint, run the magic trick again, and clear out of VirtualProtect once and for all.

![Clearing the VirtualProtect Breakpoint](images/Screenshot%202026-09-19%20023931.png)

## Payload Decompression

the loader finishes decompressing sections and sets up the buffer, showing valid PE header signatures in rax:

![PE Header Reconstitution](images/Screenshot%202026-09-19%20024225.png)

the unpack helper finishes up its cleanup:

![Unpacker Cleanup](images/Screenshot%202026-09-19%20024244.png)

and wait... DO I SEE A CALL RAX?

yes son, you are right:

```x86 Assembly
00007FF719382D37 | FFD0 | call rax
```

more precious than ever. time to step inside.

## Entry Point Execution

we land right at the unpacked oep:

![OEP Transfer Logic](images/Screenshot%202026-09-19%20024331.png)

```x86 Assembly
0000000140001440 | 48:83EC 28        | sub rsp,280000000140001444 | 48:8B05 D52F0000  | mov rax,qword ptr ds:[140004420]
000000014000144B | C700 00000000     | mov dword ptr ds:[rax],00000000140001451 | E8 CAFBFFFF       | call 140001020
```

let's step into that call at 0x140001020.

## Finding Main

now before you start crying looking at all the crt startup garbage, we just need to find the actual main. how do we find main? it usually takes 3 arguments (argc, argv, envp) loaded into registers right before the call:

![CRT Initialization and Main Call](images/Screenshot%202026-09-19%20024729.png)

```x86 Assembly
00000001400010BD | 4C:8B05 4C5F0000  | mov r8,qword ptr ds:[140007010]
00000001400010C4 | 8B0D 565F0000     | mov ecx,dword ptr ds:[140007020]
00000001400010CA | 4C:8900           | mov qword ptr ds:[rax],r8
00000001400010CD | 48:8B15 445F0000  | mov rdx,qword ptr ds:[140007018]
00000001400010D4 | E8 B7030000       | call 140001490
```

three arguments loaded right after __getmainargs setup, followed by a call. that's our target. set a breakpoint on 0x140001490 and step in.

## The Flag Decryption

inside main, it starts dumping immediate values into a stack array:

![Main Function Initialization](images/Screenshot%202026-09-19%20024743.png)

```x86 Assembly
00000001400014A5 | C745 E0 29000000  | mov dword ptr ss:[rbp-20],29
00000001400014AC | C745 E4 25000000  | mov dword ptr ss:[rbp-1C],25
00000001400014B3 | C745 E8 25000000  | mov dword ptr ss:[rbp-18],25
00000001400014BA | C745 EC 25000000  | mov dword ptr ss:[rbp-14],25
00000001400014C1 | C745 F0 1B000000  | mov dword ptr ss:[rbp-10],1B
```

first index is at [rbp-20]. it keeps pushing values all the way down until it hits the loop setup:

```x86 Assembly
00000001400015AF | C745 78 60000000  | mov dword ptr ss:[rbp+78],60
00000001400015B6 | C745 7C 00000000  | mov dword ptr ss:[rbp+7C],0
00000001400015BD | EB 1B              | jmp 1400015DA
00000001400015BF | 8B45 7C           | mov eax,dword ptr ss:[rbp+7C]
00000001400015C2 | 48:98              | cdqe
00000001400015C4 | 8B4485 E0         | mov eax,dword ptr ss:[rbp+rax*4-20]
00000001400015C8 | 3345 78           | xor eax,dword ptr ss:[rbp+78]
00000001400015CB | 89C2              | mov edx,eax
00000001400015CD | 8B45 7C           | mov eax,dword ptr ss:[rbp+7C]
00000001400015D0 | 48:98              | cdqe
00000001400015D2 | 895485 E0         | mov dword ptr ss:[rbp+rax*4-20],edx
00000001400015D6 | 8345 7C 01        | add dword ptr ss:[rbp+7C],1
00000001400015DA | 837D 7C 25        | cmp dword ptr ss:[rbp+7C],25
00000001400015DE | 7E DF              | jle 1400015BF
```

here's the routine:

- loop index [rbp+7c] starts at 0.
- loop condition checks cmp [rbp+7C], 0x25 with a jle (so 0 to 37 inclusive = 38 elements).
- each round, it grabs dword ptr [rbp + rax*4 - 0x20], xors it with [rbp+78] (which is 0x60), and writes it right back.

is it time to turn my brain into an i9 processor and calculate 38 xors by hand? nah, my actual cpu is right there, thank you brain you can rest for now.

i set a breakpoint right after the loop exits at 0x1400015E0. let the hardware do the math. hit run, breakpoint hits, and everything is decrypted cleanly in memory.

![Decryption Loop with Breakpoint](images/Screenshot%202026-09-19%20025232.png)

we follow the array starting at [rbp-20] into the hex dump:

![Decrypted Flag in Memory](images/Screenshot%202026-09-19%20025327.png)

```plaintext
000000710CF9F680  49 00 00 00 45 00 00 00 45 00 00 00 45 00 00 00  I...E...E...E...  
000000710CF9F690  7B 00 00 00 53 00 00 00 30 00 00 00 33 00 00 00  {...S...0...3...  
000000710CF9F6A0  31 00 00 00 6D 00 00 00 65 00 00 00 5F 00 00 00  1...m...e..._...  
000000710CF9F6B0  54 00 00 00 31 00 00 00 6D 00 00 00 65 00 00 00  T...1...m...e...  
000000710CF9F6C0  73 00 00 00 5F 00 00 00 57 00 00 00 65 00 00 00  s..._...W...e...  
000000710CF9F6D0  5F 00 00 00 68 00 00 00 31 00 00 00 73 00 00 00  _...h...1...s...  
000000710CF9F6E0  5F 00 00 00 74 00 00 00 30 00 00 00 5F 00 00 00  _...t...0..._...  
000000710CF9F6F0  53 00 00 00 75 00 00 00 31 00 00 00 31 00 00 00  S...u...1...1...  
000000710CF9F700  31 00 00 00 66 00 00 00 66 00 00 00 65 00 00 00  1...f...f...e...  
000000710CF9F710  72 00 00 00 7D 00 00 00 60 00 00 00 26 00 00 00  r...}...`...&...  
```

pay attention to the actual hex values so you don't mess up the casing and zeros like an idiot:

## Flag

```
IEEE{S031me_T1mes_We_h1s_t0_Su111ffer}
```

and tada. bypassed the loader, extracted the payload logic, and grabbed the flag without caring what compression was used under the hood.
