# Chapter 1 x86 and x64

| Exercise | Status | 
| --- | --- |
| [Page 11](#exercise-page-11) | :heavy_check_mark: |
| [Page 17](#exercise-page-17) | :heavy_check_mark: |
| [Page 35 - 36](#exercise-page-35---36) | :heavy_check_mark: |
 
## Exercise page 11

`[ebp + 8]` appeared to be a byte sequence of size unknown (but not greater than 0FFFFFFFFh). Speaking C way, this is a null-terminated string.  
`[ebp + 0Ch]` appeared to be a byte. In C, it is a character.

This snippet first loop through the sequence until it meets a NULL. Then it replaces all bytes of the sequence with the value of `[ebp + 0Ch]`.

I took some extra steps to get a compilable x86 MASM [file](p11.asm). I have left comment after each line of code.

## Exercise page 17  
1. Since `eip` is not a general purpose register, it doesn't support `mov eax, eip`.
We know `call` put the offset about to be executed after the call (value of EIP after the call) on top of the stack, and we can access value on the stack, maybe this [code](/p171.asm) would do the trick?  
Setting 2 breakpoints at line 11, 12 and debug, I realize that right before execute line 12, we got the right value of `eip` in our register `eax`! We can see the value of `eip` after the call is 5 bytes greater than the value before the call, since the `call` instruction size is 5 bytes (could be seen by looking at Memory view when  debug).  
  
2.  We can't just simply do `mov eip, 0xAABBCCDD` or similar since `eip` is not a GPR.  
- Knowing `ret` just pops the value on top of the stack to `eip` to execute, I came up with [this](p1721.asm) idea...  
You might face an *Frame not in module* error while running but don't worry. Now `eip` has pointed to our offset!
- Another trick is using [`call`](p1723.asm) or [`jmp`](p1722.asm) family. Try calling it with our offset, but in masm we can't do it directly so I put our value in `eax`.
`call` is just a `jmp` and a `push` onto the stack.  

3. In general, if we don't re-align the stack before calling `ret`, `eip` might receive the wrong value to execute and therefore cause much trouble.  
But in this example, since the function `addme` didn't make any `push` or `pop` instruction, the `esp` was not modified at all, so, even if we did not restore it, nothing would go wrong.    

4. Write [a simple C program](p174.c) to experiment. After compiling with MS C++ Compiler, we have an [listing file](p174.cod) here.  
Look at line 77:
```assembly
; 19   :     func(dat);

  0002c	83 ec 0c	 sub	 esp, 12			; 0000000cH
  0002f	8b c4		 mov	 eax, esp
  00031	8b 4d f0	 mov	 ecx, DWORD PTR _dat$[ebp]
  00034	89 08		 mov	 DWORD PTR [eax], ecx
  00036	8b 55 f4	 mov	 edx, DWORD PTR _dat$[ebp+4]
  00039	89 50 04	 mov	 DWORD PTR [eax+4], edx
  0003c	8b 4d f8	 mov	 ecx, DWORD PTR _dat$[ebp+8]
  0003f	89 48 08	 mov	 DWORD PTR [eax+8], ecx
  00042	8d 95 1c ff ff
	ff		 lea	 edx, DWORD PTR $T1[ebp]
  00048	52		 push	 edx
  00049	e8 00 00 00 00	 call	 _func
  0004e	83 c4 10	 add	 esp, 16			; 00000010H
```
I created a struct of 3 ints (3 double-words). The compiler first copies the value to a new offset, then pushes the base address onto the stack to pass to my function `func`. On the function, the `return` part which started from line 167 is:
```assembly
; 13   :     return obj;

  00031	8b 45 08	 mov	 eax, DWORD PTR $T1[ebp]
  00034	8b 4d 0c	 mov	 ecx, DWORD PTR _obj$[ebp]
  00037	89 08		 mov	 DWORD PTR [eax], ecx
  00039	8b 55 10	 mov	 edx, DWORD PTR _obj$[ebp+4]
  0003c	89 50 04	 mov	 DWORD PTR [eax+4], edx
  0003f	8b 4d 14	 mov	 ecx, DWORD PTR _obj$[ebp+8]
  00042	89 48 08	 mov	 DWORD PTR [eax+8], ecx
  00045	8b 45 08	 mov	 eax, DWORD PTR $T1[ebp]
```
In short, it stores the base address in `eax` to return. In case my C program has only 2 ints in the struct, it would return in `eax` and `edx`.
I tried compiling it with GCC to get [this file](p174.s):
```
$ gcc -fverbose-asm -S p174.c -o p174.s -O0 -masm=intel -m32
```
Take a look at `main` function and we can see the arguments is passed to function by stack at line 97:
```assembly
# p174.c:18:     dat.x = 2;
	mov	DWORD PTR -20[ebp], 2	# dat.x,
# p174.c:19:     func(dat);
	lea	eax, -40[ebp]	# tmp85,
	push	DWORD PTR -12[ebp]	# dat
	push	DWORD PTR -16[ebp]	# dat
	push	DWORD PTR -20[ebp]	# dat
	push	eax	# tmp85
	call	func	#
	add	esp, 12	#,
```
One more `cdecl` call and it pushes all 3 values onto the stack. In the `func` function, our `return` is implemented at line 63:
```assembly
# p174.c:13:     return obj;
	mov	eax, DWORD PTR 8[ebp]	# tmp85, .result_ptr
	mov	edx, DWORD PTR 12[ebp]	# tmp86, obj
	mov	DWORD PTR [eax], edx	# <retval>, tmp86
	mov	edx, DWORD PTR 16[ebp]	# tmp87, obj
	mov	DWORD PTR 4[eax], edx	# <retval>, tmp87
	mov	edx, DWORD PTR 20[ebp]	# tmp88, obj
	mov	DWORD PTR 8[eax], edx	# <retval>, tmp88
```
To implement `return`, the compiler stores the base address of the struct onto the `eax` register to be the return value.

So, by some experiment on the MS C++ Compiler and GCC compiler, we can say the mechanism doesn't vary between compilers.

## Exercise page 35 - 36

1. The stack layout is described in [this gg sheet](https://docs.google.com/spreadsheets/d/1AQREVVd0bjASfqp_hRH5qtiQxko3ucsZ3aO7q1_bCl4/edit?usp=sharing). If there is any mistake, hope someone would point it out!  

2. Here is my [re-decompile work](p352.c).   

3. A `_` prefix and a `@` postfix followed by a number is used with functions using `stdcall` calling convention and the number after `@` indicates how many bytes are used for function paramaters. Windows' dlls use this by default.  

4. My implement for these functions are listed below:  

| function | 
| --- | 
| [`strlen`](#strlen) |  
| [`strchr`](#strchr) |  
| [`memset`](#memset) |  
| [`strcmp`](#strcmp) |  
| [`strset`](#strset) |  

### `strlen`
Loop through string and compare until meet a NULL byte.
```assembly
strlen proc
    push    ebp
    mov     ebp, esp
    mov     edi, [ebp + 8]
    mov     ecx, 0ffffffffh
    xor     al, al
    repne   scasb 
    sub     edi, [ebp + 8]
    dec     edi
    mov     eax, edi
    mov     esp, ebp
    pop     ebp
    ret     4
strlen endp
```
### `strchr`  
Loop through the string. If meet the character, return the position. If meet a NULL, it means the string doesn't contain the character, return the `ptr`.
```assembly
strchr proc
    push    ebp
    mov     ebp, esp
    mov     edi, [ebp + 8]
    mov     al, byte ptr [ebp + 0ch]
    mov     ah, 0
    compare:
    cmp     byte ptr [edi], ah
    jz      fail
    scasb
    jne     compare  
    sub     edi, [ebp + 8]
    mov     eax, edi
    mov     esp, ebp
    pop     ebp
    ret     8
    fail:
    mov     eax, 0
    mov     esp, ebp
    pop     ebp
    ret     8
strchr endp
```
### `memset`
Loop through the string byte-to-byte for `size` time and set each to the value. Return the `ptr`.
```assembly
memset proc
    push    ebp
    mov     ebp, esp
    mov     edi, [ebp + 8]
    mov     al, byte ptr [ebp + 0ch]
    mov     ecx, [ebp + 10h]
    rep     stosb
    mov     eax, [ebp + 8]
    mov     esp, ebp
    pop     ebp
    ret     0ch
memset endp
```
### `strcmp` 
First find one string's length so that we know how maximum times we need to loop through the 2 string and compare. If meets any different before reach the limit, stop execution and return the result based on the flags affected by the `cmpsb` instruction. If we continously meet equal characters for `len` times, it is probably equal (if the other string reachs its end too) or the string we calculated length is smaller in size. 
```assembly
strcmp proc
    push    ebp
    mov     ebp, esp
    mov     ecx, 0ffffffffh
    mov     esi, [ebp + 8]
    ; calc strlen to count loops
    mov     edi, [ebp + 0ch]
    mov     ecx, 0ffffffffh
    xor     al, al
    repne   scasb
    sub     edi, [ebp + 0ch]
    dec     edi
    mov     ecx, edi

    mov     edi, [ebp + 0ch]
    compare:
    repe    cmpsb
    jg      greater
    jl      less
    test    ecx, ecx
    jnz     greater
    cmp     byte ptr [esi], 0
    jnz     greater
    mov     eax, 0
    mov     esp, ebp
    pop     ebp
    ret     8    

    greater:
    mov     eax, 1
    mov     esp, ebp
    pop     ebp
    ret     8

    less:
    mov     eax, 0ffffffffh
    mov     esp, ebp
    pop     ebp
    ret     8
strcmp endp
```

### `strset`
Set all characters in the string to `char` except the NULL character.
```assembly
strset proc
    push    ebp
    mov     ebp, esp

    ; calc strlen
    mov     edi, [ebp + 8]
    mov     al, 0
    mov     ecx, 0ffffffffh
    repne   scasb
    sub     edi, [ebp +8]
    dec     edi
    mov     ecx, edi        ; set timer
    mov     edi, [ebp + 8]
    mov     al, [ebp + 0ch]
    rep     stosb
    mov     eax, [ebp + 8]
    mov     esp, ebp
    pop     ebp
    ret     8
strset endp
```
I put all these functions inside my [masm program]([p354.asm]) here. Debug it if you need to!  

5. Decompile some kernel routines in Windows:  

First you need some preparation to be able to debug kernel (so that we can decompile it). I wrote [a short tutorial](https://github.com/pearldarkk/Windows-Kernel-Debugging) about settup between 2 Windows VM so that no amateur (like me) would take few days just to do it...  
  
At the tutorial I used a x64 target Windows but since we're learning x86 so please use a x86 Windows VM instead (I use Win10 32bit version).  Remember to break into the process (`Debug` -> `Break`).

Some resources I found useful (if you are a beginner like me, i think you'll need):  
- Common command references from `windbg.info`: http://windbg.info/doc/1-common-cmds.html  
- A video about how to debug kernel in a vm from physical computer (he also show how to decompile an example kernel too): https://www.youtube.com/watch?v=ch8AuPsZ3aM&t=156s
- Focus on how he take a shortcut to load symbols: https://voidsec.com/windows-kernel-debugging-exploitation/#More_Windows_Debuggee_Flavours
- If something goes wrong with your symbols: https://stackoverflow.com/questions/30019889/how-to-set-up-symbols-in-windbg  

Now let's get started!!! 
| function | 
| --- | 
| [`KeInitializeDpc`](#KeInitializeDpc) |  
| [`KeInitializeApc`](#KeInitializeApc) |  
| [`ObFastDereferenceObject`](#ObFastDereferenceObject) |  
| [`KeInitializeQueue`](#KeInitializeQueue) |  
| [`KxWaitForLockChainValid`](#KxWaitForLockChainValid) |  
| [`KeReadyThread`](#KeReadyThread) |  
| [`KiInitializeTSS`](#KiInitializeTSS) |  
| [`RtValidateUnicodeString`](#RtValidateUnicodeString) |  

### `KeInitializeDpc` 
  
Disassemble the function:  
```
uf keinitializedpc
```
If you're debugging a x86 Win 10 machine like me, you should receive an ouput similar to this:
```assembly
kd> uf keinitializedpc
nt!KeInitializeDpc:
8237cc3a 8bff            mov     edi,edi
8237cc3c 55              push    ebp
8237cc3d 8bec            mov     ebp,esp
8237cc3f 8b4d08          mov     ecx,dword ptr [ebp+8]
8237cc42 8b450c          mov     eax,dword ptr [ebp+0Ch]
8237cc45 83611c00        and     dword ptr [ecx+1Ch],0
8237cc49 83610800        and     dword ptr [ecx+8],0
8237cc4d 89410c          mov     dword ptr [ecx+0Ch],eax
8237cc50 8b4510          mov     eax,dword ptr [ebp+10h]
8237cc53 c70113010000    mov     dword ptr [ecx],113h
8237cc59 894110          mov     dword ptr [ecx+10h],eax
8237cc5c 5d              pop     ebp
8237cc5d c20c00          ret     0Ch
```
A `stdcall` function (`ret 0Ch` part, knowing it take 3 arguments from looking at [`msdn`](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-keinitializedpc)). 
At `3f`, `ecx` holds value of the first argument `Dpc` (_"Pointer to a KDPC structure that represents the DPC object to initialize. The caller must allocate storage for the structure from resident memory.", from msdn_). 
To explore about the structure, type in `dt _kdpc`. Output:
```assembly
kd> dt _kdpc
nt!_KDPC
   +0x000 TargetInfoAsUlong : Uint4B
   +0x000 Type             : UChar
   +0x001 Importance       : UChar
   +0x002 Number           : Uint2B
   +0x004 DpcListEntry     : _SINGLE_LIST_ENTRY
   +0x008 ProcessorHistory : Uint4B
   +0x00c DeferredRoutine  : Ptr32     void 
   +0x010 DeferredContext  : Ptr32 Void
   +0x014 SystemArgument1  : Ptr32 Void
   +0x018 SystemArgument2  : Ptr32 Void
   +0x01c DpcData          : Ptr32 Void
```

Location `42`, `eax` holds value of second argument, `(PKDEFERRED_ROUTINE) DeferredRoutine`. 
From `KDPC` structure details:  
- `[ecx+1ch]` presents `Dpc.DpcData`
- `[ecx+8]` presents `Dpc.ProcessHistory`
- `[ecx+0ch]` presents `Dpc.DeferredRoutine`
- `[ecx+10h]` presents `Dpc.DeferredContext`
- `[ecx]` presents `Dpc.TargetInfoAsUlong`  

Location `45`, `49` zeroes out `Dpc.DpcData` and `Dpc.ProcessHistory`. Location `4d` set `Dpc.DeferredRoutine` to value of `DeferredRoutine`.
Location `50`, eax holds the third argument `(PVOID) DeferredContext`.
Location `53`, `Dpc.TargetInfoAsULong` is set to 113h, which means `Dpc.Type = 13h`, `Dpc.Importance = 1`, and `Dpc.Number = 0` _(Windows are Littie Endian system)_. At last, save `eax` back to `Dpc.DeferredContext` and return.
In C, it will look similar to:
```c
typedef struct _KDPC {
  UCHAR Type;
  UCHAR Importance;
  WORD  Number;
  SINGLE_LIST_ENTRY DpcListEntry;
  DOUBLEWORD ProcessorHistory;
  PVOID DeferredRoutine;
  PVOID DeferredContext;
  PVOID SystemArgument1;
  PVOID SystemArgument2;
  PVODI DpcData;
} KDPC, *PRKDPC;

void KeInitializeDpc(PRKDPC Dpc, PKDEFERRED_ROUTINE   DeferredRoutine, PVOID  DeferredContext) {
  Dpc->DpcData = 0;
  Dpc->ProcessorHistory = 0;
  Dpc->DeferredRoutine = DeferredRoutine;
  Dpc->DeferredContext = DeferredContext;
  Dpc->Type = 13h;
  Dpc->Importance = 1;
  Dpc->Number = 0;
}
```
### `KeInitializeApc`
```assembly
kd> uf keinitializeapc
nt!KeInitializeApc:
81d69ad0 8bff            mov     edi,edi
81d69ad2 55              push    ebp
81d69ad3 8bec            mov     ebp,esp
81d69ad5 8b5508          mov     edx,dword ptr [ebp+8]
81d69ad8 8b4510          mov     eax,dword ptr [ebp+10h]
81d69adb 8b4d0c          mov     ecx,dword ptr [ebp+0Ch]
81d69ade c60212          mov     byte ptr [edx],12h
81d69ae1 c6420230        mov     byte ptr [edx+2],30h
81d69ae5 83f802          cmp     eax,2
81d69ae8 7439            je      nt!KeInitializeApc+0x53 (81d69b23)  Branch

nt!KeInitializeApc+0x1a:
81d69aea 88422c          mov     byte ptr [edx+2Ch],al
81d69aed 8b4514          mov     eax,dword ptr [ebp+14h]
81d69af0 894214          mov     dword ptr [edx+14h],eax
81d69af3 8b4518          mov     eax,dword ptr [ebp+18h]
81d69af6 894218          mov     dword ptr [edx+18h],eax
81d69af9 8b451c          mov     eax,dword ptr [ebp+1Ch]
81d69afc 894a08          mov     dword ptr [edx+8],ecx
81d69aff 8bc8            mov     ecx,eax
81d69b01 f7d9            neg     ecx
81d69b03 89421c          mov     dword ptr [edx+1Ch],eax
81d69b06 1bc9            sbb     ecx,ecx
81d69b08 234d24          and     ecx,dword ptr [ebp+24h]
81d69b0b 85c0            test    eax,eax
81d69b0d 0f94c0          sete    al
81d69b10 fec8            dec     al
81d69b12 224520          and     al,byte ptr [ebp+20h]
81d69b15 88422d          mov     byte ptr [edx+2Dh],al
81d69b18 894a20          mov     dword ptr [edx+20h],ecx
81d69b1b c6422e00        mov     byte ptr [edx+2Eh],0
81d69b1f 5d              pop     ebp
81d69b20 c22000          ret     20h

nt!KeInitializeApc+0x53:
81d69b23 8a816a010000    mov     al,byte ptr [ecx+16Ah]
81d69b29 ebbf            jmp     nt!KeInitializeApc+0x1a (81d69aea)  Branch
```  
As usual, a `stdcall` function.  
According to [codewarrior.vn](http://www.codewarrior.cn/ntdoc/winnt/ke/KeInitializeApc.htm):
- `[ebp + 8]`: `Apc`, a pointer to a control object of type `APC`
- `[ebp + 0ch]`: `Thread`
- `[ebp + 10h]`: `Environment` 
- ...

After loc `db`, `edx` holds pointer `Apc`, `eax` holds `Environment`, `ecx` hold `Thread`.
Search for struct `Apc` by typing in `dt _kapc`:
```assembly
nt!_KAPC
   +0x000 Type             : UChar
   +0x001 SpareByte0       : UChar
   +0x002 Size             : UChar
   +0x003 SpareByte1       : UChar
   +0x004 SpareLong0       : Uint4B
   +0x008 Thread           : Ptr32 _KTHREAD
   +0x00c ApcListEntry     : _LIST_ENTRY
   +0x014 KernelRoutine    : Ptr32     void 
   +0x018 RundownRoutine   : Ptr32     void 
   +0x01c NormalRoutine    : Ptr32     void 
   +0x014 Reserved         : [3] Ptr32 Void
   +0x020 NormalContext    : Ptr32 Void
   +0x024 SystemArgument1  : Ptr32 Void
   +0x028 SystemArgument2  : Ptr32 Void
   +0x02c ApcStateIndex    : Char
   +0x02d ApcMode          : Char
   +0x02e Inserted         : UChar
```
Loc `de` sets `Apc.Type` to `12h`. Loc `e1` sets `Apc.Size` to `30h`. 
Loc `e5` compares wether `Environment` equals `2`. 
If equal, jump to `nt!KeInitializeApc+0x53` to set `al` to value at `[ecx+16ah]`, 
which means `Thread.ApcStateIndex` _(type in `dt _kthread` to see)_ and 
jump to `nt!KeInitializeApc+0x1a`. If not equal, continue at `nt!KeInitializeApc+0x1a`, too.  
Loc `ea` set `Apc.ApcStateIndex` to `al`. Loc `ed` and `f0` set `Apc.KernelRoutine` to function 4th argument `KernelRoutine`.
Loc `f3`, `f6` set `Apc.RundownRoutine` to function 5th argument `RundownRoutine`.
Loc `fc` set `Apc.Thread` to `Thread`, then `ecx` is set to `Apc.NormalRoutine`.  
`neg ecx` (loc `01`) clears the carry flag `cf` (set to 0) if `ecx == 0`, else sets `cf`, and 
`sbb ecx, ecx` (loc `06`) is a common idiom _(in compiler-generated)_ to isolate `-cf`,
`and` it with function argument `NormalContext` (loc `08`) and store result to `Apc.NormalContext` (loc `18`).  
Loc `f9`, `03` set `Apc.NormalRoutine` to function argument `NormalRoutine` and test if it equals 0 (loc `0b`).
If `zf` is set (`eax` = 0), `al` is set 1, else `al` is set to 0 (loc `0d`). Decrease `al`, `and` with function argument `ApcMode` and set 
`Apc.Mode` to the result (from loc `0b` to `2d`).  
Lastly, set `Apc.Inserted` to `0` and return the function.  
In C, this function will look something similar to this:
```c
typedef struct KAPC {
  UCHAR Type;
  UCHAR SpareByte0;
  UCHAR Size;
  UCHAR SpareByte1;
  DOUBLEWORD SpareLong0;
  KTHREAD Thread
  LIST_ENTRY ApcListEntry;
  PVOID KernelRoutine;
  PVOID RundownRoutine;
  PVOID NormalRoutine;
  PVOID[3] Reserved;
  PVOID NormalContext;
  PVOID SystemArgument1;
  PVOID SystemArgument2;
  CHAR ApcStateIndex;
  CHAR ApcMode;
  UCHAR Inserted;
} KAPC, *PRKAPC;

void KeInitializeApc(PRKAPC Apc, PRKTHREAD Thread, KAPC_ENVIRONMENT Environment, PKKERNEL_ROUTINE KernelRoutine, PKRUNDOWN_ROUTINE RundownRoutine OPTIONAL, PKNORMAL_ROUTINE NormalRoutine OPTIONAL, KPROCESSOR_MODE ApcMode OPTIONAL, PVOID NormalContext OPTIONAL) {
    Apc.Type = 12h;
    Apc.Size = 30h;
    if (Environment == 2) {
      Apc.StateIndex = Thread.ApcStateIndex;
    }
    else Apc.StateIndex = Environment;
    Apc.KernelRoutine = KernelRoutine;
    Apc.RundownRoutine = RundownRoutine;
    Apc.NormalRoutine = NormalRoutine;
    Apc.Thread = Thread;
    if (NormalRoutine == 0) {
      Apc.NormalContext = 0;
      Apc.Mode = 0;
    } else {
      Apc.NormalContext = NormalContext;
      Apc.ApcMode = ApcMode;
    }
    Apc.Inserted = 0;
    return;
}
```

### `ObFastDereferenceObject`
```assembly
kd> uf obfastdereferenceobject
nt!ObfDereferenceObject:
81cf6870 8bff            mov     edi,edi
81cf6872 55              push    ebp
81cf6873 8bec            mov     ebp,esp
81cf6875 83e4f8          and     esp,0FFFFFFF8h
81cf6878 51              push    ecx
81cf6879 833d0022f88100  cmp     dword ptr [nt!ObpTraceFlags (81f82200)],0
81cf6880 53              push    ebx
81cf6881 8bd9            mov     ebx,ecx
81cf6883 56              push    esi
81cf6884 57              push    edi
81cf6885 8d7be8          lea     edi,[ebx-18h]
81cf6888 0f859ac71200    jne     nt!ObfDereferenceObject+0x12c7b8 (81e23028)  Branch

nt!ObfDereferenceObject+0x1e:
81cf688e 83ceff          or      esi,0FFFFFFFFh
81cf6891 f00fc137        lock xadd dword ptr [edi],esi
81cf6895 4e              dec     esi
81cf6896 85f6            test    esi,esi
81cf6898 7e09            jle     nt!ObfDereferenceObject+0x33 (81cf68a3)  Branch

nt!ObfDereferenceObject+0x2a:
81cf689a 8bc6            mov     eax,esi
81cf689c 5f              pop     edi
81cf689d 5e              pop     esi
81cf689e 5b              pop     ebx
81cf689f 8be5            mov     esp,ebp
81cf68a1 5d              pop     ebp
81cf68a2 c3              ret

nt!ObfDereferenceObject+0x33:
81cf68a3 8b4704          mov     eax,dword ptr [edi+4]
81cf68a6 85c0            test    eax,eax
81cf68a8 0f858fc71200    jne     nt!ObfDereferenceObject+0x12c7cd (81e2303d)  Branch

nt!ObfDereferenceObject+0x3e:
81cf68ae 85f6            test    esi,esi
81cf68b0 0f88b1c71200    js      nt!ObfDereferenceObject+0x12c7f7 (81e23067)  Branch

nt!ObfDereferenceObject+0x46:
81cf68b6 64a124010000    mov     eax,dword ptr fs:[00000124h]
81cf68bc 6683b83e01000000 cmp     word ptr [eax+13Eh],0
81cf68c4 7540            jne     nt!ObfDereferenceObject+0x96 (81cf6906)  Branch

nt!ObfDereferenceObject+0x56:
81cf68c6 e835801000      call    nt!KeAreInterruptsEnabled (81dfe900)
81cf68cb 84c0            test    al,al
81cf68cd 7437            je      nt!ObfDereferenceObject+0x96 (81cf6906)  Branch

nt!ObfDereferenceObject+0x5f:
81cf68cf ff1594f0f781    call    dword ptr [nt!_imp__KeGetCurrentIrql (81f7f094)]
81cf68d5 3c01            cmp     al,1
81cf68d7 732d            jae     nt!ObfDereferenceObject+0x96 (81cf6906)  Branch

nt!ObfDereferenceObject+0x69:
81cf68d9 8bcf            mov     ecx,edi
81cf68db e85084ffff      call    nt!OBJECT_HEADER_TO_HANDLE_REVOCATION_INFO (81ceed30)
81cf68e0 85c0            test    eax,eax
81cf68e2 7407            je      nt!ObfDereferenceObject+0x7b (81cf68eb)  Branch

nt!ObfDereferenceObject+0x74:
81cf68e4 8bc8            mov     ecx,eax
81cf68e6 e845823900      call    nt!ObpHandleRevocationBlockRemoveObject (8208eb30)

nt!ObfDereferenceObject+0x7b:
81cf68eb 833d0022f88100  cmp     dword ptr [nt!ObpTraceFlags (81f82200)],0
81cf68f2 751b            jne     nt!ObfDereferenceObject+0x9f (81cf690f)  Branch

nt!ObfDereferenceObject+0x84:
81cf68f4 32d2            xor     dl,dl
81cf68f6 8bcf            mov     ecx,edi
81cf68f8 e8b3f83000      call    nt!ObpRemoveObjectRoutine (820061b0)
81cf68fd 5f              pop     edi
81cf68fe 8bc6            mov     eax,esi
81cf6900 5e              pop     esi
81cf6901 5b              pop     ebx
81cf6902 8be5            mov     esp,ebp
81cf6904 5d              pop     ebp
81cf6905 c3              ret

nt!ObfDereferenceObject+0x96:
81cf6906 8bcf            mov     ecx,edi
81cf6908 e8f3b10b00      call    nt!ObpDeferObjectDeletion (81db1b00)
81cf690d eb8b            jmp     nt!ObfDereferenceObject+0x2a (81cf689a)  Branch

nt!ObfDereferenceObject+0x9f:
81cf690f 8bcf            mov     ecx,edi
81cf6911 e834cd5200      call    nt!ObpDeregisterObject (8222364a)
81cf6916 ebdc            jmp     nt!ObfDereferenceObject+0x84 (81cf68f4)  Branch

nt!ObfDereferenceObject+0x12c7b8:
81e23028 6844666c74      push    746C6644h
81e2302d 6a01            push    1
81e2302f 32d2            xor     dl,dl
81e23031 8bcf            mov     ecx,edi
81e23033 e8bf2c0a00      call    nt!ObpPushStackInfo (81ec5cf7)
81e23038 e95138edff      jmp     nt!ObfDereferenceObject+0x1e (81cf688e)  Branch

nt!ObfDereferenceObject+0x12c7cd:
81e2303d 50              push    eax
81e2303e 8bc7            mov     eax,edi
81e23040 c1e808          shr     eax,8
81e23043 0fb6c8          movzx   ecx,al
81e23046 0fb6470c        movzx   eax,byte ptr [edi+0Ch]
81e2304a 33c8            xor     ecx,eax
81e2304c 0fb605fcb7f881  movzx   eax,byte ptr [nt!ObHeaderCookie (81f8b7fc)]
81e23053 6a01            push    1
81e23055 33c8            xor     ecx,eax
81e23057 53              push    ebx
81e23058 8b048d00b8f881  mov     eax,dword ptr nt!ObTypeIndexTable (81f8b800)[ecx*4]
81e2305f 50              push    eax
81e23060 6a18            push    18h
81e23062 e8edb5fdff      call    nt!KeBugCheckEx (81dfe654)

nt!ObfDereferenceObject+0x12c7f7:
81e23067 56              push    esi
81e23068 6a02            push    2
81e2306a 53              push    ebx
81e2306b 6a00            push    0
81e2306d 6a18            push    18h
81e2306f e8e0b5fdff      call    nt!KeBugCheckEx (81dfe654)
81e23074 cc              int     3
81e23075 33c0            xor     eax,eax
81e23077 e9823dedff      jmp     nt!KeAbPreAcquire+0xfe (81cf6dfe)  Branch

nt!KeAbPreAcquire+0xfe:
81cf6dfe 5f              pop     edi
81cf6dff 5e              pop     esi
81cf6e00 8be5            mov     esp,ebp
81cf6e02 5d              pop     ebp
81cf6e03 8be3            mov     esp,ebx
81cf6e05 5b              pop     ebx
81cf6e06 c20400          ret     4

nt!ObFastDereferenceObject:
81cf3f40 8bff            mov     edi,edi
81cf3f42 56              push    esi
81cf3f43 8b31            mov     esi,dword ptr [ecx]
81cf3f45 8bc6            mov     eax,esi
81cf3f47 57              push    edi
81cf3f48 8bfa            mov     edi,edx
81cf3f4a 33c7            xor     eax,edi
81cf3f4c 83f807          cmp     eax,7
81cf3f4f 7310            jae     nt!ObFastDereferenceObject+0x21 (81cf3f61)  Branch

nt!ObFastDereferenceObject+0x11:
81cf3f51 8d5601          lea     edx,[esi+1]
81cf3f54 8bc6            mov     eax,esi
81cf3f56 f00fb111        lock cmpxchg dword ptr [ecx],edx
81cf3f5a 3bc6            cmp     eax,esi
81cf3f5c 750c            jne     nt!ObFastDereferenceObject+0x2a (81cf3f6a)  Branch

nt!ObFastDereferenceObject+0x1e:
81cf3f5e 5f              pop     edi
81cf3f5f 5e              pop     esi
81cf3f60 c3              ret

nt!ObFastDereferenceObject+0x21:
81cf3f61 8bcf            mov     ecx,edi
81cf3f63 5f              pop     edi
81cf3f64 5e              pop     esi
81cf3f65 e906290000      jmp     nt!ObfDereferenceObject (81cf6870)  Branch

nt!ObFastDereferenceObject+0x2a:
81cf3f6a 8bf0            mov     esi,eax
81cf3f6c 33c7            xor     eax,edi
81cf3f6e 83f807          cmp     eax,7
81cf3f71 72de            jb      nt!ObFastDereferenceObject+0x11 (81cf3f51)  Branch

nt!ObFastDereferenceObject+0x33:
81cf3f73 ebec            jmp     nt!ObFastDereferenceObject+0x21 (81cf3f61)  Branch
```