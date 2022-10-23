# Coin Flip

## Challenge

You have one chance to solve each board. More information is provided in
the challenge.

Some more information regarding the challenge:

- Initially, Bob sets up the coins in the chessboard completely
  randomly.
- After, that, he shows Alice which is the answer coin.
- Then, Alice flips a coin strategically so as to communicate to you
  which coin is the correct one.
- After all of these tasks are done, you see the board and you have to
  figure out in one go which is the correct coin. You do not know the
  initial configuration of the board, neither the coin which Alice
  flipped.
- You have to answer the coin index (starting from 0) below which the
  paper is there.

<!-- -->

    nc 34.76.206.46 10007

## Solution

The socket returns a more detailed explanation:

    You and two of your friends (let's call them Alice and Bob) are playing a game.
    There is a chessboard, with coins in each square. Bob takes a small piece of paper and hides it under one of these coins.
    Now, he shows this to Alice but not to you. Alice's task is to flip one (AND ONLY ONE) coin from the board and using this information, you have to guess the position of the piece of paper.

    Some rules:
    -> Alice cannot communicate to you in any way after flipping the coin.
    -> Assume that Alice has already flipped the coin.
    -> You just need to find the position of the piece of paper.

    Best of luck :)

    BOARD NUMBER - 0
    ---------------------------------
    | H | H | T | H | H | T | H | T |
    ---------------------------------
    | T | H | T | T | H | T | H | H |
    ---------------------------------
    | H | T | T | T | H | T | H | T |
    ---------------------------------
    | T | T | H | T | H | H | T | H |
    ---------------------------------
    | T | H | H | H | H | T | H | T |
    ---------------------------------
    | H | T | T | T | T | H | H | H |
    ---------------------------------
    | T | H | H | T | T | H | T | T |
    ---------------------------------
    | T | T | H | H | T | H | T | T |
    ---------------------------------

    Enter the position of the key:

I searched a bit and this is actually a fairly well known puzzle.
There's some very good videos on YouTube on how it works, but I ended up
using [this
article](https://datagenetics.com/blog/december12014/index.html).

We need to parse the board and subdivide it into sections. Each section
has a parity depending on the number of heads it contains. The answer
will be the binary number formed by the parities of all sections.

A bit of guessing was necessary to figure out how the board was divided
and the order of the sections.

Here's the script with the solution:

    #!/usr/bin/env python3

    from pwn import remote, context

    context.update(log_level="debug")

    N = 8

    r = remote("34.76.206.46", 10007)

    def parity(l):
        count = 0
        for i in l:
            for j in i:
                if j == "H":
                    count += 1

        return "0" if count % 2 == 0 else "1"

    def do_round(n):
        r.recvuntil(f"BOARD NUMBER - {n}\n".encode("utf-8"))

        rows = []
        for i in range(N):
            r.recvline()
            row = [s.strip() for s in r.recvline().decode("utf-8").split("|")[1:-1]]
            rows.append(row)

        cols = []
        for i in range(N):
            col = []
            for row in rows:
                col.append(row[i])
            cols.append(col)

        regions = [
            [rows[4], rows[5], rows[6], rows[7]],
            [rows[2], rows[3], rows[6], rows[7]],
            [rows[1], rows[3], rows[5], rows[7]],
            [cols[4], cols[5], cols[6], cols[7]],
            [cols[2], cols[3], cols[6], cols[7]],
            [cols[1], cols[3], cols[5], cols[7]],
        ]

        binary = "".join([parity(region) for region in regions])
        answer = str(int(binary, 2))
        r.sendline(answer.encode("utf-8"))

    def main():
        for i in range(100):
            do_round(i)
        r.interactive()

    if __name__ == "__main__":
        main()

After running it, 100 boards are solved and we get the flag.

    Congratulations! You won
    Here's your flag :) --> jadeCTF{n3v3r_pl4y3d_ch355_l1k3_th1s}

Flag: `jadeCTF{n3v3r_pl4y3d_ch355_l1k3_th1s}`
