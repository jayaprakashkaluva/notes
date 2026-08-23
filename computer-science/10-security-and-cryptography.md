# Security & Cryptography: The Adversarial Lens

The capstone that re-reads every previous document with a hostile reader in mind. Assumes the full stack: [hardware](01-how-computers-work.md), [programs](02-programming-fundamentals.md), [OS](operating-systems.md), [networks](05-computer-networks.md), [databases](06-databases.md), [theory](07-theory-of-computation.md). Security is not a feature but a property of the whole system under attack — and it's defined by its weakest layer. This document covers how to think adversarially, the cryptographic toolkit, the attacks that actually happen, and defensive engineering. (Framing: everything here is for building and *authorized* testing of defenses.)

---

## Table of Contents

1. [Thinking Like an Attacker (So You Can Defend)](#1--thinking-like-an-attacker-so-you-can-defend)
2. [Cryptography I: Symmetric Encryption and Hashes](#2--cryptography-i-symmetric-encryption-and-hashes)
3. [Cryptography II: Public Keys, Signatures, Key Exchange](#3--cryptography-ii-public-keys-signatures-key-exchange)
4. [Cryptography III: The Rules of Use](#4--cryptography-iii-the-rules-of-use)
5. [Authentication: Passwords and Beyond](#5--authentication-passwords-and-beyond)
6. [The Classic Attacks on Software](#6--the-classic-attacks-on-software)
7. [Web Security](#7--web-security)
8. [The Human Layer and Operations](#8--the-human-layer-and-operations)
9. [Secure Design Principles](#9--secure-design-principles)
10. [The Road to Expertise](#10--the-road-to-expertise)

---

# 1 — Thinking Like an Attacker (So You Can Defend)

The mindset shift that defines the field: every other document asked "does it work?"; security asks **"what happens when someone *makes* it misbehave on purpose?"** Your program's input is no longer data — it's a *message from an adversary* crafted to hit the case you didn't handle. Ground rules of the discipline:

- **Define the threat model** first: what assets, against which attackers, with what capabilities? "Secure" is meaningless unqualified — a diary is secure against a sibling, not the NSA, and *that's fine if the sibling is the threat model*. Over-defending costs; under-defending costs more; defending the wrong thing costs everything.
- **CIA** — the three properties at stake: **Confidentiality** (secrets stay secret), **Integrity** (data/code isn't tampered), **Availability** (the system stays up — DoS attacks target this leg). Add **authentication** (who are you?) and **authorization** (what may you do?) — *different questions*, chronically conflated.
- **Asymmetry**: the defender must close every hole; the attacker needs one. Hence defense in depth (§9) rather than one perfect wall.
- **Security is economics**: attacks happen when expected profit exceeds cost; defense raises attacker cost and lowers blast radius, rarely achieving "impossible." Corollary — most real compromises are not exotic zero-days but the boring reliables: phishing, unpatched known bugs, stolen/reused passwords, misconfiguration (§8).
- **Kerckhoffs's principle**: the system must stay secure even if the attacker knows *everything but the key*. Secrecy of design ("security through obscurity") always leaks; secrecy of *keys* is manageable. This is why real cryptography is public, standardized, and brutally peer-reviewed — and why "we invented our own encryption" is a red flag, not a moat.

# 2 — Cryptography I: Symmetric Encryption and Hashes

Cryptography is math that survives the threat model — mechanisms whose security *reduces to* problems believed hard ([doc 07 §6](07-theory-of-computation.md): easy to verify with the key, infeasible to invert without it).

## 2.1 Symmetric encryption — one shared key

Same key encrypts and decrypts. The standard: **AES** (or ChaCha20), operating on data via a **mode**. What a practitioner must know:

- Key sizes 128/256 bits are not brute-forceable — 2¹²⁸ trials outlasts the sun (doc 01 §1.2's exponentials, now working *for* you). Attacks therefore target everything *around* the cipher: modes, keys, implementations, protocols (§4).
- **ECB mode is broken by design** — identical plaintext blocks → identical ciphertext blocks; the encrypted-penguin image (still visibly a penguin) is the canonical demonstration. Modern practice: **authenticated encryption (AEAD)** — AES-GCM or ChaCha20-Poly1305 — which encrypts *and* integrity-protects in one construction, because encryption without integrity is a trap: attackers flip ciphertext bits to flip plaintext bits (ciphertext malleability), and padding-oracle attacks decrypted whole messages from nothing but error-message differences. **Confidentiality without integrity is not confidentiality.**
- Modes need a **nonce/IV** — a never-repeated value per encryption; *reusing a nonce with the same key* is catastrophic (GCM: total integrity collapse; stream ciphers: XOR of plaintexts revealed — how WEP Wi-Fi died).

## 2.2 Cryptographic hashes — fingerprints

A **hash function** (SHA-256, SHA-3, BLAKE3) maps any input to a fixed 256-bit digest, with three escalating properties: **preimage resistance** (given digest, can't find an input), **second-preimage resistance** (given input, can't find a twin), **collision resistance** (can't find *any* two inputs colliding — hardest, and first to fall: broken MD5 and SHA-1 fell here; birthday paradox, [doc 07 §1.3](07-theory-of-computation.md), halves the exponent: 256-bit hash → 2¹²⁸ collision work). One bit of input change flips ~half the output bits (avalanche).

Uses everywhere: file/download integrity, git commit IDs (doc 09 §2 — content-addressing *is* hashing), deduplication, blockchain linking, password storage (§5 — with crucial modifications), and **HMAC** (hash + shared key = message authentication: proof of integrity *and* origin — how cookies are signed and API requests authenticated). Note the difference from doc 03 §6's hash-table hashes: those optimize speed and spread; these must survive an adversary — never substitute one for the other.

# 3 — Cryptography II: Public Keys, Signatures, Key Exchange

Symmetric crypto has a bootstrap problem: how do strangers agree on a key over a wire the adversary reads? The 1976 revolution (Diffie-Hellman-Merkle; RSA 1977): **key pairs** — a public key anyone may know, a private key never shared, linked by one-way math (factoring for RSA; discrete logs on elliptic curves for the modern, smaller-key standard: ECDH/Ed25519).

- **Public-key encryption**: encrypt *to* a public key; only the private key decrypts. (In practice used only to move symmetric keys — asymmetric ops are ~1000× slower, so every real protocol is *hybrid*: public-key handshake, symmetric bulk.)
- **Digital signatures** — the reverse direction and arguably the more important primitive: sign with the *private* key, anyone verifies with the public. Yields integrity + authenticity + *non-repudiation* over anything: software updates and package signing (doc 09 §7's supply chain), TLS certificates, JWTs, git tags, Secure Boot ([OS doc Part 10](operating-systems.md)), cryptocurrencies (a coin transfer is a signed statement).
- **Diffie-Hellman key exchange**: two parties each combine their private value with the other's public value and arrive at the *same* shared secret — while an eavesdropper seeing both public values can compute nothing. This is [doc 05 §7](05-computer-networks.md)'s "closest thing to magic," and (as *ephemeral* DH — fresh values per session) it buys **forward secrecy**: steal the server's long-term key tomorrow, and yesterday's recorded traffic still can't be decrypted.
- **The binding problem** — public keys need *identity attached*, else the adversary substitutes their own (man-in-the-middle) and relays: hence **certificates** (a CA signs "this key belongs to example.com" — doc 05 §7's trust chains) or key fingerprint verification (SSH's first-connection prompt, Signal's safety numbers). Unbound public keys are unauthenticated encryption — to *someone*.
- On the horizon: quantum computers running Shor's algorithm would break RSA/ECC (not AES/hashes, roughly — [doc 07 §6](07-theory-of-computation.md)); **post-quantum** replacements (lattice-based: ML-KEM/Kyber) are standardized and already deployed hybrid in TLS by major browsers — "harvest now, decrypt later" is the threat model driving the timeline.

# 4 — Cryptography III: The Rules of Use

Where practitioners actually fail — crypto is almost never broken at the math and almost always at the seams:

1. **Don't roll your own** — not the primitives, not the *protocols*, not even the composition of sound primitives (padding oracles, IV reuse, and downgrade attacks are all composition failures). Use the boring blessed stack: TLS 1.3, libsodium/NaCl, AGE, your platform's keychain. The library's opinionated API *is* the security feature.
2. **Randomness is load-bearing**: keys, nonces, and tokens must come from the OS **CSPRNG** (`/dev/urandom`, `secrets` module) — never `random()`/`rand()` (predictable — [doc 07 §8](07-theory-of-computation.md): low entropy = guessable), never timestamps or PIDs. The Debian OpenSSL disaster (2008: a patch gutted entropy → every key generated for two years enumerable) and countless predictable-token breaks live here.
3. **Key management is the actual job**: generation, storage (KMS/HSM/keychain — not source code; committed secrets are a top real-world breach vector, and git history never forgets, doc 09 §2), rotation, revocation. The crypto is a vault door; key management is deciding who holds copies of the combination and where they write it down.
4. **Side channels**: implementations leak through *behavior* — timing (string comparison that returns early reveals how many bytes matched: compare secrets in constant time), error messages (padding oracles again), power draw, and CPU microarchitecture (Spectre/Meltdown — doc 01 §7.2's speculation leaking across [OS doc Part 11](operating-systems.md)'s boundaries). The lesson generalizes: **the system's observable behavior is part of its interface**, and adversaries read all of it.

# 5 — Authentication: Passwords and Beyond

- **Storing passwords**: never plaintext, never encrypted (keys get stolen), never fast-hashed (SHA-256 of a password falls to offline guessing at billions/second on GPUs). The standard: **slow, salted password hashes** — bcrypt/scrypt/**Argon2** — where *salt* (random per-user value stored beside the hash) kills precomputed rainbow tables and cross-user cracking, and deliberate slowness/memory-hardness turns "billions of guesses/sec" into thousands. Breaches still happen; this determines whether the dump is a nothingburger or a skeleton key — because of:
- **Reuse — the actual password problem**: users reuse passwords, so a breach of a gaming forum unlocks bank accounts (*credential stuffing*, run at industrial scale). The honest advice hierarchy: unique-per-site (only achievable via a **password manager** — the single highest-value practice to recommend) > long (length beats complexity theater — entropy is per doc 07 §8; `correct-horse-battery-staple` beats `P@ssw0rd!`) > rotation rituals (forced periodic change produces `Password2!` — retired from official guidance).
- **MFA**: something you know + have + are. Real ordering by strength: **hardware/passkey (FIDO2/WebAuthn) > authenticator app (TOTP) > SMS** (SIM-swapping is routine). Passkeys — per-site key pairs (§3) synced by your platform — are the emerging default: *unphishable* because the browser signs a challenge bound to the real origin; there is no secret to type into a fake site. Phishing-resistance is the property that matters (§8), and codes/passwords lack it by construction.
- **Sessions and tokens**: after login, a session cookie or signed token (JWT) *is* your identity — hence cookie theft = account theft (§7's XSS payoff), hence `HttpOnly`/`Secure` flags, expiry, and server-side revocation for anything sensitive (a signed JWT is valid until it expires — statelessness cuts both ways). **Authorization** then gates every request *server-side* ("is this user allowed *this object*?" — forgetting the object check is IDOR/broken access control, perennially the OWASP #1: change `/invoice/1001` to `/1002` and read a stranger's invoice).

# 6 — The Classic Attacks on Software

The recurring shape of nearly everything in this section: **data crossing a trust boundary gets interpreted as code** (the von Neumann duality, doc 01 §5.1, weaponized) — or bounds/state assumptions fail under adversarial input.

- **Buffer overflow / memory corruption** (C/C++ — doc 02 §11): write past an array's end (doc 03 §2's missing bounds check) and overwrite adjacent memory — classically the stack's *return address* (doc 01 §6), redirecting execution into attacker-supplied bytes. The 1988 Morris worm ran on this; it still headlines (~70% of severe CVEs in big C/C++ codebases are memory safety). Defenses in layers, each raising cost: NX/DEP + ASLR + canaries + CFI ([OS doc Part 11](operating-systems.md)), fuzzing (doc 09 §3) to find them first — and increasingly, *memory-safe languages* as the structural fix (the Rust bet; use-after-free and double-free are the heap-side siblings).
- **Injection — the same bug at every interpreter boundary**: user data concatenated into a command language becomes commands. **SQL injection** (`' OR '1'='1` — [doc 06 §3](06-databases.md)'s warning: fix is parameterized queries, structurally separating code from data), **command injection** (user input in shell strings), **path traversal** (`../../etc/passwd` in a filename), XSS (§7 — injection into HTML), log/header injection. The universal fix is the same idea each time: *never build code-strings from data; use APIs that keep the channels separate; validate at the boundary* (doc 02 §10.4, now adversarial).
- **Deserialization and parser attacks**: feeding hostile input to rich parsers (pickle, XML with external entities, image codecs) — parsers are interpreters too ([doc 08](08-compilers-and-languages.md)), and the more expressive the format, the more there is to exploit. Prefer dumb formats (JSON), allowlists, sandboxed parsing.
- **Race conditions as exploits** ([OS doc Part 6](operating-systems.md) adversarially): time-of-check-to-time-of-use (TOCTOU) — the file checked and the file opened are different by the time you act; balance-check/withdraw races double-spending. Fixes: atomic operations, transactions ([doc 06 §7](06-databases.md)), capability-style handles (act on the thing you checked, not its name).
- **DoS by asymmetry**: any endpoint where the attacker spends less than you (regex catastrophic backtracking — [doc 07 §2](07-theory-of-computation.md); hash-collision floods — doc 03 §6.3; zip bombs; unbounded uploads) — the fix is bounding resources at every boundary (timeouts, limits, [../distributed-systems/06](../distributed-systems/06-overload-control-and-resilience.md)'s overload control).

# 7 — Web Security

The web's specific attack surface, atop [doc 05](05-computer-networks.md)'s stack — three headliners plus the perimeter:

- **The browser's own rule — same-origin policy**: scripts from `evil.com` cannot read responses from `bank.com`. The invisible foundation everything below pokes at; **CORS** is the server-side mechanism for *deliberately* relaxing it (a misconfigured `*`-with-credentials undoes the foundation).
- **XSS (cross-site scripting)** = injection (§6) into HTML: attacker text rendered as script *runs in the victim's session* — reading the cookie/token (§5), acting as them. Fix: contextual output *encoding* by default (modern template engines escape unless you opt out — know your engine's escape hatches), `HttpOnly` cookies (steal-proof from JS), Content-Security-Policy as the depth layer.
- **CSRF**: the browser's helpfulness weaponized — it attaches your cookies to *any* request to a site, including one forged from an attacker's page (`<form action="https://bank/transfer">` auto-submitted). Fix: CSRF tokens (a secret the forger can't read — same-origin again) and `SameSite` cookies (now default — a case of the platform absorbing a defense).
- **The rest of the top tier, mapped**: broken access control / IDOR (§5 — check authorization per object, server-side), security misconfiguration (§8), vulnerable dependencies (doc 09 §7's supply chain: audit, pin, patch), SSRF (your server tricked into fetching internal URLs — cloud metadata endpoints being the jackpot; validate/deny-by-default outbound fetches). The **OWASP Top 10** is the living checklist of this section — read it annually; it barely changes, which is itself the lesson: the same holes, re-dug in each new stack.

# 8 — The Human Layer and Operations

Where most real breaches actually begin — and where defense is process, not math:

- **Phishing and social engineering**: the attacker doesn't break the crypto; they ask the human for the password on a convincing fake page (or call "from IT"). Defenses that work are *structural*, not exhortative: passkeys/hardware MFA (§5 — unphishable beats trained), verified side-channels for money-moving requests (call back on a known number — "CEO fraud" wires millions yearly), and a culture where reporting a click is praised (doc 09 §9's psychological safety, security edition — the silent click is the dangerous one).
- **Patching and inventory**: most exploited vulnerabilities are *old and public* — the race is attacker-weaponization vs. your patch latency (Equifax: a months-old known Struts hole). You can't patch what you don't know you run: inventory (and SBOMs for dependencies) is unglamorous and decisive.
- **Least-privilege operations**: scoped short-lived credentials, no shared root accounts, service accounts with exactly-needed permissions ([OS doc Part 11](operating-systems.md)'s principle, org-wide); secrets in vaults with rotation (§4.3); audit logs on sensitive actions (integrity-protected — attackers edit logs).
- **Assume breach**: segment networks and privileges so one compromised laptop isn't the whole company (blast-radius thinking — VMs/containers, [OS doc Part 12](operating-systems.md), are this at machine scale); monitor for anomalies; rehearse incident response like doc 09 §8's incidents (contain → eradicate → recover → blameless postmortem → disclose honestly — users forgive breaches handled with candor far more than cover-ups).
- **Backups are a security control**: ransomware turns availability (§1's third leg) into the extortion asset; offline/immutable backups, *restore-tested* (an unrestored backup is a hope, not a backup), convert "pay or die" into "restore and annoy."

# 9 — Secure Design Principles

The distilled checklist (Saltzer & Schroeder, 1975 — barely aged) — most already met in context:

1. **Least privilege** — every component gets minimum access (OS Part 11, §8).
2. **Defense in depth** — layers, because any one fails (§1's asymmetry answered).
3. **Fail secure** — errors deny by default (an auth service crash must not mean "allow").
4. **Economy of mechanism** — complexity is attack surface (doc 09 §1's enemy, adversarial edition); minimize what must be trusted (small TCB — microkernels' argument, OS Part 2).
5. **Complete mediation** — check *every* access, server-side, per object (§5's IDOR).
6. **Open design** — Kerckhoffs (§1); secrets in keys, not mechanisms.
7. **Separation of privilege** — two keys for the vault: code review + CI gates (doc 09), dual-control for wires (§8).
8. **Least astonishment / psychological acceptability** — security that fights users gets bypassed with sticky notes; make the secure path the easy path (passkeys, password managers, secure-by-default frameworks).
9. **Validate at trust boundaries; encode at output; keep code and data separate** — §6's universal fix, stated once.
10. **Threat-model early** — security retrofitted is security perforated; a one-hour "what could go wrong, from whom?" at design time (doc 09 §5's ADR moment) is the cheapest control that exists.

# 10 — The Road to Expertise

## 10.1 Books and resources

1. **Serious Cryptography** (Aumasson) — modern crypto for engineers: exactly §2–4 at full depth, honest about what breaks.
2. **Security Engineering** (Anderson) — *free online*; the field's encyclopedic masterwork: systems, people, economics, everything in §1/§8/§9 with a thousand war stories.
3. **The Web Application Hacker's Handbook** / PortSwigger **Web Security Academy** (free, interactive — the modern successor) — §7 hands-on.
4. **The Tangled Web** (Zalewski) — why the browser security model is what it is.
5. **OWASP Top 10 + Cheat Sheet Series** (free) — the living checklists.
6. **Cryptopals challenges** (cryptopals.com, free) — *the* way to learn crypto: implement the attacks (break ECB, padding oracles, nonce reuse) and §4 becomes permanent instinct.

## 10.2 Doing (in authorized environments only — that's what these are for)

1. **Cryptopals sets 1–3** — the single best exercise in this document.
2. **PortSwigger Academy labs**: execute XSS, SQLi, CSRF, IDOR against their deliberately-vulnerable targets; then fix the same bug classes in your own projects.
3. Stand up a deliberately vulnerable app (OWASP Juice Shop) locally; work through its challenge list.
4. **Audit your own project** (from doc 09's exercises) against the OWASP Top 10: parameterize every query, add CSP, check every endpoint's object-level authorization, run a dependency scanner, grep git history for secrets.
5. Set up your personal security floor: password manager, passkeys/hardware MFA on email + critical accounts (email is the root of the account tree — it resets everything else), disk encryption, tested backups.
6. **CTFs** (picoCTF for entry; then OverTheWire, HackTheBox) — capture-the-flag puzzles are the field's training grounds and genuinely fun.
7. Read one postmortem deeply (Equifax congressional report, the Debian OpenSSL writeup, CrowdStrike 2024): trace each §9 principle that was absent.

## 10.3 Ideas to retain forever

1. **Input is adversarial; threat models make "secure" meaningful** — CIA, and authn ≠ authz.
2. **The attacker needs one hole; you need layers** — defense in depth, least privilege, fail secure, assume breach.
3. **Kerckhoffs: secrets belong in keys** — use boring blessed crypto; your novelty is your vulnerability.
4. **AEAD or nothing; nonces never repeat; randomness from the CSPRNG; constant-time comparisons** — the four crypto-usage rules that prevent most crypto failures.
5. **Signatures and DH are the two miracles** — identity over insecure channels and shared secrets in public — but keys must be *bound to identities* (certificates) or MITM wins.
6. **Passwords: unique (manager) + long + slow-salted-hashed server-side; passkeys/hardware MFA because phishing-resistance is the property that matters.**
7. **Every classic software attack is code/data confusion at a trust boundary** — parameterize, encode contextually, bound every resource; memory-safe languages delete whole classes.
8. **The browser model in three: same-origin is the wall, XSS climbs it from inside, CSRF knocks politely with your cookies.**
9. **Humans and patch latency, not zero-days, are how breaches actually happen** — structural defenses beat training; inventory and patching beat glamour.
10. **Security is a property of the whole system over time** — designed in early, checked at every boundary, rehearsed like incidents, and never "done."

---

*This completes the curriculum. From here: [../distributed-systems/](../distributed-systems/) for many-machine truth, [../datastores/](../datastores/) and [../analytics/](../analytics/) for real engines, [../ai/](../ai/) for the LLM era — and the [README](README.md) for the map.*
