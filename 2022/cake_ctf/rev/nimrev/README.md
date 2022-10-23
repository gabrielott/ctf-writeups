# nimrev

<span id="-rev"></span><span id="rev" class="tag">rev</span> <span
id="-nim"></span><span id="nim" class="tag">nim</span>

This is a binary written in Nim. Here's what we get when we run it.

    nimmrev$ ./chall
    hello
    Wrong...
    nimrev$

With Ghidra I was able to find a function called `NimMainModule`, which
is where the string `Wrong...` gets printed. If the `password` is equal
to the `answer` we got the password right.

    eq = eqStrings(password,answer);
    if (eq == '\x01') {
      local_18 = copyString(&TM__V45tF8B8NBcxFcjfe7lhBw_4);
    }
    else {
      local_18 = copyString(&TM__V45tF8B8NBcxFcjfe7lhBw_5);
    }

There's a bunch of stuff at the beginning of the function, which is most
certainly just the flag obfuscated. Instead of trying to figure it all
out I decided to try and get the flag with `gdb`.

I put a breakpoint at the `eqStrings` call in order to figure out what
it was receiving as arguments.

     → 0x55555555efc6 <NimMainModule+459> call   0x55555555ec46 <eqStrings>
       ↳  0x55555555ec46 <eqStrings+0>    push   rbp
          0x55555555ec47 <eqStrings+1>    mov    rbp, rsp
          0x55555555ec4a <eqStrings+4>    sub    rsp, 0x30
          0x55555555ec4e <eqStrings+8>    mov    QWORD PTR [rbp-0x28], rdi
          0x55555555ec52 <eqStrings+12>   mov    QWORD PTR [rbp-0x30], rsi
          0x55555555ec56 <eqStrings+16>   mov    BYTE PTR [rbp-0x11], 0x0
    ────────────────────────────────────────────────────────────────────────────────────────────── arguments (guessed) ────
    eqStrings (
       $rdi = 0x00007ffff7d2d050 → 0x0000000000000005,
       $rsi = 0x00007ffff7d2e0d0 → 0x0000000000000018,
       $rdx = 0x00007ffff7d2e0d0 → 0x0000000000000018
    )

We can see that `rdi` and `rsi` are pointers to somewhere. I googled a
bit and found [this](https://stackoverflow.com/a/29411498) simple
explanation of how strings work in Nim.

> So a string is a raw pointer to an object with a len, reserved and
> data field.

This means that the `0x5` `rdi` points to is the length of `"hello"`,
which makes sense. In addition, the flag is `0x18` bytes long.

Using `gef`'s `dereference` command we can figure out what else is
around that address.

    i_gef➤  dereference 0x00007ffff7d2e0d0
    0x00007ffff7d2e118│+0x0048: 0x0000000000000000
    0x00007ffff7d2e110│+0x0040: 0x0000000000000000
    0x00007ffff7d2e108│+0x0038: 0x0000000000000000
    0x00007ffff7d2e100│+0x0030: 0x0000000000000000
    0x00007ffff7d2e0f8│+0x0028: 0x0000000000000000
    0x00007ffff7d2e0f0│+0x0020: "s_n0t_C}"
    0x00007ffff7d2e0e8│+0x0018: "s0m3t1m3s_n0t_C}"
    0x00007ffff7d2e0e0│+0x0010: "CakeCTF{s0m3t1m3s_n0t_C}"
    0x00007ffff7d2e0d8│+0x0008: 0x000000000000001c
    0x00007ffff7d2e0d0│+0x0000: 0x0000000000000018   ← $rdx, $rsi

And there's the flag: `CakeCTF{s0m3t1m3s_n0t_C}`.
