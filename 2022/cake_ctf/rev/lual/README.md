# luau

<span id="-rev"></span><span id="rev" class="tag">rev</span> <span
id="-lua"></span><span id="lua" class="tag">lua</span>

## Analysis

We get a `main.lua` file.

    local libflag = require "libflag"
    io.write("FLAG: ")
    flag = io.read("*l")
    if libflag.checkFlag(flag, "CakeCTF 2022") then
       print("Correct!")
    else
       print("Wrong...")
    end

We also get a `libflag.lua` compiled library. The version of Lua used is
`5.3`.

    luau$ file libflag.lua
    libflag.lua: Lua bytecode, version 5.3

I found a Lua decompiler called
[luadec](https://github.com/viruscamp/luadec) which kind of supports
5.3. Unfortunately, it segfaulted when I tried to run it. With some
flags esoj was able to get semi-readable code, but I found it easier to
just read the assembly.

    luau$ solve/luadec/luadec/luadec -dis libflag.lua > assembly

The code first initializes a big array with a bunch of values in `R2`.

        0 [-]: NEWTABLE  R2 26 0      ; R2 := {} (size = 26,0)
        1 [-]: LOADK     R3 K0        ; R3 := 62
        2 [-]: LOADK     R4 K1        ; R4 := 85
    <snip>
       37 [-]: LOADK     R39 K31      ; R39 := 34
       38 [-]: LOADK     R40 K31      ; R40 := 34
       39 [-]: SETLIST   R2 38 1      ; R2[0] to R2[37] := R3 to R40 ; R(a)[(c-1)*FPF+i] := R(a+i), 1 <= i <= b, a=2, b=38, c=1, FPF=50

Then it checks whether the flag (in `R0`) has the same length as the big
array. If it doesn't, it just returns `false`.

       40 [-]: LEN       R3 R0        ; R3 := #R0
       41 [-]: LEN       R4 R2        ; R4 := #R2
       42 [-]: EQ        1 R3 R4      ; if R3 ~= R4 then goto 44 else goto 46
       43 [-]: JMP       R0 2         ; PC += 2 (goto 46)
       44 [-]: LOADBOOL  R3 0 0       ; R3 := false
       45 [-]: RETURN    R3 2         ; return R3

It then creates two new arrays in `R3` and `R4`.

       46 [-]: NEWTABLE  R3 0 0       ; R3 := {} (size = 0,0)
       47 [-]: NEWTABLE  R4 0 0       ; R4 := {} (size = 0,0)

All characters from the flag get converted to bytes and put into `R3`;
this is done using the `sub` and `byte` functions. The same is then done
for the second argument (in `R1`), but it's stored in `R4` instead.

       48 [-]: LOADK     R5 K32       ; R5 := 1
       49 [-]: LEN       R6 R0        ; R6 := #R0
       50 [-]: LOADK     R7 K32       ; R7 := 1
       51 [-]: FORPREP   R5 8         ; R5 -= R7; pc += 8 (goto 60)
       52 [-]: GETTABUP  R9 U0 K33    ; R9 := U0["string"]
       53 [-]: GETTABLE  R9 R9 K34    ; R9 := R9["byte"]
       54 [-]: SELF      R10 R0 K35   ; R11 := R0; R10 := R0["sub"]
       55 [-]: MOVE      R12 R8       ; R12 := R8
       56 [-]: ADD       R13 R8 K32   ; R13 := R8 + 1
       57 [-]: CALL      R10 4 0      ; R10 to top := R10(R11 to R13)
       58 [-]: CALL      R9 0 2       ; R9 := R9(R10 to top)
       59 [-]: SETTABLE  R3 R8 R9     ; R3[R8] := R9
       60 [-]: FORLOOP   R5 -9        ; R5 += R7; if R5 <= R6 then R8 := R5; PC += -9 , goto 52 end

It then swaps bytes of `R3` in two nested `for` loops.

       74 [-]: LOADK     R5 K32       ; R5 := 1
       75 [-]: LEN       R6 R3        ; R6 := #R3
       76 [-]: LOADK     R7 K32       ; R7 := 1
       77 [-]: FORPREP   R5 9         ; R5 -= R7; pc += 9 (goto 87)
       78 [-]: ADD       R9 R8 K32    ; R9 := R8 + 1
       79 [-]: LEN       R10 R3       ; R10 := #R3
       80 [-]: LOADK     R11 K32      ; R11 := 1
       81 [-]: FORPREP   R9 4         ; R9 -= R11; pc += 4 (goto 86)
       82 [-]: GETTABLE  R13 R3 R8    ; R13 := R3[R8]
       83 [-]: GETTABLE  R14 R3 R12   ; R14 := R3[R12]
       84 [-]: SETTABLE  R3 R8 R14    ; R3[R8] := R14
       85 [-]: SETTABLE  R3 R12 R13   ; R3[R12] := R13
       86 [-]: FORLOOP   R9 -5        ; R9 += R11; if R9 <= R10 then R12 := R9; PC += -5 , goto 82 end
       87 [-]: FORLOOP   R5 -10       ; R5 += R7; if R5 <= R6 then R8 := R5; PC += -10 , goto 78 end

The final part is inside the following loop:

       88 [-]: LOADK     R5 K32       ; R5 := 1
       89 [-]: LEN       R6 R3        ; R6 := #R3
       90 [-]: LOADK     R7 K32       ; R7 := 1
       91 [-]: FORPREP   R5 14        ; R5 -= R7; pc += 14 (goto 106)
       <snip>
      106 [-]: FORLOOP   R5 -15       ; R5 += R7; if R5 <= R6 then R8 := R5; PC += -15 , goto 92 end

It stores `R3[i]` into `R9` and then stores `R4[i % #R4]` into `R10`.
There's some plus-ones and minus-ones thrown around because in Lua
indexing starts with 1. Since the solution is going to be in Python
anyway, I'll just ignore those.

       92 [-]: GETTABLE  R9 R3 R8     ; R9 := R3[R8]
       93 [-]: SUB       R10 R8 K32   ; R10 := R8 - 1
       94 [-]: LEN       R11 R4       ; R11 := #R4
       95 [-]: MOD       R10 R10 R11  ; R10 := R10 % R11
       96 [-]: ADD       R10 K32 R10  ; R10 := 1 + R10
       97 [-]: GETTABLE  R10 R4 R10   ; R10 := R4[R10]

It then XORs our two values and checks whether the result is equal to
`R2[i]`. If it's not, we return `false`. If we pass this check for every
byte, we return `true`.

       98 [-]: BXOR      R9 R9 R10    ; R9 := R9 ~ R10
       99 [-]: SETTABLE  R3 R8 R9     ; R3[R8] := R9
      100 [-]: GETTABLE  R9 R3 R8     ; R9 := R3[R8]
      101 [-]: GETTABLE  R10 R2 R8    ; R10 := R2[R8]
      102 [-]: EQ        1 R9 R10     ; if R9 ~= R10 then goto 104 else goto 106
      103 [-]: JMP       R0 2         ; PC += 2 (goto 106)
      104 [-]: LOADBOOL  R9 0 0       ; R9 := false
      105 [-]: RETURN    R9 2         ; return R9
      106 [-]: FORLOOP   R5 -15       ; R5 += R7; if R5 <= R6 then R8 := R5; PC += -15 , goto 92 end
      107 [-]: LOADBOOL  R5 1 0       ; R5 := true
      108 [-]: RETURN    R5 2         ; return R5
      109 [-]: RETURN    R0 1         ; return 

## Solution

The only part of the code that actually does something is the last `for`
loop with the XOR cypher, so we just really need to undo the XOR.

There's also the scrambling loop, but you can still figure out the flag
even if you ignore it. I still undid the scrambling anyway just because.

    #!/usr/bin/env python3

    r2 = bytearray(b"\x3e\x55\x19\x54\x2f\x38\x76\x47\x6d\x00\x5a\x47\x73\x09\x1e\x3a\x20\x65\x28\x14\x42\x6f\x03\x5c\x77\x16\x5a\x0b\x77\x23\x3d\x66\x66\x73\x57\x59\x22\x22")
    r3 = bytearray()
    r4 = bytearray(b"CakeCTF 2022")

    for i in range(len(r2)):
        # r4[index] ^ r3[i] == r2[i]
        index = i % len(r4)
        r3.append(r4[index] ^ r2[i])

    for i in range(len(r3)):
        for j in range(i + 1, len(r3)):
            r3[i], r3[j] = r3[j], r3[i]

    print(r3.decode("utf-8"))

And with that we get the flag: `CakeCTF{w4n1w4n1_p4n1c_uh0uh0_g0ll1r4}`
