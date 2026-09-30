---
title: "Is Quantum the Next AI? What It Means for Cryptography"
date: 2026-09-30
description: "What quantum computing actually is, explained simply and technically. How it could break the encryption the internet runs on, how far away that really is in 2026, and whether we should be as nervous as we are about AI."
tags: ["quantum", "cryptography", "post-quantum", "pqc", "encryption"]
categories: ["cybersecurity"]
image: "images/quantum/ibm-q-system.jpg"
---

A few years ago, AI could barely write a proper sentence.

Now it lives in my terminal.

That jump made me ask a new question: **what's the next technology that's going to sneak up on us like that?**

And one answer keeps coming up in security circles:

**Quantum.** ⚛️

The headlines say quantum computers will "break all encryption". That sounds terrifying for someone who works in cybersecurity. So I went down the rabbit hole to answer three questions for myself:

1. What even *is* quantum technology?
2. How exactly does it threaten cryptography?
3. How close is it, really? Should I be scared the way I'm a little scared of AI?

Here's what I learned.

---

## Part 1: What is quantum technology?

![An IBM Quantum System One, a real quantum computer](/images/quantum/ibm-q-system.jpg)

*An IBM quantum computer. That shiny cylinder is mostly cooling: the chip inside runs colder than outer space.*

### The non-technical version

Everything around you (your phone, your laptop, the servers running this blog) is built on **bits**. A bit is either **0** or **1**. On or off. Every photo, message, and password is just a very long line of zeros and ones.

A quantum computer uses **qubits** instead.

The easiest way I found to picture it: 🪙

- A normal bit is a **coin lying on a table**. It's heads or tails. Done.
- A qubit is a **coin spinning in the air**. While it spins, it isn't "heads" or "tails". It's a bit of both, with some chance of landing each way. The moment you catch it (measure it), it becomes one or the other.

![A classical bit versus a qubit](/images/quantum/bit-vs-qubit.svg)

Now here's where it gets weird. With many spinning coins, they can be **linked**, so that how one lands is tied to how another lands, even though you haven't looked at either. And a clever quantum program can make the **wrong answers cancel each other out** and the **right answer reinforce itself**, like noise-cancelling headphones for mathematics. 🎧

That's the whole trick. A quantum computer is **not** a faster normal computer. It's a *different kind* of computer, amazing at a few very specific problems and useless for most everyday things. It won't run your browser faster. But for certain maths problems, it's not just faster, it's in a completely different league.

And one of those specific problems happens to be **the maths that protects the internet**. More on that soon.

### The technical version

If you want the real terms, a quantum computer uses four ideas:

**1. Superposition.** A qubit's state is a combination of |0⟩ and |1⟩, written as:

```text
|ψ⟩ = α|0⟩ + β|1⟩      where |α|² + |β|² = 1
```

α and β are "amplitudes" (complex numbers). When you measure, you get 0 with probability |α|² and 1 with probability |β|². The sphere in the diagram above (the Bloch sphere) is a way of drawing every possible state of one qubit.

**2. Entanglement.** Qubits can share one joint state that can't be split into separate states for each qubit. Measure one, and you instantly know something about the other. This is what lets qubits work together as a system instead of as separate coins.

**3. Exponential state space.** Describing *n* qubits takes 2ⁿ amplitudes. That's the source of the power:

![Every extra qubit doubles the state space](/images/quantum/qubit-scaling.svg)

At around 50 qubits, simulating the state on a normal computer gets impractical. At 300, you'd need more numbers than there are atoms in the observable universe.

**4. Interference.** This is the part pop-science usually skips, and it's the most important. A quantum computer does **not** "try every answer at once and give you the best one". When you measure, you get **one** random outcome. The whole art of quantum algorithms is to arrange the maths so that the amplitudes of wrong answers **cancel out** (destructive interference) and the right answer's amplitude **grows** (constructive interference). Then when you measure, you probably get the right answer.

That's why only *some* problems get a quantum speed-up. You need a problem with hidden structure that interference can exploit.

**And the big enemy: noise (decoherence).** Qubits are extremely fragile. Heat, vibration, stray radiation, even cosmic rays knock them out of their state. Today's qubits make an error roughly once every few hundred to a few thousand operations. The transistors in your laptop, by comparison, are so reliable that you never have to think about errors at all. **Almost all of the engineering challenge is fighting this noise.**

### What are qubits physically made of?

Different companies are betting on completely different physics:

| Approach | How it works | Who's betting on it |
|---|---|---|
| **Superconducting circuits** | Tiny circuits cooled to about −273 °C, a hair above absolute zero | Google, IBM |
| **Trapped ions** | Single charged atoms held in place by electric fields and controlled with lasers | Quantinuum, IonQ |
| **Neutral atoms** | Arrays of atoms held by laser "tweezers" | QuEra, Atom Computing, Pasqal |
| **Photons** | Particles of light travelling through chips | PsiQuantum, Xanadu |
| **Topological** | Exotic states of matter that could be naturally resistant to noise | Microsoft (still early and debated) |

![Google's Sycamore quantum processor chip](/images/quantum/google-sycamore-chip.jpg)

*Google's Sycamore chip, the one behind the 2019 "quantum supremacy" experiment.*

### It's bigger than just computers

"Quantum technology" is actually a family:

- **Quantum computing:** the one this post is mostly about.
- **Quantum sensing:** ultra-precise sensors for navigation without GPS, medical imaging, and finding things underground. This is arguably the most *practically* advanced branch today.
- **Quantum communication:** e.g. Quantum Key Distribution (QKD), which uses physics to detect eavesdroppers on a link.
- **Quantum simulation:** using quantum systems to model molecules and materials, useful for drug discovery, batteries, and fertiliser chemistry. This is widely expected to be quantum computing's first truly useful job.

---

## Part 2: A 3-minute crypto refresher

To understand the threat, you need to know what's being threatened.

### Symmetric encryption: one shared key 🔑

Both sides share the **same secret key**. It's fast, and it's used for bulk data: your disk, your VPN tunnel, the actual content of your HTTPS traffic. The main example is **AES**.

The problem: how do two strangers on the internet agree on a shared key without someone in the middle stealing it?

### Asymmetric (public-key) encryption: the padlock trick 🔓

You publish a **public key** (think: an open padlock anyone can snap shut) and keep a **private key** (the only key that opens it). Anyone can lock a message for you; only you can open it.

Public-key crypto is what lets your browser and a website **agree on a shared AES key** safely, and it's what **digital signatures** use to prove "this software update really came from Microsoft" or "this certificate really belongs to your bank". The main examples are **RSA** and **Elliptic Curve Cryptography (ECC)**.

### Why is public-key crypto safe? Because some maths is one-way

RSA is built on a simple fact:

- Multiplying two huge prime numbers is **easy**: `p × q = n`.
- Going backwards, taking `n` and finding `p` and `q`, is **absurdly hard**.

For a 2048-bit RSA key, `n` is a number with **617 digits**. The best classical algorithms would need longer than the age of the universe to factor it. ECC relies on a similar one-way problem (the "discrete logarithm" on elliptic curves).

**The entire security of HTTPS, SSH, VPNs, code signing, and crypto wallets rests on "nobody can do this backwards maths in practice".**

Keep that sentence in mind. 👀

---

## Part 3: Where quantum and cryptography collide

### Shor's algorithm: the one that breaks things 💥

In 1994, mathematician **Peter Shor** showed that a big enough quantum computer could factor huge numbers, and solve discrete logarithms, **efficiently**. Not "a bit faster". The problem goes from *longer than the age of the universe* to *hours or days*.

That means: **RSA, ECC, and Diffie-Hellman all break.**

#### How does it work? A toy example

The trick is to turn factoring into a problem of **finding a repeating pattern** (a "period"), which quantum computers are brilliant at thanks to interference.

Let's factor **15** the Shor way. Pick a random number, say **7**, and look at powers of 7, keeping only the remainder after dividing by 15:

```text
7¹ mod 15 = 7
7² mod 15 = 4
7³ mod 15 = 13
7⁴ mod 15 = 1   ← back to 1
7⁵ mod 15 = 7   ← and the pattern repeats
```

The pattern repeats every **4** steps, so the period `r = 4`. Now a bit of classical maths:

```text
7^(r/2) = 7² = 49
gcd(49 − 1, 15) = gcd(48, 15) = 3
gcd(49 + 1, 15) = gcd(50, 15) = 5
```

**15 = 3 × 5.** 🎉 Factored.

For 15, you can find the period by hand. For a 617-digit number, the pattern is so long that no classical computer could ever find it. **Finding that period is the one step the quantum computer does**, using something called the Quantum Fourier Transform, which makes the period "ring out" through interference. Everything else is ordinary maths.

You can play with the classical part yourself:

```python
from math import gcd

def find_period(a, n):
    # This loop is the part a quantum computer does exponentially faster
    x, r = a % n, 1
    while x != 1:
        x, r = (x * a) % n, r + 1
    return r

n, a = 15, 7
r = find_period(a, n)
print("period:", r)
print("factors:", gcd(a**(r//2) - 1, n), gcd(a**(r//2) + 1, n))
```

### Grover's algorithm: the one that only dents things 🔨

Grover's algorithm speeds up brute-force search, but only by a **square root**. Brute-forcing a 128-bit key takes about 2¹²⁸ tries classically, and about 2⁶⁴ with Grover. In practice (Grover also parallelises badly and needs very long error-free runs), the takeaway is simple: **use 256-bit symmetric keys and you're fine**.

### Put together

![What a big quantum computer does to today's crypto](/images/quantum/crypto-impact.svg)

So the danger is **not** "all encryption breaks". It's more specific, and in a way worse: **the public-key layer that everything else depends on breaks**. The AES key protecting your data is fine. But the RSA or ECC handshake that *delivered* that AES key isn't.

### "Harvest now, decrypt later": why this matters today

Here's the part that made me take this seriously:

![Harvest now, decrypt later](/images/quantum/harvest-now.svg)

An attacker doesn't need a quantum computer **today**. They can **record encrypted traffic now**, store it cheaply, and decrypt it **years later** when the hardware exists.

So the real question isn't *"when will a quantum computer exist?"* It's:

> **How long does my data need to stay secret, and how long will it take me to migrate?**

If the answer to "how long must this stay secret" is 10+ years (medical records, government secrets, long-term intellectual property, identity data), then the threat window for that data **may already be open**.

---

## Part 4: So how far along is quantum, really? (2026)

This is where the hype and reality split, so let's be precise.

### Physical qubits vs. logical qubits

- A **physical qubit** is one real, noisy piece of hardware.
- A **logical qubit** is one *reliable* qubit built from **many** physical qubits using **quantum error correction**: the qubits constantly check on each other, and errors get detected and fixed faster than they pile up.

Breaking RSA needs **thousands of logical qubits** running reliably for **days**. That translates into a very large number of physical ones.

### Where we are

The honest summary of the last two years:

- **December 2024:** Google's **Willow** chip (105 physical qubits) showed that making the error-correcting code **bigger made the logical qubit better**, not worse. That's called "below threshold", and it's the proof that the error-correction strategy actually works in real hardware. This was a genuine milestone.
- **2025:** Quantinuum's **Helios** (98 trapped-ion qubits) reported **48 logical qubits** using an efficient error-detecting code. Other teams, including QuEra and Atom Computing with Microsoft, have also demonstrated dozens of logical qubits since late 2024. These records come with fine print about *how* "logical" they are, but the trend is real.
- **Roadmaps:** IBM publicly targets **Starling** by **2029**, about **200 logical qubits** able to run 100 million operations. Several companies target 100+ logical qubits around 2027–2028.

### What does breaking RSA-2048 require?

The estimates have been **falling fast**, and that's the scary part:

| Year | Estimate for RSA-2048 | Who |
|---|---|---|
| 2019 | ~**20 million** noisy qubits, ~8 hours | Craig Gidney & Martin Ekerå (Google) |
| May 2025 | **< 1 million** noisy qubits, < 1 week | Craig Gidney (Google) |
| Feb 2026 | ~**100,000** qubits, ~1 month (*not peer-reviewed; assumes hardware nobody has shown yet*) | Iceberg Quantum ("Pinnacle" preprint) |

Now compare that with what exists:

![Qubits we have vs. qubits RSA-2048 needs](/images/quantum/qubit-gap.svg)

We're at **around 100 high-quality physical qubits** in the best machines (bigger chips exist, but they're noisier). The most optimistic estimate needs **~100,000**, and those qubits must also be much **better**, stable for weeks, with fast decoding. The mainstream estimate needs about a million.

So the gap is still **roughly 1,000×**, not counting quality.

**Has anyone broken real encryption with a quantum computer?** No. The largest numbers factored with actual Shor-style quantum circuits are tiny (like 15 and 21). Headlines claiming "Chinese researchers broke RSA with a quantum computer" have, so far, all turned out to be misleading: either tiny keys or classical tricks.

![Timeline of quantum computing and cryptography](/images/quantum/timeline.svg)

### Is it still growing? Very much so

- The **resource estimates** for breaking RSA dropped about **20×** in six years, from algorithms and engineering, not just hardware.
- **Error correction** went from theory to working demos.
- **Money** keeps pouring in from governments and big tech.
- Every major player publishes a **roadmap** with dates, and so far they've mostly hit them.

Nobody knows the exact date of "Q-Day", the day a quantum computer can break RSA-2048. Expert surveys spread their guesses widely, from roughly the early 2030s to "decades away". But almost nobody serious says *never* any more.

---

## Part 5: So, is quantum as scary as AI?

This was my original question, so here's my honest answer.

| | 🤖 AI | ⚛️ Quantum |
|---|---|---|
| **Is it here today?** | Yes, in everyone's pocket | Only in labs and the cloud, for experiments |
| **How broad is the impact?** | Touches nearly everything | Narrow: specific maths problems |
| **Impact on security** | Faster phishing, malware, and recon, *today* | Could break public-key crypto, *later* |
| **Does the fix exist?** | Not really, still being figured out | **Yes.** Post-quantum crypto is already standardised |
| **Can it hurt us before it arrives?** | n/a, it's here | **Yes**, via harvest now, decrypt later |

My conclusion:

- **AI is scarier *right now*** because it's already here, it's everywhere, and it's moving fast in unpredictable directions.
- **Quantum is scarier *in one very specific way*:** it's a **deadline**. When it hits, it breaks a foundation that almost everything depends on, all at once. And some of today's secrets are already on the clock.

The good news: **unlike with AI, we already know how to defend against it.** We "just" have to actually do it. (And if you've read my post on [default credentials](/posts/default-creds-iot/), you know that *"just do the basics"* is exactly where organisations struggle 😅)

---

## Part 6: The fix already exists (post-quantum cryptography)

**Post-quantum cryptography (PQC)** means new algorithms, running on **normal computers**, built on maths problems that neither classical nor quantum computers are known to solve efficiently (mostly structured lattice problems and hash functions).

In **August 2024**, the US standards body **NIST** published the first three standards:

| Standard | Algorithm | Replaces | Used for |
|---|---|---|---|
| **FIPS 203** | **ML-KEM** (was "Kyber") | RSA/ECDH key exchange | Agreeing on keys (e.g. in TLS) |
| **FIPS 204** | **ML-DSA** (was "Dilithium") | RSA/ECDSA signatures | Signing (certificates, code) |
| **FIPS 205** | **SLH-DSA** (was "SPHINCS+") | Backup signature | Signing, based only on hash functions |

In March 2025, NIST also picked **HQC** as a backup key-exchange algorithm built on different maths, in case lattices ever turn out to be weaker than we think.

NIST's proposed roadmap (**IR 8547**) says RSA and ECC should be **deprecated after 2030** and **disallowed after 2035**.

### It's already running on your machine

This isn't theory any more:

- **Chrome, Edge, and Firefox** already use a **hybrid** key exchange (classic X25519 **plus** ML-KEM) with sites that support it, like Cloudflare and Google. Hybrid means an attacker would have to break **both**.
- **OpenSSH 10** (2025) made the hybrid **`mlkem768x25519-sha256`** its default key exchange.
- **Signal** (PQXDH) and **Apple iMessage** (PQ3) added post-quantum protection to messaging.

Check your own SSH client:

```bash
ssh -Q kex | grep -i mlkem
```

And OpenSSL 3.5+:

```bash
openssl list -kem-algorithms | grep -i ml-kem
```

If you see results, your tools already speak post-quantum. 🎉

### What should you actually do?

If you work in security or IT, here's the practical checklist I'm taking away:

1. **Inventory your crypto.** Where do you use RSA or ECC? Think TLS certificates, VPNs, SSH keys, code signing, internal PKI, IoT devices, and third-party vendors. You can't migrate what you can't see.
2. **Find your long-lived secrets.** Which data must stay confidential for 10+ years? That data needs protecting first, because of harvest now, decrypt later.
3. **Aim for crypto-agility.** Build systems where swapping an algorithm is a **config change**, not a rewrite.
4. **Turn on hybrid PQC where it's already available**: modern TLS libraries, OpenSSH 10, VPNs that support it.
5. **Use 256-bit symmetric keys** (AES-256) for anything long-term.
6. **Ask your vendors** about their PQC roadmap. Especially hardware and IoT vendors, whose devices live for 10–15 years. (Those same cameras still running `admin/admin`… yeah.)

---

## Final thoughts

I started this thinking quantum might be "the next AI": something that arrives suddenly and changes everything overnight.

What I found is different.

**Quantum is real, it's growing faster than most people expected, and the estimates for breaking RSA keep dropping.** It's not science fiction any more.

But it's also **not here yet**. We're still roughly a thousand times away in qubit count, and quality is a whole other mountain.

The part that stuck with me most is **harvest now, decrypt later**. The attack on today's secrets doesn't need a quantum computer today. It just needs patience.

So is it something to look at? **Absolutely.** Not with panic, but the same way we should treat any deadline we can see coming:

> *Know where your crypto is. Protect your long-lived secrets first. Make switching algorithms easy.*

Stay calm. Start early. That's the whole finding. 🧘🏽‍♂️⚛️

---

*Image credits: IBM Quantum System One photo by IBM Research, [CC BY 2.0](https://creativecommons.org/licenses/by/2.0/), via [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:IBM_Q_system_(Fraunhofer_2).jpg). Sycamore chip by Google, [CC BY 3.0](https://creativecommons.org/licenses/by/3.0/), via [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Google_Sycamore_Chip_002.png). All diagrams are my own.*

*Sources: [NIST PQC standards (FIPS 203/204/205)](https://csrc.nist.gov/projects/post-quantum-cryptography), [NIST IR 8547 draft](https://csrc.nist.gov/pubs/ir/8547/ipd), [Gidney 2025, "How to factor 2048 bit RSA integers with less than a million noisy qubits"](https://arxiv.org/abs/2505.15917), [Google Quantum AI: Willow](https://blog.google/technology/research/google-willow-quantum-chip/), [PostQuantum.com on the Pinnacle preprint](https://postquantum.com/post-quantum/pinnacle-architecture-break-rsa-2048-critical/).*
