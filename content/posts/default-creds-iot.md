---
title: "The Basics Nobody Implements"
date: 2026-08-29
description: "An internal assessment of our IoT vendors. Cameras, fire alarms, and more, still running default admin credentials. The most basic control, skipped even in the newest deployments, turned out to be our single biggest risk."
tags: ["iot", "default-credentials", "assessment"]
categories: ["cybersecurity"]
---

---

## The Basics Nobody Implements 🔐

Recently I ran an internal assessment on the **IoT devices** across my organization.

The kind of hardware nobody really thinks about.

CCTV cameras. Fire alarm panels. The quiet boxes on the wall that just *work*.

Or so we assume.

---

Here's what I kept finding:

**Most vendors never change the default credentials.**

The device ships. It gets installed. It goes live.

And the admin login is still whatever the manual says on page one.

`admin` / `admin`. You know the type.

---

The **legacy devices** were the worst.

Older cameras and panels that have been running for years —

still sitting on factory logins.

No one ever touched them after install.

---

To be fair, the **newer devices** were better.

A lot of them now *force* you to change the password during setup.

You can't finish configuration until you do.

That's the right move. 👏

But even then —

some deployments still slipped through with defaults intact.

Someone found a way to skip it, or reused an old image, or just rushed the install.

---

### How I Checked

I kept the approach simple.

I used a small **PowerShell script** to sweep the IP range where the devices live —

just checking which hosts were up and answering.

Then I tested the login pages against known vendor defaults.

For the web-based panels, I used **Burp Suite** with the **Intruder** feature

to work through the common default combinations in a controlled way.

Nothing fancy.

That's kind of the point.

---

### Why This Matters

A camera with default creds isn't just *a camera*.

It's a live view into a building.

A fire alarm panel with default creds isn't just *a panel*.

It's a safety system someone could silence.

These aren't edge cases.

They're on the wall, right now, watching and protecting.

---

## Final Thoughts

Here's what stuck with me.

We spend so much energy on the **advanced stuff** —

the fancy exploits, the clever chains, the zero-days.

But in daily assessment work, the thing that keeps showing up

is the **most basic control of all**, just not implemented.

**Change the default password.**

That's it. That's the whole finding.

Something you'd assume is *too obvious to miss*

turns out to be the biggest security gap on the floor —

and it hides in even the newest technology.

The basics aren't basic if nobody actually does them.

Anyway — back to the assessment 🕵🏽‍♂️
