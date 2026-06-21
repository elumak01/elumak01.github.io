---
layout: post
title: "Least Privilege on a Clock: What PIM Actually Solves"
date: 2026-06-19
categories: [identity, azure, security]
tags: [PIM, Entra ID, Privileged Identity Management, Zero Trust]
---

Open any Microsoft Entra tenant and check who holds Global Administrator. Nine times out of ten you'll find more permanent holders than anyone expected — an admin added two years ago for a project that's long since closed, a service account nobody remembers provisioning, a support lead who got "temporary" access during an outage and never had it removed.

That's standing privilege. And it's one of the easiest wins you can hand an attacker.

## The problem isn't the role, it's the duration

Every second an account holds elevated rights, it's a target worth going after. Phishing, token theft, a leaked credential from some unrelated breach — none of it matters if the account it lands on is sitting there with admin rights switched on permanently. Standing access doesn't just increase your blast radius if something goes wrong. It increases the odds something goes wrong in the first place, because it's always available to be misused, whether by an attacker or by an admin who didn't need to touch that role today.

This is the gap Privileged Identity Management is built to close.

## What PIM actually is

PIM is a Microsoft Entra ID capability for managing, controlling, and monitoring access to privileged roles — across Entra ID itself, Azure resources, and services like Microsoft 365 and Intune. It covers three surfaces: Entra roles, Azure resource roles, and group-based access. I'll get into how each of those works in later posts — for this one, the product surface matters less than the concept underneath it.

## Eligible vs Active — the concept that matters

PIM's core idea comes down to one distinction:

- **Active** assignment — the privilege is on, all the time. This is what most orgs default to without thinking about it.
- **Eligible** assignment — the user holds no privilege until they ask for it. They activate the role, provide a justification, clear MFA (or an approval step, if you've configured one), and get the role for a fixed window. When the window closes, the privilege drops away on its own.

That's the whole shift: from "this person is an admin" to "this person can become an admin, for a defined period, with a record of why." I've found a 4-hour activation window works well as a starting point for most admin roles — long enough to actually get work done, short enough that nobody's sitting on standing access by accident.

It sounds like a small change. In practice it's the difference between an attacker landing on a token that's immediately useful, and landing on a token that gets them nothing without going through activation, MFA, and an audit trail with their name on it.

## Why this is more than a compliance checkbox

Three things happen when you move from active to eligible by default:

1. **The window of opportunity shrinks.** A compromised account with no active privilege is a much less interesting target.
2. **You get an audit trail for free.** Every activation carries a timestamp, an identity, and a justification — no extra logging effort required.
3. **You're set up for Zero Trust controls on top.** Once activation is a discrete event, you can require fresh authentication at the moment of activation, not just at sign-in. More on that when I cover Conditional Access integration later in this series.

## "We're too small for this" / "It's too complicated"

Two objections come up constantly, and neither holds up well:

- **Licensing** — PIM needs Entra ID P2 or Entra ID Governance, but you only need to license the accounts that hold privileged roles, not your whole tenant. For most orgs that's a handful of seats, not hundreds.
- **Complexity** — initial setup is portal-driven. You don't need scripts or automation to get real value on day one; that comes later if you want it.

If you're running any tenant with standing tier 0 admin roles right now, this is worth fixing before anything else on your security backlog. this is applies to other tier 0 roles

## Next in this series

Next post: a full walkthrough of PIM for Microsoft Entra roles — activation flow, MFA enforcement, approval workflows, and how to actually choose an activation duration instead of guessing.
