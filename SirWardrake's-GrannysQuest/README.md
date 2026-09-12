# CrackMe Write-Up: SirWardrake's GrannysQuest

| | |
|---|---|
| **Author (of CrackMe)** | [SirWardrake](https://crackmes.one/user/SirWardrake) |
| **Language** | C/C++ |
| **Platform** | Windows (x86-64) |
| **Difficulty** | 3.5 |
| **Quality** | 4.2 |
| **Upload Date** | 2026-02-19 |
| **Labels** | KeyGen, Reversing, CPU Exploitation, Hash Mixing |

---

## Day 1: Grannys teeth

![Day 1 Start](images/day1_grandmas_birthday.png)

Upon starting the crackme we get this message:

"what about a bad day to be a nephew
i am not gonna take guesses or grandma might complain i thought she was born in the medival era"

So let's take a look at the code:

```c
if (x1 == ((0x3A24 + 0x79) * 2026.0 - 120739968.0) / 2.0 - (double)(dat + 0x49)) {
    // pass
}
```

Seems like time for kindergarten math. Ok maybe not kindergarten now let's calculate this equation.

![Equation Analysis](images/day1_equation_analysis.png)

`dat = 0x7A`

```
0x3A24 = (double)pow(dat,2);
0x79 = (double)pow(0xb,2);
((0x3A24 + 0x79) * 2026.0 - 120739968.0) / 2.0 - (double)(dat + 0x49)
```

gives `0xF320`

`x1` must equal this value to pass, but what's `x1`?

```c
iVar1 = FUN_1400038f0();
x1 = (double)iVar1;
```

It is the return value of this function. Now let's check this function:

![Keygen Function](images/day1_keygen_function.png)

```c
int FUN_140002360(basic_string<> *Input) {
    int x1 = 0x29a;  // 666
    
    if ((int)CopyInput == 8) {
        for (Index = 0; Index < 8; Index++) {
            c_Char = std::basic_string<>::operator[](Input, Index);
            
            if ((*c_Char < '0') || ('9' < *c_Char)) {
                return -1;  // Not a digit
            }
            
            x1 = x1 + *c_Char;  // Sum ASCII values
        }
    }
    
    zx = _rotl(x1 - 0x29aU ^ 0x29a, 4);
    iVar1 = (int)zx * 5;
    return iVar1;
}
```

So we take user input. It must be 8 long. If it isn't a digit we return -1. Then we sum the values and do the math.

The math:
```
iVar1 = 0xF320 = 62240
zx * 5 = 62240
zx = 12448 (0x30A0)

zx = _rotl(x1 - 0x29a ^ 0x29a, 4)
so we want to get x1

rotr(0x30A0, 4) then xor 0x29a then plus 0x29a

But why not the plus first? Because we are doing inverse. Normally the plus is first but xor is first in inverse.

x1 = x1 + *c_Char;

So we got x final = 1066 and initial = 666
And 8 letters huh.

So x_final = x_initial + summation of chars from 0 to 7
summation of chars from 0 to 7 = 400
and each character is ascii digit so starts from 48, 0 = 48

∑(digit_i + 48) = 400 
∑digit_i + 384 = 400 
∑digit_i = 16
```

So the password is any 8 digits that give us 16.

Which is `22222222`

Damn grandma is so old.

✅ **Day 1 Solution: `22222222`**

---

## Day 2: Grannys rollator walker

![Day 2 Walker Wheel](images/day2_manufacturer_walker.png)

Now for day 2:

```c
output((longlong *)cout_exref,"Manufacturer: ");
::input((longlong *)cin_exref,Input);
local_60 = local_50;
input = (basic_string<> *)FUN_140005130(local_60,Input);
condition = operation(input);
flag = (uint)condition;
```

Let's check the operation function:

```c
index = 0;
while( true ) {
    uVar1 = FUN_140004ee0(0x14000c248);
    if (uVar1 <= (ulonglong)(longlong)index) {
        ~basic_string<>((undefined8 *)param1);
        return 1;
    }
    c_char = std::basic_string<>::operator[]((basic_string<> *)&dat,(longlong)index);
    if (*c_char < 97) {
        c_char = std::basic_string<>::operator[]((basic_string<> *)&dat,(longlong)index);
        if (*c_char < 65) {
            local_38 = 45;
        }
        else {
            c_char = std::basic_string<>::operator[]((basic_string<> *)&dat,(longlong)index);
            local_38 = -0x65 - *c_char;
        }
    }
    else {
        c_char = std::basic_string<>::operator[]((basic_string<> *)&dat,(longlong)index);
        local_38 = -0x25 - *c_char;
    }
    std::basic_string<>::operator[]((basic_string<> *)&dat,(longlong)index);
    std::basic_string<>::operator[](param1,(longlong)index);
    c_char = std::basic_string<>::operator[](param1,(longlong)index);
    if (local_38 != *c_char) break;
    index = index + 1;
}
~basic_string<>((undefined8 *)param1);
return 0;
```

Interesting. So we are just checking if current letter is a character or digit and do an operation if character and different if digit.

We are checking if the thing at `&dat` is equal to current char in `param1` which is our input and if false it is over.

So now let's check `dat`:

```c
void FUN_140001000(void) {
    Output(&dat,"Dlyyov#Dsvvoh#Fmornrgvw");
    atexit(FUN_1400079c0);
    return;
}
```

`Dlyyov#Dsvvoh#Fmornrgvw`

Amazing that's what we got.

So now we gotta do the math on it to get the correct param. Here's the code:

![Validation Code](images/day2_validation_code.png)

```python
def process(s: str) -> str:
    out = []
    for ch in s:
        val = ord(ch)

        if val < 97:
            if val < 65:
                res = 45
            else:
                res = (-0x65 - val) & 0xFF
        else:
            res = (-0x25 - val) & 0xFF

        out.append(chr(res))

    return "".join(out)

user_input = input("Enter input: ")
output = process(user_input)

print(f"ASCII Output: {output}")
```

Enter input: `Dlyyov#Dsvvoh#Fmornrgvw`

ASCII Output: `Wobble-Wheels-Unlimited`

Amazing correct.

✅ **Day 2 Solution: `Wobble-Wheels-Unlimited`**

---

## Day 3: Grannys autistic traits and what numbers have to do with them

![Day 3 Setup](images/day3_grannys_autistic_traits.png)

"As you know, Grandma's a little on the autistic side... Which means she only remembers things on names once she turned them into numbers.
She's forgotten your name again... So better start calculating before she gives you a new one!"

Your name: asking for my name huh, omar for sure no tricks this time.

Crazy number code? a weird name for my phone number.

Now let's check the code:

```c
do {
    output((longlong *)cout_exref,"\nYour name: ");
    input((longlong *)cin_exref,(_String_val<> *)&Name);
    LengthOfName = FUN_140004ee0(0x14000c0c0);
} while (8 < LengthOfName);

output((longlong *)cout_exref,"\nYour crazy number code: ");
input((longlong *)cin_exref,NumberInput);
local_60 = local_50;
CopyInput = (basic_string<> *)FUN_140005130(local_60,NumberInput);
flag0 = operation(CopyInput);
flag = (uint)flag0;
```

Similar pattern again.

```c
int Fib(int param_1) {
    if (1 < param_1) {
        iVar1 = Fib(param_1 + -1);
        iVar2 = Fib(param_1 + -2);
        param_1 = iVar1 + iVar2;
    }
    return param_1;
}

LetterFibResult = Fib(lowercaseLetter + -0x61);
```

Interesting. We do fibonacci on lowercased name + `-0x61` (which is -'a').

```c
lowercaseLetter = tolower((int)*pointer);
LetterFibResult = Fib(lowercaseLetter + -0x61);
local_60 = (undefined8 *)FUN_1400018d0((undefined1 *)local_50,LetterFibResult);
stringLoading((undefined8 *)local_30,local_60);
```

So just like previous day we just take input and compare it with the calculated local_30.

We just do lowercase on name and fibonacci.

![Fibonacci Function](images/day3_fibonacci_function.png)

```python
def fib(n):
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a

def keygen(name: str) -> str:
    code = ""
    for ch in name:
        if ch.isalpha():
            idx = ord(ch.lower()) - ord('a')
            code += str(fib(idx))
    return code

# Quick test
name = "omar"
print(f"Name: {name}")
print(f"Key:  {keygen(name)}")
```

Output:
```
Name: omar
Key:  37714401597
```

![Complete Output](images/day3_complete_output.png)

✅ **Day 3 Solution for "omar": `37714401597`**

---

## Day 4: The Rocket Launch

![Day 4 Setup](images/day4_rocket_launch.png)

Grandma's got a problem: Her wicked daughters, your oh-so-charming aunts, have stashed a lot of cash, time to dump her in a nursing home. Needless to say, Grandma strongly disagrees. But don't worry, she's got a plan: Down in the basement, she's still got that old rocket her long-dead husband smuggled home from Vietnam. It's ancient, sure, but probably still works. The only snag? She's forgotten the launch code, not shocking considering she's older than the Jurassic era.

Luckily, your grandma once sang you a lullaby that hid the code inside. Time to remember it...

Oh, and by the way, your lovely aunts are conveniently on vacation right now. Grandma would like their exact coordinates. You know... for "planning purposes".

**Launch Code:** `111-11111-111`

**The sisters' coordinates:** `40.8214,14.4262`

**Verification number for launch sequence and targets:** (need to calculate)

---

Now for the final validation:

```c
undefined4 FUN_140003050(_String_val<> *launchcode,basic_string<> *cord) {
    bool bVar1;
    bool bVar2;
    undefined4 uVar3;
    ulonglong flagcord;
    
    flagcord = FUN_140002f20(cord);
    if ((flagcord & 0xff) == 0) {
        ~basic_string<>((undefined8 *)launchcode);
        ~basic_string<>((undefined8 *)cord);
        return 11;
    }
    
    SegCord = (_String_val<> *)FUN_140004e80((_String_val<> *)cord,(undefined1 *)local_30,0,7);
    bVar2 = false;
    dVar4 = FUN_140001800(SegCord,(longlong *)0x0);
    dVar5 = FUN_140002820((_String_val<> *)&Name,launchcode,1);
    if (dVar5 == dVar4) {
        SegCord = (_String_val<> *)FUN_140004e80((_String_val<> *)cord,(undefined1 *)local_50,8,7);
        bVar2 = true;
        dVar4 = FUN_140001800(SegCord,(longlong *)0x0);
        dVar5 = FUN_140002820((_String_val<> *)&Name,launchcode,2);
        if (dVar5 == dVar4) {
            bVar1 = false;
            goto LAB_14000319f;
        }
    }
    bVar1 = true;
}
```

We divide coordinates into segments and check format. Then we call this function with param values 1 and 2:

```c
double FUN_140002820(_String_val<> *Name,_String_val<> *LaunchCode,int param_3) {
    // ... 50+ lines of complex math ...
    
    if (param_3 == 1) {
        local_50 = 40.8214;
    }
    else {
        local_50 = 14.4262;
    }
    
    dVar6 = local_50 + (dVar8 - dVar8) + ((double)HashName - (double)HashName) +
            ((double)uVar1 - (double)uVar1) + ((double)param_3 - (double)param_3);
}
```

You see this heavy math? I fell into the trap once. We ain't using any of it lmao.

```
dVar6 = local_50 + (dVar8 - dVar8) + ((double)HashName - (double)HashName) + ...
```

It is all 0 except `local_50`.

```
if (param_3 == 1) {
    local_50 = 40.8214;
}
else {
    local_50 = 14.4262;
}
```

Our coordinates bro.

After the coordinates return check:

```c
Copy = (_String_val<> *)FUN_140005130(local_30,PIN);
bVar1 = FUN_1400034a0((_String_val<> *)LaunchCode,Cord,Copy);
if (bVar1) {
    flag = 1;
}
```

Then for the final validator:

```c
SegCord = (_String_val<> *)FUN_140004e80(cord,(undefined1 *)local_30,8,7);
SegCord_00 = (_String_val<> *)FUN_140004e80(cord,(undefined1 *)local_50,0,7);
hashLaunchCode = hashing(launchCode);
value = FUN_140004ee0(0x14000c0c0);  // len(Name)
local_88 = (double)(ulonglong)(value * hashLaunchCode);
dVar2 = FUN_140001800(SegCord_00,(longlong *)0x0);
dVar3 = FUN_140001800(SegCord,(longlong *)0x0);
local_a0 = (double)xoring((ulonglong)(dVar3 + dVar2),0x4084d00000000000);
local_a0 = local_a0 + local_88;
for (; local_a0 < 0.0; local_a0 = local_a0 * 100.0) {}
for (; local_a0 < 1000000000.0; local_a0 = local_a0 * 100.0) {}
dVar2 = (double)round(local_a0);
nameLength = FUN_140004ee0(0x14000c0c0);
uVar1 = FUN_140001730(Pin,(longlong *)0x0,10);  // std::stoll(Pin, 10)
~basic_string<>((undefined8 *)Pin);
return ((longlong)dVar2 ^ nameLength * 0x309) == uVar1;
```

**Decompiler Pitfalls vs. Assembly Reality**

Relying strictly on high-level C decompiler output fails here due to three low-level CPU / compiler behaviors:

**CDQE Truncation on Hash:**

`hashing()` returns an unsigned 64-bit FNV-1a hash, but the calling function processes it via CDQE (0x14000351d), sign-extending EAX into RAX.

For "111-11111-111", FNV-1a produces `0xCA401C46A832B068`.
Lower 32 bits: `0xA832B068` (bit 31 is set).
Sign-extended: `0xFFFFFFFF A832B068` = `-1473073048`.

**IEEE-754 Raw Bitwise XOR:**

`xoring` does not XOR integers; it moves the raw 64-bit bit patterns of (40.8214 + 14.4262 = 55.2476) and 666.0 (0x4084D00000000000) across XMM registers, yielding an astronomically small value (~10^-305) that is completely negligible.

**CVTTSD2SI Integer Indefinite Overflow:**

`local_88` evaluates to 4 × 0xFFFFFFFFA832B068 ≈ 1.844674 × 10^19.

Since this value exceeds INT64_MAX (9.22 × 10^18), the x86 CVTTSD2SI instruction overflows and returns the integer indefinite value: 0x8000000000000000.

**Calculating the Final PIN**

The validation check reduces directly to:

```
Expected PIN = 0x8000000000000000 ⊕ (len(Name) × 0x309)
```

For Name = "omar" (length = 4):

```
Expected PIN = 0x8000000000000000 ⊕ (4 × 777)
Expected PIN = 0x8000000000000000 ⊕ 3108 = 0x8000000000000C24
```

As a signed 64-bit integer:

PIN: `-9223372036854772700`

![Final Success](images/day4_final_success.png)

✅ **Day 4 Solution for "omar": `-9223372036854772700`**

---

## Summary of Valid Inputs

| Prompt | Input Value |
|--------|------------|
| **Granny's B-Day (ddmmyyyy)** | 22222222 |
| **Manufacturer** | Wobble-Wheels-Unlimited |
| **Your name** | omar |
| **Launch Code** | 111-11111-111 |
| **The sisters' coordinates** | 40.8214,14.4262 |
| **Verification number** | -9223372036854772700 |

**Result:**

Yay! The sisters are off the map, the nursing home's off the list, and Grandma's on cloud nine! Way to go, dude!

---

## Key Insights

1. **Dead Code is Everywhere** — The 50-line hash mixing function is pure obfuscation. Only `local_50` matters.

2. **CPU-Level Exploitation** — CDQE sign-extension, IEEE-754 bitwise XOR, and CVTTSD2SI overflow are the real tricks. Decompilers hide this.

3. **Static Analysis Wins** — You don't need to run this to solve it. Ghidra + assembly understanding = complete solution.

---

*Thanks to SirWardrake for the well-designed challenge. Damn grandma is so old.*
